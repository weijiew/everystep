# Lab 1: Booting a PC

阅读：https://pdos.csail.mit.edu/6.828/2018/labs/lab1/

这个实验由三部分组成，第一部分主要是为了熟悉使用 x86 汇编语言、QEMU x86 仿真器、以及 PC 的加电引导过程。第二部分查看我们的 6.828 内核的引导加载器，它位于 lab 的 boot 目录中。第三部分深入到名为 JOS 的 6.828 内核模型内部，它在 kernel 目录中。

## 1. 环境配置

    % mkdir ~/6.828
    % cd ~/6.828
    % git clone https://pdos.csail.mit.edu/6.828/2018/jos.git lab
    Cloning into lab...
    % cd lab
    % 

接下来阅读 [tools](https://pdos.csail.mit.edu/6.828/2018/tools.html) 进行环境配置。


环境：WSL2 ubuntu20.04

    sudo apt-get install -y build-essential gdb
    sudo apt-get install gcc-multilib

    git clone https://github.com/mit-pdos/6.828-qemu.git qemu
    sudo apt-get install libsdl1.2-dev libtool-bin libglib2.0-dev libz-dev libpixman-1-dev
    ./configure --disable-kvm --disable-werror --target-list="i386-softmmu x86_64-softmmu"

出错：

    /usr/bin/ld: qga/commands-posix.o: in function `dev_major_minor':
    /home/yunwei/qemu/qga/commands-posix.c:633: undefined reference to `major'
    /usr/bin/ld: /home/yunwei/qemu/qga/commands-posix.c:634: undefined reference to `minor'
    collect2: error: ld returned 1 exit status

在 `qga/commands-posix.c` 文件中加上头文件: `#include<sys/sysmacros.h>`

    make && make install

进入 lab 报错：

    $ make
    + ld obj/kern/kernel
    ld: warning: section `.bss' type changed to PROGBITS
    ld: obj/kern/printfmt.o: in function `printnum':
    lib/printfmt.c:41: undefined reference to `__udivdi3'
    ld: lib/printfmt.c:49: undefined reference to `__umoddi3'
    make: *** [kern/Makefrag:71: obj/kern/kernel] Error 1

解决方案是安装 4.8 的 gcc ，但是报错，原因是这个包没有在这个源中。

    $ sudo apt-get install -y gcc-4.8-multilib
    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done
    E: Unable to locate package gcc-4.8-multilib
    E: Couldn't find any package by glob 'gcc-4.8-multilib'
    E: Couldn't find any package by regex 'gcc-4.8-multilib'

经过一番折腾，看到了这篇[文章](https://blog.csdn.net/feinifi/article/details/121793945)。简单来说就是这个包在 Ubuntu16.04 下可以正常下载，那么增加这个办的源即可。在 `/etc/apt/sources.list` 中添加如下内容：

    deb http://dk.archive.ubuntu.com/ubuntu/ xenial main
    deb http://dk.archive.ubuntu.com/ubuntu/ xenial universe

切记，需要更新

    sudo apt-get update

然后再次启动 qemu 依旧报错（此时已经过去一天了🥲）

    $ make
    + ld obj/kern/kernel
    ld: warning: section `.bss' type changed to PROGBITS
    ld: obj/kern/printfmt.o: in function `printnum':
    lib/printfmt.c:41: undefined reference to `__udivdi3'
    ld: lib/printfmt.c:49: undefined reference to `__umoddi3'
    make: *** [kern/Makefrag:71: obj/kern/kernel] Error 1

经过分析，发现 gcc 版本没有修改

    $ gcc --version
    gcc (Ubuntu 8.4.0-3ubuntu2) 8.4.0
    Copyright (C) 2018 Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.  There is NO
    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

于是将 gcc 版本改为 4.8 。删除原来的软连接，增加指向 4.8 版本的 软连接。查看版本更新成功。

    $ sudo rm /usr/bin/gcc
    $ sudo ln -s /usr/bin/gcc-4.8 /usr/bin/gcc
    $ gcc --version
    gcc (Ubuntu 4.8.5-4ubuntu2) 4.8.5
    Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.  There is NO
    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

再次编译，没有问题了！

    $ make
    + ld obj/kern/kernel
    ld: warning: section `.bss' type changed to PROGBITS
    + as boot/boot.S
    + cc -Os boot/main.c
    + ld boot/boot
    boot block is 380 bytes (max 510)
    + mk obj/kern/kernel.img

    $ sudo make qemu

至此，环境配置完成：

![20220417223135](https://cdn.jsdelivr.net/gh/weijiew/pic/images/20220417223135.png)

接下来继续阅读 lab1 ：https://pdos.csail.mit.edu/6.828/2018/labs/lab1/ 

使用 `make grade` 来测试，验证程序是否正确。

## Part 1: PC Bootstrap

介绍 x86 汇编语言和 PC 引导过程，熟悉 QEMU 和 QEMU/GDB 调试。不用写代码但是需要回答问题。

### Exercise 1.

熟悉汇编语言。

### The PC's Physical Address Space

`make qemu` 和 `make qemu-nox` 都是用来启动 qemu ，区别是后者不带图形界面。

    +------------------+  <- 0xFFFFFFFF (4GB)
    |      32-bit      |
    |  memory mapped   |
    |     devices      |
    |                  |
    /\/\/\/\/\/\/\/\/\/\

    /\/\/\/\/\/\/\/\/\/\
    |                  |
    |      Unused      |
    |                  |
    +------------------+  <- depends on amount of RAM
    |                  |
    |                  |
    | Extended Memory  |
    |                  |
    |                  |
    +------------------+  <- 0x00100000 (1MB)
    |     BIOS ROM     |    
    +------------------+  <- 0x000F0000 (960KB)
    |  16-bit devices, |
    |  expansion ROMs  |    
    +------------------+  <- 0x000C0000 (768KB)
    |   VGA Display    |    用于视频显示
    +------------------+  <- 0x000A0000 (640KB)
    |                  |
    |    Low Memory    |  早期 PC 唯一可以访问的区域
    |                  |  早期 PC 一般内存大小为 16KB, 32KB, 或 64KB
    +------------------+  <- 0x00000000


早期的 PC 是 16bit ，例如 8088 处理器，只能处理 1MB 的物理内存。所以物理空间从 0x00000000 开始到 0x000FFFFF 结束，并非是 0xFFFFFFFF 结束。

1. 其中前 640KB 是低内存，这是早期 PC 唯一可以随机访问的区域，此外早期 PC 的内存可以设置为 16KB，32KB 或 64KB 。

2. 从 0x000A0000 到 0x000FFFFF 这片内存区域留给硬件使用，例如视频显示的缓冲区，Basic Input/Output System (BIOS) 。起初这篇区域是用 ROM 来实现的，也就是只能读不能写，目前是用 flash 来实现，读写均可。此外 BIOS 负责初始化，完成后会将 OS 加载到内存中，然后将控制权交给 OS 。

随着时代的发展，PC 开始支持 4GB 内存，所以地址空间扩展到了 0xFFFFFFFF 。但是为了兼容已有的软件，保留了 0 - 1MB 之间的内存布局。0x000A0000 到 0x00100000 这区域看起来像是一个洞，前 640kb 是传统内存，剩余的部分是扩展内存。此外在 32 位下，PC 顶端的一些空间保留给 BIOS ，方便 32 位 PCI 设备使用。但是支持的内存空间已经超过了 4GB 的物理内存，也就是物理内存可以扩展到 0xFFFFFFFF 之上。但是依旧为了兼容 32 位设备的映射，在 32 位高地址部分留给 BIOS 的这片内存区域依旧保留，看起来像第二个洞。本实验中， JOS 只使用了前 256MB，可以假设只有 32 位的物理内存。

* 为什么 16 位 PC 的寻址空间是 1MB ？

以 8086 CPU 为例，数据线是 16 位($2^{16} = 64KB$)的，而地址线是 20 位( $2^{20} = 1MB$)。数据线决定了一次能获取的数据量，所以一次只能取 64KB，地址线决定了可寻址空间大小，所以寻址空间是 1MB 。这也解释了实模式下为什么段长是 64KB 。

* 新的问题产生了，寄存器都是 16 位的，怎么表示 20 位的地址？

既然一个寄存器无法表示那么就用两个寄存器来表示，也就是分段。将 1MB 的空间在逻辑上以 64KB 为单位切分，段长就是 64KB 。地址由段基地址和段内偏移两部分组成，其中段基址左移四位，加上段内偏移。

### The ROM BIOS

这一部分将会使用 qemu 的 debug 工具来研究计算机启动。

![20220504195748](https://cdn.jsdelivr.net/gh/weijiew/pic/images/20220504195748.png)

可以用 tmux 开两个窗口，一个窗口输入 `make qemu-nox-gdb` 另一个窗口输入 `make gdb` 摘取其中一行输入信息：

    [f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b

PC 从 0x000ffff0 开始执行，第一条要执行的指令是 jmp，跳转到分段地址 CS=0xf000 和 IP=0xe05b 。

起初因特尔是这样设计的，而 BIOS 处于 0x000f0000 和 0x000fffff 之间。这样设计确保了 PC 启动或重启都能获得机器的控制权。

QEMU 自带 BIOS 并且会将其放置在模拟的物理地址空间的位置上，当处理器复位时，模拟的处理器进入实模式，将 CS 设置为 0xf000，IP 设置为 0xfff0 。然后就在 CS:IP 段处开始执行。

分段地址 0xf000:ffff0 如何变成物理地址？这里面有一个公式：

    address = 16 * segment + offset

例如：

    16 * 0xf000 + 0xfff0   # in hex multiplication by 16 is
    = 0xf0000 + 0xfff0     # easy--just append a 0.
    = 0xffff0 

0xffff0 是 BIOS 结束前的16个字节（0x100000）。如果继续向后执行， 16 字节 BIOS 就结束了，这么小的空间能干什么？

### Exercise 2.

使用 gdb 的 si 指令搞清楚 BIOS 的大致情况，不需要搞清楚所有细节。

使用 si 逐行查看指令：

    [f000:fff0]    0xffff0: ljmp   $0xf000,$0xe05b  # 跳转到 `$0xfe05b` 处
    [f000:e05b]    0xfe05b: cmpl   $0x0,%cs:0x6ac8  # 若 0x6ac8 处的值为零则跳转
    [f000:e062]    0xfe062: jne    0xfd2e1
    [f000:e066]    0xfe066: xor    %dx,%dx          # 将 dx 寄存器清零
    [f000:e068]    0xfe068: mov    %dx,%ss          # 将 ss 寄存器清零
    [f000:e06a]    0xfe06a: mov    $0x7000,%esp     # esp = 0x7000 esp 始终指向栈顶
    [f000:e070]    0xfe070: mov    $0xf34c2,%edx    # edx = 0xf34c2 
    [f000:e076]    0xfe076: jmp    0xfd15c          # 跳转到 0xfd15c
    [f000:d15c]    0xfd15c: mov    %eax,%ecx        # ecx = eax
    [f000:d15f]    0xfd15f: cli                     # 关闭中断
    [f000:d160]    0xfd160: cld                     # 设置了方向标志，表示后续操作的内存变化
    [f000:d161]    0xfd161: mov    $0x8f,%eax       # eax = 0x8f  接下来的三条指令用于关闭不可屏蔽中断
    [f000:d167]    0xfd167: out    %al,$0x70        # 0x70 和 0x71 是用于操作 CMOS 的端口
    [f000:d169]    0xfd169: in     $0x71,%al        # 
    [f000:d16b]    0xfd16b: in     $0x92,%al        
    [f000:d16d]    0xfd16d: or     $0x2,%al
    [f000:d16f]    0xfd16f: out    %al,$0x92
    [f000:d171]    0xfd171: lidtw  %cs:0x6ab8       
    [f000:d177]    0xfd177: lgdtw  %cs:0x6a74       
    [f000:d17d]    0xfd17d: mov    %cr0,%eax        # eax = cr0
    [f000:d180]    0xfd180: or     $0x1,%eax        # 
    [f000:d184]    0xfd184: mov    %eax,%cr0
    [f000:d187]    0xfd187: ljmpl  $0x8,$0xfd18f
    => 0xfd18f:     mov    $0x10,%eax
    => 0xfd194:     mov    %eax,%ds
    => 0xfd196:     mov    %eax,%es

当 BIOS 启动的时候会先设置中断描述表，然后初始化各种硬件，例如 VGA 。

当初始化 PCI 总线和 BIOS 知晓的所有重要设备后，将会寻找一个可启动的设备，如软盘、硬盘或CD-ROM。

最终，当找到一个可启动的磁盘时，BIOS 从磁盘上读取 boot loader 并将控制权转移给它。

## Part 2: The Boot Loader

磁盘是由扇区组成，一个扇区为 512 B。磁盘的第一个扇区称为 boot sector ，这里面存放着 boot loader 。

BIOS 将 512B 的 boot sector 从磁盘加载到内存 0x7c00 到 0x7dff 之间。然后使用 jmp 指令设置 CS:IP 为 0000:7c00 最后将控制权传递给引导装载程序。

与 BIOS 的加载地址一样，这些地址是相当随意的--但它们对PC来说是固定的和标准化的。

在 6.828 中使用传统的硬盘启动机制，也就是 boot loader 不能超过 512B 。

boot loader 由汇编语言 `boot/boot.S` 和一个 C 语言文件 `boot/main.c` 组成。需要搞明白这两个文件的内容。

Boot Loader 负责两个功能：

1. boot loader 从实模式切换到 32 位的保护模式，因为只有在保护模式下软件才能访问超过 1MB 的物理内存。此外在保护模式下，段偏移量就变为了 32 而非 16 。

2. 其次，Boot Loader 通过 x86 的特殊 I/O 指令直接访问 IDE 磁盘设备寄存器，从硬盘上读取内核。

理解了 Boot Loader 的源代码后，看看 `obj/boot/boot.asm` 文件。这个文件是 GNUmakefile 在编译 Boot Loader 后创建的 Boot Loader 的反汇编。这个反汇编文件使我们很容易看到 Boot Loader 的所有代码在物理内存中的位置，也使我们更容易在 GDB 中跟踪 Boot Loader 发生了什么。同样的，`obj/kern/kernel.asm` 包含了 JOS 内核的反汇编，这对调试很有用。

在 gdb 中使用 b *0x7c00 在该地址处设置断点，然后使用 c 或 si 继续执行。c 将会跳转到下一个断点处，而 si 跳转到下一条指令，si N 则一次跳转 N 条指令。

使用 `x/Ni ADDR` 来打印地址中存储的内容。其中 N 是要反汇编的连续指令的数量，ADDR 是开始反汇编的内存地址。

### Exercise 3. 

阅读 [lab tools guide](https://pdos.csail.mit.edu/6.828/2018/labguide.html)，即使你已经很熟悉了，最好看看。

在 0x7c00 设置一个断点，启动扇区将会加载到此处。跟踪 `boot/boot.S` 并使用 `obj/boot/boot.asm` 来定位当前执行位置。使用 GDB 的 x/i 命令来反汇编 Boot Loader 中的指令序列并和 `obj/boot/boot.asm` 比较。

跟踪 boot/main.c 中的 bootmain() 函数，此后追踪到 readsect() 并研究对应的汇编指令，然后返回到 bootmain() 。确定从磁盘上读取内核剩余扇区的for循环的开始和结束。找出循环结束后将运行的代码，在那里设置一个断点，并继续到该断点。然后逐步完成 Boot Loader 的剩余部分。

* 回答下面的问题：

1. 在什么时候，处理器开始执行32位代码？究竟是什么原因导致从16位到32位模式的转换？

boot.S 57 line.

2. Boot Loader执行的最后一条指令是什么，它刚刚加载的内核的第一条指令是什么？

3. 内核的第一条指令在哪里？

4. Boot Loader如何决定它必须读取多少个扇区才能从磁盘上获取整个内核？它在哪里找到这些信息？

接下来进一步研究 `boot/main.c` 中的 C 语言部分。

### Exercise 4.

建议阅读 'K&R' 5.1 到 5.5 搞清楚指针，此外弄清楚 [pointers.c](https://pdos.csail.mit.edu/6.828/2018/labs/lab1/pointers.c) 的输出，否则后续会很痛苦。

需要了解 ELF 二进制文件才能搞清楚 `boot/main.c` 。

当编译链接一个 C 语言程序时，首先需要将 .c 文件编译为 .o 结尾的 object 文件，其中包含了相应的二进制格式的汇编指令。

链接器将所有的 .o 文件链接为单个二进制镜像，例如 `obj/kern/kernel` ，这是一个 ELF 格式的二进制文件，全称叫做 “Executable and Linkable Format” 。

此处可以简单的将 ELF 认为该文件头部带有加载信息，然后是是程序部分，每部分都是连续的代码或数据块，将指定的地址加载到内存中。Boot Loader 将其加载到内存中并开始执行。

ELF 的二进制文件头部的长度是固定的，然后是长度可变的程序头，其中包含了需要加载的程序部分。在 `inc/elf.h` 中包含了 ELF 文件头部的定义。

* .text: 程序指令.
* .rodata: 只读数据，例如由 C 编译器生成的 ASCII 字符常量。(这个只读并没有在硬件层面实现)
* .data: 数据部分，包含了程序初始化的数据，例如声明的全局变量 x = 5 。

当链接器计算一个程序的内存布局之时，它为没有初始化的程序保留了空间，例如 int x ，在内存中紧随.data之后的一个名为.bss的部分。C 默认未初始化的全局变量为零，所以 .bss 此时没有存储内容，因此链接器只记录 .bss 部分的地址和大小并将其置为零。

通过键入检查内核可执行文件中所有部分的名称、大小和链接地址的完整列表。


    athena% objdump -h obj/kern/kernel
    (If you compiled your own toolchain, you may need to use i386-jos-elf-objdump)

其中还包含了一些帮助调试的信息。

请特别注意.text部分的 "VMA"（或链接地址）和 "LMA"（或加载地址）。一个部分的加载地址是指该部分应该被加载到内存中的内存地址。

一个部分的链接地址是从等待执行的内存地址。

通常，链接和加载地址是相同的。例如阅读 boot loader 中的 .text 部分。

    athena% objdump -h obj/boot/boot.out

boot loader 根据 ELF 文件的头部决定加载哪些部分。程序头部指定了哪些信息需要加载及其地址。可以通过下面的命令来查看程序头部。

    athena% objdump -x obj/kern/kernel

程序头部已经在 "Program Headers" 下列出，ELF 对象的区域需要加载到内存中然后被标记为 "LOAD"。

每个程序头的其他信息也被给出，如虚拟地址（"vaddr"），物理地址（"paddr"），以及加载区域的大小（"memsz "和 "filesz"）。

回到 `boot/main.c` 每一个程序的 `ph->p_pa` 字段包含了段的物理地址。此处是一个真正的物理地址，尽管 ELF 对这个描述不清晰。

BIOS 将 boot sector 加载到内存中并从 0x7c00 处开始，这是 boot sector 的加载地址。boot sector 从这里开始执行。这也是 boot sector 执行的地方，所以这也是它的链接地址。

通过 -Ttext 0x7C00 设置链接地址在 boot/Makefrag 中，所以链接器将会在生成代码中生成正确的地址。

### Exercise 5.

再次追踪 Boot Loader 的前几条指令，找出第一条指令，如果把 Boot Loader 的链接地址弄错了，就会 "中断 "或做错事。然后把boot/Makefrag中的链接地址改成错误的，运行make clean，用make重新编译实验室，并再次追踪到boot loader，看看会发生什么。不要忘了把链接地址改回来，然后再做一次清理。

回头看内核加载和链接的地址，和 Boot Loader 不同的是，这两个地址并不相同。内核告诉 Boot Loader 在一个低的地址（1 兆字节）将其加载到内存中，但它希望从一个高的地址执行。我们将在下一节中深入探讨如何使这一工作。

此外 ELF 还有很多重要的信息。例如 e_entry 是程序 entry point 的地址。可以通过如下命令查看：

    athena% objdump -f obj/kern/kernel

此时应当理解 `boot/main.c` 中的 ELF loader 。它将内核的每个部分从磁盘上读到内存中的该部分的加载地址，然后跳转到内核的入口点。

### Exercise 6.

可以使用 GDB 的 x 命令来查看内存。此处知晓 `x/Nx ADDR` 就够用了，在ADDR处打印N个字的内存。

重新打开 gdb 检测，在 BIOS 进入 Boot Loader 时检查 0x00100000 处的 8 个内存字，然后在 Boot Loader 进入内核时再次检查。为什么它们会不同？在第二个断点处有什么？(你不需要用 QEMU 来回答这个问题，只需要思考一下。)

## Part 3: The Kernel

最初先执行汇编，然后为 C 语言执行做一些准备。

使用虚拟内存来解决位置依赖的问题。

当你在上面检查 Boot Loader 的链接地址和加载地址时，它们完全匹配，但是在内核的链接地址（由 objdump 打印的）和加载地址之间有一个（相当大的）差异。回去检查这两个地址，确保你能看到我们在说什么。(链接内核比启动加载器更复杂，所以链接和加载地址都在kern/kernel.ld的顶部)。

操作系统内核通常喜欢在非常高的虚拟地址上链接和运行，例如0xf0100000，以便将处理器的虚拟地址空间的低部分留给用户程序使用。这种安排的原因将在下一个实验中变得更加清楚。

许多机器在地址0xf0100000处没有任何物理内存，所以我们不能指望能在那里存储内核。相反，我们将使用处理器的内存管理硬件将虚拟地址 0xf0100000（内核代码期望运行的链接地址）映射到物理地址0x00100000（引导加载器将内核加载到物理内存的地方）。这样，尽管内核的虚拟地址足够高，可以为用户进程留下足够的地址空间，但它将被加载到物理内存中，位于PC的RAM的1MB处，就在BIOS ROM上方。这种方法要求PC至少有几兆字节的物理内存（这样物理地址0x00100000才行），但这可能是1990年以后制造的任何PC的真实情况。

事实上，在下一个实验中，我们将把PC的整个底部256MB的物理地址空间，从物理地址0x00000000到0x0fffffff，分别映射到虚拟地址0xf0000000到0xffffffff。现在你应该明白为什么JOS只能使用前256MB的物理内存了。


现在，我们只需映射前4MB的物理内存，这就足以让我们开始运行。我们使用`kern/entrypgdir.c`中手工编写的、静态初始化的页目录和页表来做这件事。现在，你不需要了解这个工作的细节，只需要了解它的效果。在`kern/entry.S`设置CR0_PG标志之前，内存引用被视为物理地址（严格来说，它们是线性地址，但`boot/boot.S`设置了从线性地址到物理地址的身份映射，我们永远不会改变）。一旦CR0_PG被设置，内存引用就是虚拟地址，被虚拟内存硬件翻译成物理地址。 entry_pgdir将0xf0000000到0xf0400000范围内的虚拟地址翻译成物理地址0x00000000到0x00400000，以及虚拟地址0x00000000到0x00400000到物理地址0x00000000到0x00400000。任何不在这两个范围内的虚拟地址都会引起硬件异常，由于我们还没有设置中断处理，这将导致QEMU转储机器状态并退出（如果你没有使用6.828补丁版本的QEMU，则会无休止地重新启动）。


### Exercise 7.

使用QEMU和GDB追踪到JOS的内核，在 `movl %eax, %cr0` 处停止。检查0x00100000和0xf0100000处的内存。现在，使用stepi GDB命令对该指令进行单步操作。再次，检查0x00100000和0xf0100000处的内存。确保你明白刚刚发生了什么。

在新的映射建立后的第一条指令是什么，如果映射没有建立，它将不能正常工作？把`kern/entry.S`中的`movl %eax, %cr0`注释出来，追踪到它，看看你是否正确。

大多数人认为printf()这样的函数是理所当然的，有时甚至认为它们是C语言的 "原语"。但是在操作系统的内核中，我们必须自己实现所有的I/O。

阅读kern/printf.c、lib/printfmt.c和kern/console.c，并确保你理解它们之间的关系。在后面的实验中会清楚为什么 printfmt.c 位于单独的 lib 目录中。

### Exercise 8.

练习 8. 我们省略了一小段代码--使用"%o "形式的模式打印八进制数字所需的代码。找到并填入这个代码片段。

能够回答以下问题: 

1. 解释一下printf.c和console.c之间的接口。具体来说，console.c输出了什么函数？这个函数是如何被printf.c使用的？
2. 从console.c中解释如下：

    1      if (crt_pos >= CRT_SIZE) {
    2              int i;
    3              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
    4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
    5                      crt_buf[i] = 0x0700 | ' ';
    6              crt_pos -= CRT_COLS;
    7      }

3. 对于下面的问题，你可能希望参考第2讲的注释。这些笔记涵盖了GCC在X86上的调用惯例。

    逐步跟踪以下代码的执行。

    在对cprintf()的调用中，fmt指向什么？ap指的是什么？

    列出对cons_putc、va_arg和vcprintf的每个调用（按执行顺序）。对于cons_putc，也要列出其参数。对于va_arg，列出调用前后ap所指向的内容。对于vcprintf，列出其两个参数的值。

4. 运行以下代码。

    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);


输出是什么？解释一下这个输出是如何按照前面练习的方式一步步得出的。这里有一个ASCII表，将字节映射到字符。

这个输出取决于x86是小字节的事实。如果x86是big-endian的，你要把i设置成什么样子才能产生同样的输出？你是否需要将57616改为不同的值？

这里有一个关于小-序数和大-序数的描述，还有一个更奇特的描述。

5. 在下面的代码中，'y='后面要打印什么？(注意：答案不是一个具体的数值。)为什么会出现这种情况？

    cprintf("x=%d y=%d", 3);

假设GCC改变了它的调用惯例，使它按声明顺序把参数推到堆栈上，这样最后一个参数就被推到了。你要如何改变cprintf或它的接口，使它仍然有可能传递可变数量的参数？


* 挑战 加强控制台，允许用不同颜色打印文本。传统的方法是让它解释嵌入在打印到控制台的文本字符串中的ANSI转义序列，但你可以使用任何你喜欢的机制。在6.828参考页和网络上其他地方有很多关于VGA显示硬件编程的信息。如果你觉得很冒险，你可以尝试将VGA硬件切换到图形模式，让控制台在图形帧缓冲区上绘制文本。

在本实验的最后一个练习中，我们将更详细地探讨C语言在x86上使用堆栈的方式，并在此过程中编写一个有用的新内核监控函数，打印出堆栈的回溯：从导致当前执行点的嵌套调用指令中保存的指令指针（IP）值列表。


### Exercise 9. 

确定内核在哪里初始化它的堆栈，以及它的堆栈在内存中的确切位置。内核是如何为其堆栈保留空间的？堆栈指针被初始化为指向这个保留区域的哪个 "末端"？

x86的堆栈指针（esp寄存器）指向堆栈上目前正在使用的最低位置。在为堆栈保留的区域中，低于该位置的所有内容都是空闲的。将一个值推入堆栈包括减少堆栈指针，然后将该值写入堆栈指针指向的位置。从堆栈中弹出一个值包括读取堆栈指针指向的值，然后增加堆栈指针。在32位模式下，堆栈只能容纳32位的值，而esp总是能被4整除。各种x86指令，如调用，都是 "硬接线 "来使用堆栈指针寄存器。

x86的堆栈指针（esp寄存器）指向堆栈上目前正在使用的最低位置。在为堆栈保留的区域中，低于该位置的所有内容都是空闲的。将一个值推入堆栈包括减少堆栈指针，然后将该值写入堆栈指针指向的位置。从堆栈中弹出一个值包括读取堆栈指针指向的值，然后增加堆栈指针。在32位模式下，堆栈只能容纳32位的值，而esp总是能被4整除。各种x86指令，如调用，都是 "硬接线 "来使用堆栈指针寄存器。

### Exercise 10. 

为了熟悉x86上的C语言调用习惯，在`obj/kern/kernel.asm`中找到`test_backtrace`函数的地址，在那里设置一个断点，并检查内核启动后每次调用该函数时发生的情况。test_backtrace的每个递归嵌套层在堆栈上推多少个32位字，这些字是什么？

请注意，为了使这个练习正常进行，你应该使用工具页上或Athena上的QEMU的补丁版本。否则，你将不得不手动将所有断点和内存地址转换为线性地址。

上面的练习应该给你提供了实现堆栈回溯函数所需的信息，你应该把它叫做mon_backtrace()。这个函数的原型已经在kern/monitor.c中等着你了。你可以完全用C语言完成，但你可能会发现inc/x86.h中的read_ebp()函数很有用。你还必须把这个新函数挂到内核监视器的命令列表中，这样它就可以被用户交互地调用。

回溯函数应该以下列格式显示函数调用框架的清单。

        Stack backtrace:
        ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
        ebp f0109ed8  eip f01000d6  args 00000000 00000000 f0100058 f0109f28 00000061
        ...

每一行都包含一个ebp、eip和args。ebp值表示该函数所使用的进入堆栈的基本指针：即刚进入函数后堆栈指针的位置，函数序言代码设置了基本指针。列出的eip值是该函数的返回指令指针：当函数返回时，控制将返回到该指令地址。返回指令指针通常指向调用指令之后的指令（为什么呢）。最后，在args后面列出的五个十六进制值是有关函数的前五个参数，这些参数在函数被调用之前会被推到堆栈中。当然，如果函数被调用时的参数少于5个，那么这5个值就不会全部有用。(为什么回溯代码不能检测到实际有多少个参数？如何才能解决这个限制呢？）


打印的第一行反映当前执行的函数，即mon_backtrace本身，第二行反映调用mon_backtrace的函数，第三行反映调用该函数的函数，以此类推。你应该打印所有未完成的堆栈帧。通过研究kern/entry.S，你会发现有一种简单的方法可以告诉你何时停止。


以下是你在《K&R》第五章中读到的几个具体要点，值得你在下面的练习和今后的实验中记住。

1. 如果int *p = (int*)100，那么(int)p + 1和(int)(p + 1)是不同的数字：第一个是101，但第二个是104。当把一个整数加到一个指针上时，就像第二种情况一样，这个整数隐含地乘以指针所指向的对象的大小。
2. p[i]被定义为与*(p+i)相同，指的是p所指向的内存中的第i个对象。当对象大于一个字节时，上面的加法规则有助于这个定义发挥作用。
3. &p[i]与(p+i)相同，产生p所指向的内存中的第i个对象的地址。

s尽管大多数C语言程序不需要在指针和整数之间进行转换，但操作系统经常需要。每当你看到涉及内存地址的加法时，要问自己这到底是整数加法还是指针加法，并确保被加的值被适当地乘以。



### Exercise 11. 

按照上面的规定实现回溯功能。使用与例子中相同的格式，因为否则评分脚本会感到困惑。当你认为你的工作正常时，运行make grade，看看它的输出是否符合我们的评分脚本所期望的，如果不符合，就修正它。在你交出lab1的代码后，欢迎你以任何方式改变回溯函数的输出格式。

如果你使用read_ebp()，注意GCC可能会生成 "优化 "代码，在mon_backtrace()的函数序幕之前调用read_ebp()，这将导致不完整的堆栈跟踪（最近的函数调用的堆栈帧丢失）。虽然我们已经尝试禁用导致这种重新排序的优化，但你可能想检查mon_backtrace()的汇编，确保对read_ebp()的调用发生在函数序幕之后。

在这一点上，你的回溯函数应该给你堆栈上导致mon_backtrace()被执行的函数调用者的地址。然而，在实践中，你经常想知道这些地址所对应的函数名称。例如，你可能想知道哪些函数可能包含一个导致内核崩溃的错误。

为了帮助你实现这一功能，我们提供了函数debuginfo_eip()，它在符号表中查找eip并返回该地址的调试信息。这个函数在kern/kdebug.c中定义。

在这一点上，你的回溯函数应该给你堆栈上导致mon_backtrace()被执行的函数调用者的地址。然而，在实践中，你经常想知道这些地址所对应的函数名称。例如，你可能想知道哪些函数可能包含一个导致内核崩溃的错误。

为了帮助你实现这一功能，我们提供了函数debuginfo_eip()，它在符号表中查找eip并返回该地址的调试信息。这个函数在kern/kdebug.c中定义。

### Exercise 12. 

修改你的堆栈回溯函数，为每个eip显示函数名、源文件名和与该eip对应的行号。

在debuginfo_eip中，__STAB_*来自哪里？这个问题有一个很长的答案；为了帮助你发现答案，这里有一些你可能想做的事情。

* 在kern/kernel.ld文件中查找__STAB_*。
* 运行 objdump -h obj/kern/kernel
* 运行objdump -G obj/kern/kernel
* 运行gcc -pipe -nostdinc -O2 -fno-builtin -I. -MD -Wall -Wno-format -DJOS_KERNEL -gstabs -c -S kern/init.c，并查看init.s。
* 看看bootloader是否在内存中加载符号表作为加载内核二进制的一部分
* 完成debuginfo_eip的实现，插入对stab_binsearch的调用，以找到地址的行号。

在内核监控器中添加一个回溯命令，并扩展你的mon_backtrace的实现，以调用debuginfo_eip并为每个堆栈帧打印一行。

    K> backtrace
    Stack backtrace:
    ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
            kern/monitor.c:143: monitor+106
    ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
            kern/init.c:49: i386_init+59
    ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
            kern/entry.S:70: <unknown>+0
    K> 

每一行都给出了stack frame 的 eip 文件名和在该文件中的行数，然后是函数的名称和 eip 与函数第一条指令的偏移量（例如，monitor+106表示返回eip比monitor的开头多106字节）。

请确保将文件和函数名单独打印在一行，以避免混淆 grading 脚本。

提示：printf格式的字符串提供了一种简单但不明显的方法来打印非空尾的字符串，如STABS表中的字符串。 printf("%.*s", length, string)最多能打印出字符串的长度字符。看一下printf手册，了解为什么这样做。

你可能会发现在回溯中缺少一些函数。例如，你可能会看到对monitor()的调用，但没有对runcmd()的调用。这是因为编译器对一些函数的调用进行了内联。其他优化可能导致你看到意外的行数。如果你把GNUMakefile中的-O2去掉，回溯可能会更有意义（但你的内核会运行得更慢）。


这样就完成了实验室的工作。在实验室目录中，用git commit提交你的修改，然后输入make handin提交你的代码。
