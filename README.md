# stc8h8g_lib_template

STC8H 和 STC8G 的通用裸机工程模板：Timer0 驱动 **1ms Tick 时基 + 钩子机制**，串口1（115200）已初始化、`printf` 开箱即用，支持 **EIDE（VS Code）+ Keil uVision** 双工程。

## ☕ 支持与打赏

如果这个工程帮你省了时间，欢迎请作者喝杯咖啡；也欢迎点 ⭐ Star 让更多人看到：

- 爱发电：`https://afdian.com/a/你的ID`（**发布前替换成你的链接**）
- GitHub Sponsors：`https://github.com/sponsors/你的用户名`（可选）

> 打赏不设门槛，一块钱也是鼓励。

# 编辑日期：20260203

# 文件说明：
│  keilclean.bat -> 清除临时文件脚本
│      
├─build
│  └─Target 1
│          STC8G-H-LIB-template.hex -> 量产烧录文件
│          
├─Driver -> 官方硬设函数库
│  │  UPDATE-NOTE.txt -> 更新说明
│  ├─inc -> 头文件
│  ├─isr -> 中断函数
│  └─src -> 函数原型
│          
├─RVMDK -> 项目工程文件
│  │  STARTUP.A51
│  │  STC8G-H-LIB.uvproj
│  ├─Listings
│  └─Objects
│          STC8G-H-LIB.hex -> 量产烧录文件
│          stc_tool_config.cfg -> STC烧录软件配置文件
│          stc_tool_config使用说明.png -> STC烧录软件配置文件使用说明
│          
└─User -> 程序源代码文件

# 芯片不同工况的电流情况
   1、使用全浮空输入时，端口输入模拟信号，待机电流会增高
   2、浮空输入端口，如果使用开关电源供电，待机电流会增高，使用电池则不会出现这种问题
   3、设置内部32kHz时钟后，需要把IRC(内部高速时钟)关闭才能使功耗优化
   休眠3.3uA时，没有关闭实测839uA；关闭后与规格书标称相符，实测491uA。
      a、  1倍分频：491uA
      b、 32倍分频：486uA
      c、 64倍分频：487uA
      d、128倍分频：486uA
      e、255倍分频：486uA
   功耗和时钟分频没有太直接的关系。

# 20260203
   在官方基础上加入：
   1、在"Type_def.h"头文件加入一些新定义

# 20260923
   1、新增 Tick 1ms 时基与钩子机制（Timer0 中断驱动、钩子表分发），新增模块程序模板，并补全 User/ 层 if/for 大括号统一风格
