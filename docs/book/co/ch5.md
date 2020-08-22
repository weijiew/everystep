# 第五章：输入输出系统

## 5.1 概述

### 输入输出系统的发展概况

1. 早期

分散连接
CPU 和 IO 设备串行工作，程序查询方式。

2. 接口模块和 DMA 阶段

总线连接
CPU 和 IO 设备并行工作，中断方式，DMA方式

3. 具有通道结构的阶段

4. 具有 I/O 处理机的阶段

微处理器或直接用主机的处理器来执行。

### 输入输出系统的组成

1. I/O 软件

* I/O 指令，CPU 指令的一部分。操作码，命令码，设备码。

* 通道指令，通道自身的指令。指出数组的首地址，传送字数，操作命令。

2. I/O 硬件

设备，I/O 接口

设备，设备控制器  通道。

### I/O 设备于主机的联系方式

1. I/O 设备的编址方式。

* 统一编址：直接采用 CPU 的取数，存数指令即可。
* 不统一编址：单独编址，针对 I/O 设备单独编写一套指令。

2. 设备选址

用设备选择电路识别是否被选中。选中后传输数据。

3. 传送方式

* 串行：数据逐个传送。
* 并行：多条数据同时传送。

4. 联络方式

* 立即响应
* 异步工作应答信号
* 同步工作的方式    

5. I/O 设备与主机的连接方式

* 辐射式连接，每台设备都配一套控制线路的信号线。早期设备少可以这样搞，但是后期设备多了。导致可移植性差，并且不利于增删设备可扩展性差。
* 总线连接，外部设备的数据存放到缓存中。


### 5.1.4 I/O 设备与主机信息传送的控制方式

1. 程序查询方式

CPU 不断的访问 I/O ，查询状态，经过 CPU 写入内存中。

2. 程序中断方式

CPU 不查询，当 CPU 在执行的时候发现了 启动 I/O 设备的指令，于是 CPU 



3. DMA 方式

## 5.2 IO 设备

## 5.3 IO 接口

## 5.4 程序查询方式

## 5.5 程序中断方式

## 5.6 DMA方式
