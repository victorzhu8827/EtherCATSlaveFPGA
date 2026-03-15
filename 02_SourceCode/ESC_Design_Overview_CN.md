# EtherCAT Slave Controller（ESC）Verilog 软件设计概要

## 1. 文档目的

本文档用于定义基于 Verilog 实现完整 EtherCAT 从站控制器（ESC）的软件与 RTL 设计概要，在详细架构设计、寄存器设计、接口时序设计及 RTL 编码开展之前，建立统一、清晰、可执行的工程基线。

本次修订后的目标不再是“通用型 ESC”，而是明确以 Beckhoff `ET1810 / ET1811 / ET1812`，即 `EtherCAT IP Core for Intel/Altera FPGAs V4.0.0` 为功能对标基线。后续详细设计、寄存器定义、RTL 划分与验证计划，均应以此基线为准。

本文档的设计依据来源于 [01_Docs](D:/CodexWorkspace/EtherCATSlave/01_Docs) 目录中的以下核心资料：

- `ETG2200_V3i1i1_G_R_SlaveImplementationGuide.pdf`
- `ethercat_esc_datasheet_sec1_technology_v2.5.pdf`
- `ethercat_esc_datasheet_sec2_registers_v3.3.pdf`
- `ethercat_et1100_datasheet_v2i1.pdf`
- `ethercat_ipcore_altera_v4.0.0_datasheet_sec3_v1.0.pdf`
- `ethercat_ipcore_altera_v3.0.10_datasheet_v1i0.pdf`

## 2. 设计范围与边界

### 2.1 设计范围

本项目面向 FPGA/ASIC 形式的硬件 ESC 内核实现，范围包括：

- EtherCAT 帧的接收、解析、处理与转发
- ESC 寄存器空间、用户 RAM、过程数据 RAM 与私有 PDI RAM
- EtherCAT 应用层（AL）状态机相关硬件支持
- FMMU 地址映射
- SyncManager 邮箱模式、缓冲模式与高级一致性特性
- 分布式时钟（Distributed Clocks，DC）
- SII EEPROM 接口及 EEPROM 仿真支持
- 中断、事件与看门狗逻辑
- 单/双 PDI 接口
- GPIO、LED、状态特征镜像
- ESC 所需的 PHY/MII/RMII/RGMII 侧接口管理

### 2.2 对标目标

本文档中的“完整功能”定义为：

1. 以 EtherCAT 标准 ESC 通用功能为基础。
2. 对齐 ET1810/ET1811/ET1812 IP Core V4.0.0 所公开的功能边界、接口类型、资源上限、诊断能力和集成能力。
3. 对于 V4.0.0 已明确取消的能力，不再作为实现目标。

### 2.3 范围外内容

以下内容不属于纯 ESC 硬件内核的基础范围，除非后续阶段明确纳入，否则应视为外部软件或上层固件功能：

- 完整的应用层协议栈语义实现：CoE、FoE、EoE、SoE、AoE
- Object Dictionary 语义处理
- 设备 Profile 实现，例如 CiA402、MDP 等
- 用户应用逻辑
- EtherCAT 主站侧 ENI/ESI 工具链

说明：

ESC 必须提供上述协议所依赖的邮箱传输机制以及硬件资源，但协议语义本身不应纳入基础 ESC RTL 内核。

## 3. 需求定义

### 3.1 功能需求

ESC 应满足以下完整功能要求：

1. 支持 `1` 至 `4` 个 EtherCAT 通信端口。
2. 支持端口物理接口类型：`MII`、`RMII`、`RGMII`，并允许所有端口统一选择同一种类型。
3. 支持 Auto-Forwarder 与 Loopback 结构，并支持与端口开启/关闭状态对应的标准帧处理顺序。
4. 支持 EtherCAT 数据报在线处理，并以尽可能低的延迟完成帧转发。
5. 提供 64 KB 的逻辑 ESC 地址空间，包括：
   - `0x0000` 至 `0x0FFF` 的寄存器空间
   - `0x0F80` 至 `0x0FFF` 的 User RAM / Feature Mirror 区
   - `0x1000` 起始的过程数据 RAM 空间
6. 支持站地址（Configured Station Address）及别名地址（Alias Address）处理。
7. 实现 AL Control、AL Status、AL Status Code 寄存器。
8. 支持 EtherCAT 状态机：`INIT`、`PRE-OP`、`SAFE-OP`、`OP`，以及可选的 `BOOT` 状态。
9. 支持 `0` 至 `16` 个 FMMU 实例，支持位级映射。
10. 支持 `0` 至 `16` 个 SyncManager 实例。
11. 支持 SyncManager 邮箱模式与缓冲模式操作。
12. 支持 SyncManager Sequential Mode。
13. 支持 SyncManager Deactivation Delay。
14. 支持由 SyncManager 保证的过程数据一致性控制。
15. 支持 AL Event 事件生成与 ECAT Event 事件生成。
16. 支持扩展 AL Event Mask，包括 PDI0 与 PDI1 独立 AL Event Mask。
17. 支持过程数据看门狗与 PDI 看门狗。
18. 支持组合式 PDI Watchdog 策略。
19. 支持 SII EEPROM 的读、写及重载控制。
20. 支持 EEPROM 仿真模式。
21. 支持 EEPROM I2C 基地址配置与 I2C 时钟配置。
22. 支持 Distributed Clocks 功能。
23. 支持 `32` 位或 `64` 位 DC 系统时间。
24. 支持 DC Receive Times。
25. 支持最多 `4` 路 DC SyncSignal。
26. 支持最多 `4` 路 DC LatchSignal。
27. 支持 DC SyncManager Event Times。
28. 支持 DC Speed Counter Direct Control。
29. 支持将 `SyncSignal[0..3]` 选择性映射到 `AL Event Request` 中断路径。
30. 支持最多 `2` 个 PDI 接口：`PDI0` 与 `PDI1`。
31. 支持 PDI private RAM，用作双 PDI 间共享但对 EtherCAT 主站不可见的私有存储区。
32. 支持 PDI User Mode 配置寄存器及运行时模式切换能力。
33. 支持 PDI Information Register。
34. 支持 PDI SyncManager/Interrupt Acknowledge by Write。
35. 支持将上述写确认机制限制为“仅中断确认”模式。
36. 支持以下 PDI 类型：
   - Digital I/O
   - SPI Slave
   - 异步 8/16/32 位 µController 接口（PDI0）
   - Avalon Memory-Mapped
   - AXI4
   - AXI4-Lite
37. 支持 SPI Slave 的 `1/2/4/8` 路 MISO/MOSI、单向/双向工作模式，以及直接访问 SyncManager 的附加片选信号。
38. 支持 Digital I/O PDI 的输入锁存源选择：SOF、外部锁存、DC Sync0、DC Sync1。
39. 支持 Digital I/O PDI 的输出触发源选择：EOF、OUT_START、DC Sync0、DC Sync1。
40. 支持 µController PDI 的 BUSY 极性、BUSY 驱动、Default BUSY、Read BUSY Delay、Write on Falling Edge 等接口细节。
41. 支持 Avalon/AXI4/AXI4-Lite PDI 的异步外部总线时钟、可变数据宽度及可选独立 DC Sync 中断输出。
42. 支持 Device Emulation 功能。
43. 支持读写偏移寄存器、写保护寄存器、ECAT/PDI Reset 寄存器及 RESET_OUT 行为。
44. 支持扩展错误计数与 RX Error Code，包括 `0x0314:0x0327`。
45. 支持链路状态、端口状态、Forwarded RX Error、Lost Link Counter、PDI Error Code 等诊断状态。
46. 支持共享 MDIO/MI 总线或每端口独立 MI 接口。
47. 支持 PHY 地址偏移方式、独立 PHY 地址方式、动态导出 PHY 地址方式。
48. 支持 MI Link Detection and Configuration。
49. 支持 Enhanced Link Detection。
50. 支持原生 FX 端口能力所需的底层支持。
51. 支持 PHY Reset 输出与 PHY 初始化 ROM 接口。
52. 支持 MII 手动/自动 TX Shift 补偿。
53. 支持 RGMII 时序适配与 RMII 参考时钟配合。
54. 支持 GPIO。
55. 支持 RUN、ERR、STATE_RUN、Link/Activity LED。
56. 支持 RUN/ERR LED Override 与 LED Test。
57. 支持 `0x0F80:0x0FFF` 的特性镜像区，用于反映 ESC feature、port feature、PDI feature 等上电特征。
58. 支持产品 ID、Vendor ID、IP Core Revision/Build 等 ESC 专用标识信息。
59. 应明确不实现 V4.0.0 已废弃能力：
   - 内部三态驱动
   - AXI3 On-Chip Bus
   - PDI 直接控制的 DC System Time

### 3.2 性能需求

1. 帧处理流水线应支持 100 Mbps EtherCAT 线速运行。
2. 转发路径应采用流式处理架构；除协议明确要求外，不应采用整帧存储转发。
3. 内部 RAM 及仲裁设计应保证 EtherCAT、PDI0、PDI1、EEPROM 与 DC 事件处理具备确定性。
4. PDI 内部处理路径应支持 `8/16/32 bit` 内部总线宽度。
5. PDI 内部处理时钟应支持 `25/50/100 MHz` 配置。
6. DC 子系统应提供稳定的本地时间基准，并满足 EtherCAT DC 机制要求的高精度同步能力，至少达到优于微秒级的同步性能目标。

### 3.3 可配置性需求

RTL 应支持以下参数化配置：

- 物理端口数量：1/2/3/4
- 物理层接口类型：MII/RMII/RGMII
- FMMU 数量：0-16
- SyncManager 数量：0-16
- 过程数据 RAM 大小：0-60 KB
- PDI 数量：1/2
- PDI 类型
- PDI 内部总线宽度：8/16/32 bit
- PDI 内部时钟：25/50/100 MHz
- DC 使能/关闭
- DC 位宽：32/64 bit
- DC Sync 数量：0-4
- DC Latch 数量：0-4
- EEPROM 仿真使能/关闭
- User RAM 初始化使能/关闭
- GPIO 字节数：0/1/2/4/8
- LED 扩展功能使能/关闭
- 私有 PDI RAM 大小
- PHY 地址模式、MI 独立接口、PHY 初始化功能

### 3.4 验证与一致性需求

1. 实现应与参考资料中定义的 EtherCAT ESC 技术模型及寄存器模型保持一致。
2. 寄存器行为、复位值、访问权限及副作用应能够追溯至寄存器规范。
3. 设计结构应支持后续使用 EtherCAT Conformance Test Tool（CTT）开展一致性验证。
4. 实现应支持标准 EtherCAT 主站所依赖的、基于 ESI/EEPROM 的初始化过程。
5. 实现目标应与 ET1810/ET1811/ET1812 IP Core V4.0.0 的特性边界保持一致。

## 4. 功能分解

依据 ESC 技术资料与 ET1810/ET1811/ET1812 IP Core V4.0.0 功能表，完整 ESC 硬件可划分为以下功能域。

### 4.1 端口与 PHY 功能域

职责包括：

- 支持 MII、RMII、RGMII 三类物理接口封装
- Auto-Forwarder 与 Loopback 数据路径
- 链路检测与端口状态采集
- 环路控制及端口开启/关闭逻辑
- 端口间转发路径选择
- 对端口 0 与未使用端口的特殊处理支持
- MI/MDIO 管理接口
- 共享或每端口独立 PHY 管理
- PHY 地址配置与导出
- PHY Reset 输出
- PHY 初始化 ROM 接口
- MII TX Shift、RGMII RX/TX 时序适配、RMII 参考时钟配合
- Enhanced Link Detection、MI Link Detection、FX 端口支持

### 4.2 EtherCAT 帧处理功能域

职责包括：

- EtherCAT 帧识别
- Ethernet 帧头与 EtherCAT 数据报解析
- 地址模式处理
- 命令译码
- Working Counter 更新
- 读/写/读写命令执行
- 本地修改后的帧继续转发
- 环帧防护与帧处理顺序控制

该功能域是 ESC 的实时核心，必须采用低时延流水线结构实现。

### 4.3 寄存器与 CSR 功能域

职责包括：

- `0x0000` 至 `0x0FFF` 寄存器映射实现
- 读写译码
- 复位默认值管理
- 副作用处理
- 事件与状态回显
- 写保护与复位控制
- Product ID、Vendor ID、Revision/Build 等标识信息管理
- PDI Information、ESC Feature、Port Feature 上电信息寄存器/特性区管理

该模块是整个设计的控制平面核心。

### 4.4 过程数据存储功能域

职责包括：

- 面向 ECAT、PDI0、PDI1 的双口或仲裁式 RAM
- 寄存器空间与过程数据 RAM 空间分离
- 邮箱与 PDO 缓冲区分配
- 字节使能与对齐处理
- EtherCAT 访问、PDI 访问以及 DC 时间戳/事件访问之间的仲裁
- PDI private RAM 分区
- User RAM 初始化与特性镜像管理

### 4.5 FMMU 功能域

职责包括：

- 逻辑起始地址与长度匹配
- 逻辑位到物理字节/位的转换
- 方向控制
- 激活控制
- 支持输入/输出方向的独立映射
- 支持最多 16 个 FMMU 实例

该模块负责将 EtherCAT 逻辑内存访问转换为本地过程存储器访问。

### 4.6 SyncManager 功能域

职责包括：

- SyncManager 寄存器组实现
- 邮箱模式状态控制
- 缓冲模式与 3-buffer 模式处理
- 生产者/消费者所有权握手机制
- 缓冲区满/空/可用状态管理
- 按需支持 PDI 侧写确认机制
- 中断及看门狗触发生成
- Sequential Mode
- Deactivation Delay
- 多帧传输一致性保护

SyncManager 是网络侧与应用侧存储器访问一致性管理的关键模块。

### 4.7 AL 状态机功能域

职责包括：

- AL 控制请求接收
- 状态切换合法性检查
- AL Status 更新
- AL Status Code 生成
- 支持 `INIT`、`PRE-OP`、`SAFE-OP`、`OP`、`BOOT` 状态切换
- 与邮箱就绪、SyncManager 合法性、FMMU 合法性及输出安全门控联动
- Device Emulation 支持

重要边界如下：

- ESC 硬件负责提供 AL 硬件状态机及与 ESC 资源相关的状态切换检查。
- 与设备应用相关的状态确认可通过 PDI 交由外部固件完成。

### 4.8 中断与事件功能域

职责包括：

- ECAT Event Mask 与 Event Request 寄存器
- PDI0/PDI1 独立 AL Event Mask 与 AL Event Request
- 对 SyncManager、EEPROM、DC、看门狗及状态机事件进行聚合
- 向 PDI 或主机控制器输出中断信号
- 支持 PDI 中断写确认与仅中断写确认模式
- 支持 DC Sync 信号映射到 AL Event Request

### 4.9 看门狗功能域

职责包括：

- 看门狗分频器
- 过程数据看门狗
- PDI0/PDI1 看门狗
- 超时状态与计数器
- 双 PDI 看门狗组合策略
- 故障安全输出指示接口
- 数字输出在 EOF/OUT_START/DC Sync 触发条件下的安全控制

该模块对 `SAFE-OP` 与 `OP` 状态下输出有效性控制具有安全关键意义。

### 4.10 SII EEPROM 功能域

职责包括：

- 基于 I2C 的 EEPROM 访问
- ECAT/PDI 所有权控制
- 读/写/重载命令处理
- EEPROM Loaded 指示
- 可选的 EEPROM 仿真路径
- I2C 基地址配置
- I2C 频率配置
- 面向 PDI1 的 EEPROM Loaded 指示扩展

由于标准 EtherCAT 启动过程依赖 SII 数据，例如 Vendor ID、Product Code、Revision、Mailbox 默认配置以及 SyncManager/FMMU 默认配置，因此该模块是必需的。

### 4.11 Distributed Clocks 功能域

职责包括：

- 可选 32 位或 64 位系统时间计数器
- 端口接收时间戳采集
- 偏移与延时补偿寄存器
- 同步控制回路支持
- 最多 4 路 Sync 脉冲产生
- 最多 4 路 Latch 输入时间戳采集
- SyncManager 事件时间采集
- DC Receive Time 常开
- DC Speed Counter Direct Control
- SyncSignal 映射 AL Event/IRQ

若目标产品需要与驱动器或同步 I/O 实现高精度同步，则 DC 支持为必选能力。

### 4.12 PDI 功能域

职责包括：

- 支持最多两个 PDI：PDI0/PDI1
- 主机侧寄存器与 RAM 访问接口
- PDI 时序适配
- 读写事务处理
- 字节序与访问位宽转换
- 中断握手
- 支持 Digital I/O、SPI、异步 8/16/32 位 µController、Avalon、AXI4、AXI4-Lite
- 支持内部 PDI 总线宽度 8/16/32 bit 与 25/50/100 MHz 内核频率
- 支持 PDI user mode、PDI information、GPIO 及独立/组合 IRQ
- 支持 µController BUSY/IRQ 电气选项
- 支持 SPI 直接访问 SyncManager 的附加片选模式

### 4.13 GPIO 与 LED 功能域

职责包括：

- 通用输入/输出 GPIO 映射
- Link/Activity LED 输出
- RUN/ERR/STATE_RUN LED 输出
- RUN/ERR Override 支持
- LED Test 支持
- 与 AL 状态、错误状态、看门狗状态联动

## 5. 推荐的顶层 RTL 架构

建议的顶层 RTL 架构逻辑划分如下：

1. `esc_port_if`
2. `esc_frame_engine`
3. `esc_datagram_executor`
4. `esc_reg_bank`
5. `esc_pd_ram`
6. `esc_fmmu_array`
7. `esc_sm_array`
8. `esc_al_fsm`
9. `esc_event_irq`
10. `esc_watchdog`
11. `esc_eeprom_ctrl`
12. `esc_dc_unit`
13. `esc_pdi0_if`
14. `esc_pdi1_if`
15. `esc_phy_mi`
16. `esc_phy_init_rom_if`
17. `esc_gpio_led`
18. `esc_feature_rom`
19. `esc_reset_clock`

建议的数据流如下：

- PHY RX -> 端口接口 -> Auto-Forwarder / Loopback -> 帧处理引擎 -> 数据报执行单元
- 数据报执行单元 -> 寄存器组 / FMMU / SyncManager / 过程数据 RAM
- 处理结果 -> Working Counter 更新 -> 帧重写 -> 转发路径 -> PHY TX
- PDI0/PDI1 -> 寄存器组 / 过程数据 RAM / 私有 PDI RAM / AL 握手 / 中断状态
- DC、看门狗、EEPROM、SyncManager、GPIO/LED -> 事件/中断汇聚网络

## 6. 模块划分建议

### 6.1 顶层模块

`esc_top`

职责包括：

- 时钟/复位分发
- 实例化所有子模块
- 全局参数绑定
- 顶层总线与事件互联

### 6.2 实时数据通路模块

`esc_port_rx`

- MII/RMII/RGMII 接收适配
- 前导码/SFD/帧边界检测
- 接收错误上报

`esc_port_tx`

- MII/RMII/RGMII 发送适配
- 发送仲裁
- 按需实现 IFG 处理

`esc_forward_switch`

- 端口到端口路由
- Auto-Forwarder / Loopback 路径控制
- 环路与转发控制

`esc_frame_parser`

- EtherType 与 EtherCAT 帧识别
- 数据报边界解析

`esc_datagram_executor`

- EtherCAT 命令执行
- 地址译码
- 本地数据插入/提取
- Working Counter 处理

### 6.3 控制平面模块

`esc_reg_bank`

- 完整寄存器译码与存储
- 访问权限检查

`esc_reset_ctrl`

- ECAT 复位
- PDI 复位
- RESET_OUT 时序控制
- 本地软复位时序控制

`esc_status_ctrl`

- 链路、错误、LED、特性状态汇总
- User RAM 特性区状态镜像
- Product ID / Vendor ID / Revision / Build 汇总

### 6.4 存储与映射模块

`esc_pdpram`

- 过程数据与邮箱 RAM
- 双口或多主仲裁
- 私有 PDI RAM 划分
- User RAM 初始化控制

`esc_fmmu`

- 单个 FMMU 映射单元

`esc_fmmu_array`

- 多个 FMMU 实例及匹配选择

`esc_syncmanager`

- 单个 SyncManager 通道

`esc_sm_array`

- 多个 SyncManager 通道
- 邮箱/过程数据角色绑定
- Sequential Mode / Deactivation Delay 支持

### 6.5 AL 与应用接口模块

`esc_al_fsm`

- AL 状态切换控制
- 状态码生成
- Device Emulation 联动

`esc_pdi0_if`

- PDI0 接口封装
- 支持 Digital I/O / SPI / 异步 µController / Avalon / AXI4 / AXI4-Lite

`esc_pdi1_if`

- PDI1 接口封装
- 支持 Digital I/O / SPI / Avalon / AXI4 / AXI4-Lite

`esc_output_safe_ctrl`

- 在 `SAFE-OP`、看门狗超时或错误状态下实现输出有效性门控

### 6.6 服务类模块

`esc_irq_event`

- 事件屏蔽/请求处理
- 双 PDI 中断输出生成

`esc_watchdog`

- 看门狗定时器与超时事件
- PDI0/PDI1 组合看门狗状态

`esc_eeprom_ctrl`

- SII EEPROM 的 I2C 主控制器
- EEPROM 仿真桥接
- 重载时序控制

`esc_phy_mi`

- 共享或每端口独立 MDIO/MI 管理
- PHY 地址配置
- 链路配置读写

`esc_phy_init_rom_if`

- PHY 初始化 ROM 访问
- PHY 启动阶段寄存器初始化

`esc_dc_core`

- 系统时间、偏移、延时、时间戳采集
- DC Speed Counter Direct Control

`esc_sync_latch`

- 最多 4 路 Sync 产生与最多 4 路 Latch 输入采集

`esc_gpio_led`

- GPIO 输入输出
- Link/Activity、RUN、ERR、STATE_RUN LED 控制
- LED Test

`esc_feature_rom`

- `0x0F80:0x0FFF` 特性镜像
- `0x0FE0:0x0FFF` 端口特性镜像

## 7. 外部接口定义

### 7.1 EtherCAT 端口接口

对标版本应支持：

- `1-4` 路统一物理层类型封装
- 支持 `MII` / `RMII` / `RGMII`
- 链路状态输入
- PHY 复位输出
- 共享或每端口独立 MDIO/MI 管理接口
- PHY 地址配置输入
- PHY 初始化 ROM 接口
- MII TX Shift / RGMII 时钟适配输入

### 7.2 PDI 接口

完整对标版本应支持：

- PDI0
- PDI1
- 简单内部总线桥接
- SPI 从设备型 PDI
- 异步 8/16/32 位微控制器总线
- Avalon Memory-Mapped
- AXI4
- AXI4-Lite
- GPIO 扩展

### 7.3 EEPROM 接口

- I2C 主接口信号：`SCL`、`SDA`
- EEPROM 仿真桥接接口

### 7.4 DC 接口

- `SYNC0`、`SYNC1`、`SYNC2`、`SYNC3`
- `LATCH0`、`LATCH1`、`LATCH2`、`LATCH3`

### 7.5 LED 与 GPIO 接口

- `LED_LINK_ACT[0:3]`
- `LED_RUN`
- `LED_ERR`
- `LED_STATE_RUN`
- `GPIO_IN`
- `GPIO_OUT`

## 8. 地址映射规划

RTL 应预留并严格实现以下标准与对标寄存器区域：

- `0x0000` 至 `0x001F`：ESC 信息与站地址
- `0x0020` 至 `0x0041`：写保护与复位
- `0x0100` 至 `0x0139`：DL 与 AL 控制/状态
- `0x0140` 至 `0x015D`：PDI0 配置、PDI0 信息、PDI0 User Mode
- `0x0180` 至 `0x019D`：PDI1 配置、PDI1 信息、PDI1 User Mode
- `0x0200` 至 `0x0223`：中断/事件
- `0x0300` 至 `0x0342`：错误计数器
- `0x0400` 至 `0x0448`：看门狗
- `0x0500` 至 `0x051B`：EEPROM 与 PHY 管理
- `0x0600` 至 `0x06FF`：FMMU
- `0x0800` 至 `0x087F`：SyncManager
- `0x0900` 至 `0x09FF`：Distributed Clocks
- `0x0E00` 至 `0x0E0B`：Product ID / Vendor ID 等 ESC 专用标识
- `0x0F80` 至 `0x0FFF`：用户 RAM / ESC 专用特性区域
- `0x1000` 及以上：过程数据 RAM

寄存器详细行为应在下一阶段形成专门的寄存器规格说明文档。

## 9. 推荐开发策略

为降低实现风险，建议将开发划分为以下阶段。

### 阶段 1：通信与寄存器基线

目标包括：

- 端口接口与 Auto-Forwarder / Loopback
- 基础寄存器访问
- 站地址处理
- 基础数据报执行
- 过程数据 RAM 访问
- 基础 FMMU
- 基础 SyncManager
- EEPROM 启动链路

### 阶段 2：PDI 与状态闭环

目标包括：

- PDI0 完整化
- AL 状态切换
- 看门狗
- 中断/事件
- LED/GPIO
- `PRE-OP` / `SAFE-OP` / `OP` 状态支持

### 阶段 3：完整对标扩展

目标包括：

- PDI1
- 私有 PDI RAM
- Sequential Mode / Deactivation Delay
- 扩展错误计数与诊断
- EEPROM 仿真
- 特性镜像区

### 阶段 4：DC 与物理层高级功能

目标包括：

- 32/64 位 DC
- 4 路 Sync / 4 路 Latch
- DC Event Times
- DC Speed Counter Direct Control
- MII/RMII/RGMII 全适配
- PHY 初始化与 MI Link Detection

### 阶段 5：一致性加固

目标包括：

- 寄存器边界行为完善
- 复位副作用校核
- 邮箱边界条件处理
- 错误注入处理
- 面向 CTT 的修正与完善

## 10. 当前阶段的关键工程决策建议

1. 采用流式帧处理架构，避免整帧存储转发。
2. FMMU 与 SyncManager 实例采用参数化数组形式实现。
3. 严格分离数据平面与控制平面，降低时序风险。
4. 过程数据 RAM 采用独立存储子系统，并显式设计访问仲裁。
5. 邮箱协议语义默认放在 ESC 上层软件实现，硬件仅提供传输基础设施。
6. PDI0 与 PDI1 应独立封装，而不是在单一 `esc_pdi_if` 中硬编码复用。
7. 将 DC 设计为独立时序子系统，便于 32 位/64 位与 Sync/Latch 数量配置。
8. 以 Beckhoff 风格 ESC 地址映射兼容性作为首要兼容目标。
9. 以 ET1810/ET1811/ET1812 IP Core V4.0.0 作为参数上限与接口种类基线。
10. 不纳入 V4.0.0 已废弃能力：内部三态、AXI3、PDI 直接控制 DC System Time。

## 11. 风险与技术难点

本项目的主要技术风险包括：

- 在线 EtherCAT 帧修改过程中的时序闭合问题
- Auto-Forwarder / Loopback 转发路径与端口关闭回环行为
- SyncManager 邮箱模式、缓冲模式、Sequential Mode、Deactivation Delay 的精确实现
- ECAT、PDI0、PDI1、EEPROM 与 DC 多方并发访问仲裁
- EEPROM 仿真与真实 I2C EEPROM 双模式兼容
- MII/RMII/RGMII 多物理层统一封装
- PHY 管理、PHY 初始化与动态地址配置
- 看门狗行为及输出安全门控
- DC 同步精度、时间戳正确性及中断映射
- 与标准主站初始化序列和 IP Core Feature Mirror 行为的兼容性

因此，RTL 必须采用高度模块化、可独立验证的设计方式，并配套严格的寄存器规格与定向验证方案。

## 12. 下一阶段交付物

在本概要基础上，建议下一步按如下顺序输出后续设计文档与实现产物：

1. ESC 系统架构规格说明
2. 详细寄存器规格说明
3. 内存映射与 DPRAM 分配说明
4. 物理层与 PHY 管理规格说明
5. PDI0/PDI1 接口规格说明
6. GPIO/LED 规格说明
7. EEPROM/EEPROM 仿真规格说明
8. User RAM/Feature Mirror 规格说明
9. AL 状态机规格说明
10. FMMU 与 SyncManager 详细行为说明
11. DC 详细设计说明
12. RTL 模块接口定义
13. 验证计划与测试平台架构

## 13. ET1810 / ET1811 / ET1812 对标实现基线

为满足“完全对标 EtherCAT IP Core V4.0.0”的目标，最终 RTL 目标配置至少应覆盖以下能力集合：

1. `4` 个 EtherCAT 端口，支持统一选择 `MII`、`RMII` 或 `RGMII`。
2. `16` 个 FMMU。
3. `16` 个 SyncManager。
4. `60 KB` 过程数据 RAM。
5. `2` 个 PDI 接口。
6. PDI0 支持 `Digital I/O`、`SPI Slave`、`8/16/32-bit Async µC`、`Avalon`、`AXI4`、`AXI4-Lite`。
7. PDI1 支持 `Digital I/O`、`SPI Slave`、`Avalon`、`AXI4`、`AXI4-Lite`。
8. 支持 PDI private RAM、PDI user mode、PDI information、PDI ack-by-write、interrupt-only ack-by-write。
9. 支持 `32-bit` 与 `64-bit` DC，两种模式应可参数化选择。
10. 支持 `4` 路 SyncSignal 与 `4` 路 LatchSignal。
11. 支持 DC receive times、SyncManager event times、DC speed counter direct control。
12. 支持扩展错误计数器、RX error code、完整 LED 功能、EEPROM 仿真、PHY 初始化、独立 MI 接口。
13. 支持 `0x0F80:0x0FFF` 特性镜像区，用于输出当前配置特征。
14. 明确不实现 V4.0.0 已废弃能力：内部三态、AXI3、PDI 直接控制的 DC system time。

## 14. 总结

完整的 Verilog ESC 应构建为一个标准兼容并对标 ET1810/ET1811/ET1812 IP Core V4.0.0 的硬件通信内核，其核心能力包括：

- 帧处理
- Auto-Forwarder / Loopback
- 寄存器兼容
- FMMU 地址映射
- SyncManager 一致性控制
- AL 状态控制
- 看门狗与中断处理
- EEPROM 启动与 EEPROM 仿真
- 32/64 位 Distributed Clocks 高精度时序支持
- 面向上层应用逻辑的双 PDI 接口
- PHY/MII/RMII/RGMII 兼容物理层封装
- 特性镜像、GPIO、LED、扩展诊断与集成支持

从工程边界上，应明确如下原则：

- ESC 硬件负责实现 EtherCAT 通信基础设施与资源控制
- 邮箱传输机制由硬件提供
- 邮箱协议语义及设备 Profile 行为由上层固件/软件承担

在本次修订后，本文档的目标已经从“通用 ESC 设计概要”提升为“完整对标 ET1810/ET1811/ET1812 IP Core V4.0.0 功能边界的 ESC 设计概要”。后续详细设计应以本文件中的对标基线为准，不再以精简型 ESC 为边界。
