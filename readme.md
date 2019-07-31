## Latest Update on Sentry

- Everything related to chassis movement logic will be put in `chassis_calc.c` from now on.

## 概述

- en doc   [readme](doc/en/readme.md) #TODO: Update it

此版本中，各I/O配置、时钟配置、实时操作系统的实现和整套程序框架均来自RoboMaster/RoboRTS-Firmware，在此对RoboMaster官方的编写和维护表示感谢。
香港大学机甲大师战队成员所做的更改均在注释中进行了标识，并遵循GNU通用公共许可进行使用和开源。

### 软件环境

 - Toolchain/IDE : MDK-ARM V5
 - package version: STM32Cube FW_F4 V1.21.0
 - FreeRTOS version: 9.0.0
 - CMSIS-RTOS version: 1.02

### 编程规范

- 变量和函数命名方式遵循 Unix/Linux 风格
- chassis\_task与gimbal\_task是强实时控制任务，优先级最高，禁止被其他任务抢占或者阻塞
- 不需要精确计时的任务，采用自行实现的软件定时器实现，定时精度受任务调度影响

### 注意事项

1. 由于发射机构触动开关阻力非常小，轻微转动拨弹电机就可能造成子弹越过触动开关触碰摩擦轮，导致摩擦轮摩擦过大无法启动，**造成摩擦轮磨损和电机启动失败**。因此，开电前务必检查是否有子弹已经越过触动开关。**取出办法**：用手按住一侧摩擦轮，旋转另一个摩擦轮。
**为防止此情况在比赛中出现，拨弹电机在摩擦轮未开启之前不能转动，同时每次关闭摩擦轮时，摩擦轮电机将延迟一秒停转；以上均由软件实现，是为正常现象**

1. 国际开发板A型上有一枚跳线帽，用于短接板载4pin排针串口R与G（只有底盘需要短接R和G）。固件通过R引脚是否拉低区分底盘还是云台。出现不能控制且无声光报警请检查跳线帽是否松动，

### 云台校准方法

底层代码集成了云台自动校准功能，原理是先利用imu校准pitch，控制云台保持在水平面，当误差小于0.15度时，记录pitch中值，此过程大概花费20~30s；然后YAW向左旋转直到触碰左侧限位，再向右旋转至右侧限位，计算YAW中值。

触发条件：

1.开发板首次刷入程序或者参数区被清空

2.按下板载白色按键触发

3.上位机发送协议命令触发

注意：校准时务必将底盘放在水平地面；校准中可能出现pitch始终无法到达指定误差范围内，此时是因为云台配重问题导致，可以用手向上掰一下然后松开云台，加速调节过程。

### 模块离线说明

当车辆的某个模块离线时，可以根据开发板蜂鸣器的不同状态判断哪个模块出现了问题，并进行错误定位

蜂鸣器鸣叫次数按照离线模块的优先级进行错误指示，例如云台电机优先级高于拨弹电机，如果同时发生离线，先指示当前离线设备是云台电机

模块离线对应的状态如下，数字对应蜂鸣器每次鸣叫的次数，按照优先级排序：

#### 底盘模块

1. 右前轮电机掉线
2. 左前轮电机掉线
3. 左后轮电机掉线
4. 右后轮电机掉线

#### 云台模块

5. 云台 YAW 电机掉线
6. 云台 PITCH 电机掉线
7. 拨盘电机掉线
8. 【英雄专用】上拨弹电机掉线

#### 遥控器离线

此时红灯常亮

### 文档

- 协议文档  [protocol](doc/ch/protocol.md)
- protocol [document](doc/en/protocol.md)

## 快速开始

### 硬件接口

主控板使用 RM 开发板 A 型，各个功能接口的位置如下：

**底盘硬件接口**

注意：uwb、单轴陀螺仪、通信can2硬件上使用一个can，无需考虑接线顺序。
![](doc/image/chassis.PNG)

单轴陀螺仪本版本中无需使用

**云台硬件接口**

![](doc/image/gimbal.PNG)

注意：步兵机器人上位机连接至云台控制板USB接口；裁判系统连接至底盘UART接口
注意：英雄机器人上位机连接至底盘UAB接口

### 功能模块

#### 失能模式：

所有电机、电容设备均不能使用。

#### 手动模式：

提供遥控器、键盘、鼠标控制。

#### 计算机视觉辅助模式：

这种模式下云台将接受上位机的辅助瞄准信号。

#### 操作档位说明：

##### 右拨杆

- 上：计算机辅助瞄准 （亦可由按住鼠标右键触发）
- 中：手动控制
- 下：失能

##### 左拨杆

- 由中间推上：开启/关闭摩擦轮和激光照准，并解除/开启射击保险（亦可由F键开启，Z+F键关闭）
- 中：关闭弹仓盖
- 下：打开弹仓盖（亦可由R键开启）

##### 拨轮

- 上推：单发
- 下拉：连发

- （在按下Z X键时）：调整静止状态下云台和底盘间差角
- （在按下左SHIFT Z X键时）：抵消云台偏航轴（yaw轴）IMU漂移

##### 键盘

- W 前进
- A 左平移
- S 后退
- D 右平移
- Q 逆时针自旋
- E 顺时针自旋
- R 打开弹仓盖（与下拉左拨杆等效）
- F 开启摩擦轮和激光照准，解除射击保险（与在摩擦轮关闭状态下上推左拨杆等效）
- V 避弹
- X 云台归中稳定（在云台失控时强制使云台解脱）
- Z+F 关闭摩擦轮和激光照准，开启射击保险（与在摩擦轮开启状态下上推左拨杆等效）
- Z+X 配合遥控器拨轮，调整静止状态下云台和底盘间差角
- LEFT SHIFT+Z+X 配合遥控器拨轮，抵消云台偏航轴（yaw轴）IMU漂移
- LEFT SHIFT 高速
- LEFT CTRL 低速


##### 鼠标

- 平移 云台旋转
- 单击左键 单发
- 长按左键 连发
- 长按右键 计算机辅助瞄准（与上推右拨杆等效）

## 程序说明

### 程序体系结构

#### 体系框架

1. 使用免费及开源的 freertos 操作系统，兼容其他开源协议 license；
2. 使用标准 CMSIS-RTOS 接口，方便不同操作系统或平台间程序移植；
3. 提供一套抽象步兵机器人bsp，简化上层逻辑；

![](doc/image/frame.png)

**Driver**：直接操作底层硬件的设备驱动，在库函数和寄存器基础上，增加锁和异步等机制,提供更加易于使用的api。

**Device**：具有一种或者多种总线输入，一种或者多种数据输出的外部模块，或者通用的软件模块（掉线保护）。目前主要针对RM物资的驱动抽象成设备。

**Controller**：单输入单输出模型，提供一些公用的调用接口，更换不同的控制算法取决于不同的注册函数。

**Algorithm**：提供模块算法，基本文件间无相互依赖。

**Module**：由driver、device、controller、algorithm组成的，可以实现某种特殊功能的模块。比如RM电机组成的二轴云台、RM电机构成的麦轮底盘。

**Utilities**：通用系统组件，比如log系统。

**Protocol**：面向接口的一套上层和底层通信协议。

**Application**：上层应用逻辑，包括各类模式切换。

### 软件体系

#### 程序启动

程序启动通过国际开发板A型的跳线帽区分启动任务
![](doc/image/startup.png)

### 继承关系

![](doc/image/object.png)

### 硬件体系

1. 主控 MCU：STM32F427IIHx，配置运行频率180MHz
1. 模块通信方式：CAN1；CAN1设备：电机电调
1. 控制板通信方式：CAN2 （包含遥控指令转发，裁判系统的机器人等级、功率和热量数据和上位机信号等）
1. 麦轮安装方式：X型

### 协议数据

#### 数据分类

协议数据按照通信方向可以分为三大类：

* 控制板接收的裁判系统信息

参照裁判系统学生协议

* 控制板接收的上位机数据：

1. 控制信息：上位机对底层 3 个执行机构的控制信息；

* 控制板间通讯

#TODO: Inter communication & routing

