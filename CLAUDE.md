# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 用中文交流和注释

本项目所有沟通、代码注释、文档均使用中文。

## 工程性质

STC8H / STC8G 系列单片机的**裸机工程模板**（Keil C51 编译器，`LARGE` 存储模型）。无测试框架、无 CI、无 lint 脚本——验证方式是上板后用串口 1（115200）观察输出。`printf` 开箱即用（UART1 已初始化）。

## 构建

双工程并存，**修改源码后两个工程都要能编译**：

| 工具 | 入口 | 产物 |
|---|---|---|
| EIDE（VS Code） | `Ctrl+Shift+P` → `EIDE: Build Project` / `Flash` / `Rebuild` / `Clean` | `build/Target 1/*.hex` |
| Keil uVision | `RVMDK/STC8G-H-LIB.uvproj` | `RVMDK/Objects/*.hex` |
| 清理临时文件 | `keilclean.bat`（递归删除 `.obj/.hex/.map` 等） | — |

编译器：C51 `-O8`（SPEED 优化）。`EIDE: Flash` 用外部烧录脚本（`uploadConfigMap.Custom` 为空，需自行配置）。

## 新增源文件必须同步两处

添加 `.c` 文件后，**两个工程清单都要手动加**，否则一处能编一处不能：

- `.eide/eide.yml` → `targets."Target 1".virtualFolder.folders[].files` 下追加 `{ path: ... }`
- `RVMDK/STC8G-H-LIB.uvproj` → 对应 `<FilePath>` 节点

`User/程序模板.c/.h` 是**模块模板参考件，故意未加入任何工程清单**，不会被编译——复制它后改名并登记到上面两处。

## 文件编码（关键）

`User/`、`Driver/` 下所有 `.c` / `.h` 均为 **GB2312/GBK + Tab 缩进 + CRLF**。`README.md`、`User/Tick使用说明.md` 是 UTF-8。

编辑源码文件时**必须保持 GBK 编码**，否则文件里已有的中文注释会变成乱码。不要批量转码，不要用 Write 整文件重写含中文的 `.c`/`.h`（改用 Edit 做局部替换）。工作区已设 `files.autoGuessEncoding: true`。

## 架构

三层结构：

- `User/` — 业务层。`main.c` 负责初始化顺序 + `while(1)` 轮询；每类外设一个 `Xxx.c/.h` 模块。
- `Driver/` — STC 官方硬件库（勿改动风格）：`inc/` 头文件、`src/` 初始化/控制函数、`isr/` 中断服务函数。
- `RVMDK/` — Keil 工程与启动代码（`STARTUP.A51`）。

### Tick 时基 + 钩子机制

`Timer0`（1T、16 位自动重装、1000Hz）驱动 1ms 时基：

```
Timer0 ISR (Driver/isr/STC8G_H_Timer_Isr.c)
  → TickHookDispatch() 宏 (Tick.h)
    → TickHooks[0] = TickTime_Inc()   每 1ms TickTime++
    → TickHooks[1..7] = 各模块注册的短任务
```

模块三件套约定（见 `User/程序模板.c`）：

- `Xxx_Init()` — 显式清零模块变量 + `Tick_Hook_Register(Xxx_Tick)`
- `Xxx_Handler()` — 主循环轮询，用 `TickTime_Elapsed()` 做周期/超时判断
- `Xxx_Tick()` — 1ms 钩子，只允许置标志 / 翻转 IO / 计数

### 初始化顺序（不可乱序）

```
EAXSFR()          // 扩展寄存器访问使能，必须在最前
Uart_Init()
Tick_Init()       // 内部会清空整个钩子表，必须早于一切注册
Xxx_Init()        // 各模块注册钩子
EA = 1;           // 全局开中断——所有 Register 调用都必须在它之前
```

`Tick_Init()` 会先清 `TickHookCnt` 和 `TickHooks[]` 再注册 `TickTime_Inc`，任何早于它的注册都会被抹掉。

## 硬性约束

1. **全局变量 / static 必须显式赋初值。** `STARTUP.A51` 里 `XDATALEN EQU 0`，LARGE 模型下 XRAM 不做零初始化，上电是垃圾值；声明期 `= 0` 会被编译器折叠、不生成初始化指令，同样无效。`Tick_Hook_Register` 里的空指针检查依赖 `Tick_Init()` 手工填 0。
2. **钩子内禁止阻塞操作**（`printf`、忙等串口等）——会长时间占住 Timer0 中断导致时基抖动，并与 main 的串口输出交错乱码。
3. **周期/超时一律用 `TickTime_Elapsed(start)` 差值判断**，禁止 `TickTime == N` 相等比较（`TickTime++` 非原子、16 位会回绕，周期约 65.5 秒）。
4. **Timer1 已被 UART1 波特率发生器占用，不要挪用。** Timer0 优先级低于 UART1（若调高 Timer0 优先级会产生死锁）。
5. **钩子函数只能由 `TickHookDispatch()` 在中断中调用**，不要同时在 main 里直接调用——C51 函数不可重入。
6. **跨模块的钩子函数必须外部链接（不能 `static`）**；仅注册源文件内使用的可 `static`。
7. `Config.h` 的 `MAIN_Fosc` 必须与实际晶振一致（默认 11059200），否则 `TIM_Value` 算错、时基不准。`MAIN_Fosc / TICK_FRE` 为整数除法，会截断。
8. 钩子表容量 `TICK_HOOK_MAX = 8`，槽位 0 已被 `TickTime_Inc` 占用，业务最多 7 个。

## 代码风格

- Tab 缩进，大括号独立成行（Allman），所有 `if` / `for` / `while` 必须带大括号。
- 类型用 `u8` / `u16` / `u32` / `int16` 等（定义于 `Type_def.h`），不用 `unsigned short`。
- 状态用 `ENABLE` / `DISABLE` / `SUCCESS`(0) / `FAIL`(-1)，中断优先级用 `Priority_0..3`（数字越小优先级越高）。
- 每个函数上方保留块注释头：`函数 / 说明 / 参数 / 返回 / 版本`。
- 头文件统一以 `#include "Config.h"` 开头（它会拉入 `Type_def.h`、`STC8H.H`、`intrins.h`、`stdlib.h`、`stdio.h`）。
- 注意 `Config.h` 里写的是 `#include "type_def.h"` 和 `"STC8H.H"`，实际文件名是 `Type_def.h` / `STC8H.h`——大小写不一致但在 Windows 上可用，**不要"顺手改正确"**，会破坏与官方库的一致引用。

## 参考文档

- `User/Tick使用说明.md` — Tick 机制完整说明、API 表、约束清单、验证方法（权威文档，改动 Tick 后请同步更新）。
- `Driver/UPDATE-NOTE.txt` — 官方库更新记录（库版本 2024.04.29）。
- `README.md` — 文件说明、功耗实测数据、编辑日志。
