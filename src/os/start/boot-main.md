接下来是 Boot Loader 的 C 语言部分，此时处理器切换到保护模式，Boot Loader 需要加载内核到内存，并跳转到内核的入口点开始执行。

接下来需要弄清楚几个问题，内核加载到内存后存放在哪里？内核本身是如何组织的？C 语言读取内核到内存的细节。

### 内核加载到内存后存放在哪里？

下面这段代码定义了一个宏`ELFHDR`，它将一个内存地址（0x10000）转换为一个`Elf`结构体的指针。这个地址通常被称为暂存区（scratch space），在引导加载程序中，它被用于临时存储和处理从硬盘读取的内核映像，直到内核映像被加载到其最终目标位置。

```c
// 暂存区（scratch space），用于加载和存储 ELF（Executable and Linkable Format）格式的内核映像。
//  在引导加载程序中，这个地址用于临时存储和处理从硬盘读取的内核映像，直到它被加载到其最终目标位置。
#define ELFHDR		((struct Elf *) 0x10000)
```

`Elf`是 Executable and Linkable Format 的缩写，它是一种常见的文件格式，用于存储程序或者其他类型的可执行文件。在这个上下文中，`Elf`可能是一个 C 结构体，用于表示 ELF 文件的头部（header）。

这个宏的主要用途是提供一个方便的方式来访问这个暂存区中的 ELF 头部。例如，你可以使用`ELFHDR->e_entry`来访问 ELF 头部中的`e_entry`字段，这个字段通常包含程序的入口点。

### ELF

ELF（Executable and Linkable Format）是一种常见的文件格式，用于存储程序或者其他类型的可执行文件。下面是 JOS 中定义 ELF 对应的结构体字段。 这个结构体的主要用途是解析 ELF 文件，通过读取并解析这些字段，我们可以获取到 ELF 文件的各种重要信息，如程序入口点、程序头表和节头表的位置等。

```c
struct Elf {
    uint32_t e_magic;    // 必须等于ELF_MAGIC，是一个魔数，用于验证文件是否为ELF格式
    uint8_t e_elf[12];   // ELF标识，包含了文件类别、数据编码方式等信息
    uint16_t e_type;     // 文件类型，指明是可执行文件、可重定位文件还是共享对象文件等
    uint16_t e_machine;  // 目标机器类型，指明了文件是为哪种处理器设计的
    uint32_t e_version;  // ELF版本信息
    uint32_t e_entry;    // 程序入口的虚拟地址，如果没有入口点则为0
    uint32_t e_phoff;    // 程序头表的文件偏移量，如果没有程序头表则为0
    uint32_t e_shoff;    // 节头表的文件偏移量，如果没有节头表则为0
    uint32_t e_flags;    // 与处理器相关的标志
    uint16_t e_ehsize;   // ELF头的大小，字节为单位
    uint16_t e_phentsize;// 程序头表中每个条目的大小，字节为单位
    uint16_t e_phnum;    // 程序头表的条目数
    uint16_t e_shentsize;// 节头表中每个条目的大小，字节为单位
    uint16_t e_shnum;    // 节头表的条目数
    uint16_t e_shstrndx; // 节头字符串表在节头表中的索引，用于定位每个节的名称
};
```

下面是展示了一个简化的 ELF 文件在内存中的布局。

```
+------------------------+
|       ELF Header       |
+------------------------+
|       Program Header   |
|       Table (PH)       |
+------------------------+
|        Section         |
|        Header          |
|        Table (SH)      |
+------------------------+
|        .text Section   |
|        (Code)          |
+------------------------+
|        .data Section   |
|      (Initialized Data)|
+------------------------+
|        .bss Section    |
|    (Uninitialized Data)|
+------------------------+
|       Other Sections   |
|           ...          |
+------------------------+
```

以下是各个部分的简要解释：

- **ELF Header：** 包含了 ELF 文件的基本信息，如文件类型、目标机器等。

- **Program Header Table (PH)：** 包含了可执行文件在内存中的加载信息，每个条目描述了一个段的大小、在文件中的偏移以及在内存中的位置等。

- **Section Header Table (SH)：** 包含了各个节的信息，每个条目描述了一个节的大小、在文件中的偏移、在内存中的位置等。

- **Sections：** 包含了实际的代码和数据，如：
  - **.text Section：** 存储可执行代码和指令。
  - **.data Section：** 存储已初始化的数据。
  - **.bss Section：** 存储未初始化的数据，占用的空间在运行时被初始化为零。
  - **Other Sections：** 其他可能存在的节，包括符号表、字符串表等。

这个布局展示了 ELF 文件在加载到内存中时的整体结构，每个部分的内容在执行过程中会根据程序的需要被加载和执行。

### 引导加载程序如何确定它必须读取多少个扇区才能从磁盘中获取整个内核？它在哪里可以找到这些信息？

在 JOS 中，引导加载器（boot loader）通过读取 ELF（Executable and Linkable Format）格式的内核映像文件头部来确定需要读取多少个扇区以获取整个内核。ELF 文件头部包含了一些元数据，其中就包括了内核的大小。

这个过程在 JOS 的源代码中的 `boot/main.c` 文件中有具体的实现。

### 内核的第一条指令在哪里？

在 JOS 操作系统中，内核的第一条指令位于内核映像的入口点。这个入口点是在链接内核时由链接器确定的，通常在链接脚本中指定。链接脚本定义了各个段（如代码段、数据段等）在内核映像中的布局和顺序。

在 JOS 的链接脚本（通常是一个名为 `kernel.ld` 的文件）中，有一行类似于 `ENTRY(start)` 的代码，这里的 `start` 就是内核的入口点，也就是内核的第一条指令的位置。

当引导加载器加载内核到内存并跳转到内核入口点时，就会开始执行内核的第一条指令。这个过程在 JOS 的引导加载器的源代码中有具体的实现。

### 执行的引导加载程序的最后一条指令是什么，它刚刚加载的内核的第一条指令是什么？

在 JOS 操作系统中，引导加载器（boot loader）的最后一条指令通常是一个跳转指令，用于跳转到内核的入口点开始执行。这个跳转指令通常是 `jmp` 或者 `call`，具体取决于引导加载器的实现。

在 JOS 的引导加载器的源代码中，你可能会看到类似于以下的代码：

```asm
jmp *%eax
```

这条指令的意思是跳转到 `%eax` 寄存器中的地址开始执行。在这个上下文中，`%eax` 寄存器中存储的就是内核的入口点地址。

至于内核的第一条指令，它位于内核映像的入口点。这个入口点是在链接内核时由链接器确定的，通常在链接脚本中指定。在 JOS 的链接脚本（通常是一个名为 `kernel.ld` 的文件）中，有一行类似于 `ENTRY(start)` 的代码，这里的 `start` 就是内核的入口点，也就是内核的第一条指令的位置。

当引导加载器加载内核到内存并跳转到内核入口点时，就会开始执行内核的第一条指令。这个过程在 JOS 的引导加载器的源代码中有具体的实现。

### bootmain

下面是 Boot Loader 中的 C 语言代码，这段代码从磁盘读取 ELF 格式的可执行文件，并将其加载到内存中，然后跳转到程序的入口点开始执行。如果在读取或加载过程中发现文件不是有效的 ELF 文件，或者其他任何错误，它将通过向特定的 I/O 端口发送特定的数据来表示错误，然后进入一个无限循环，停止执行。

```c
void
bootmain(void)
{
	struct Proghdr *ph, *eph;

    // 从磁盘读取第一页
    // ELFHDR是指向ELF头的指针，SECTSIZE*8定义了要读取的扇区数量
    // 从磁盘的0位置开始，读取SECTSIZE*8个扇区的数据到ELFHDR指向的内存位置
	readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);

    // 检查这是否是有效的ELF文件
    // ELF_MAGIC是ELF文件的魔数，用于验证文件格式
    // 如果ELFHDR指向的内存位置的e_magic字段不等于ELF_MAGIC，说明这不是一个有效的ELF文件
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad; // 如果不是有效的ELF文件，则跳转到错误处理
    }

    // 加载每个程序段（忽略ph标志）
    // ph是指向程序头的指针，e_phoff是程序头表的偏移量
    // 通过ELFHDR和e_phoff计算得到程序头表的起始位置
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
    // eph是指向程序头表末尾的指针，通过程序头数量e_phnum计算得到
    eph = ph + ELFHDR->e_phnum;
	for (; ph < eph; ph++) {
        // p_pa 是该段的加载地址（物理地址）
        // p_memsz 是段在内存中的大小
        // p_offset 是段在文件中的偏移量
        // 从磁盘读取数据并加载到指定地址
        // 从磁盘的p_offset位置开始，读取p_memsz个字节的数据到p_pa指向的内存位置
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
	}

    // 调用ELF头中的入口点
    // e_entry是程序入口点的地址
    // 注意：该函数不会返回！
    // 将控制权转交给ELF文件指定的入口点，开始执行程序
	((void (*)(void)) (ELFHDR->e_entry))();

bad:
    // 错误处理
    // 发送两个字（word）的数据到I/O端口0x8A00
    // 通过outw函数，向0x8A00端口写入0x8A00，表示出现错误
	outw(0x8A00, 0x8A00);
    // 再次发送两个字的数据到I/O端口0x8A00
    // 通过outw函数，向0x8A00端口写入0x8E00，表示出现错误
	outw(0x8A00, 0x8E00);
    // 进入无限循环，什么也不做，用于处理错误情况
    // 如果出现错误，程序将停在这里，不再继续执行
	while (1)
		/* do nothing */;
}
```

这段代码是在引导加载程序中执行的，其目的是加载 ELF（Executable and Linkable Format）格式的内核映像的头部信息。ELF 头部包含了一些重要的元数据，如程序入口点、程序头表的位置和大小等，这些信息对于后续的内核加载和执行至关重要。

在这个过程中，`readseg` 函数被用来执行实际的磁盘读取操作。它的参数分别是目标内存地址、要读取的字节数以及磁盘上的偏移量。

### readseg

```cpp
// 从内核的'offset'位置开始，读取'count'字节数据到物理地址'pa'。
// 可能会读取比请求的更多数据
void readseg(uint32_t pa, uint32_t count, uint32_t offset) {
    uint32_t end_pa;

    end_pa = pa + count; // 计算结束地址

    // 将物理地址向下舍入到扇区边界
    // SECTSIZE是扇区大小，这里通过位运算清除低位，实现向下舍入
    pa &= ~(SECTSIZE - 1);

    // 将字节偏移量转换为扇区偏移量，内核从第1个扇区开始
    // 这里假设offset是以字节为单位的，先除以SECTSIZE得到扇区数，再加1得到实际的扇区偏移
    offset = (offset / SECTSIZE) + 1;

    // 如果这种一次读取一个扇区的方式太慢，我们可以一次读取多个扇区。
    // 这样做会向内存写入比请求的更多的数据，但这并不重要——
    // 因为我们是按顺序加载的。
    while (pa < end_pa) {
        // 由于我们还没有启用分页，并且正在使用恒等段映射（见boot.S），
        // 我们可以直接使用物理地址。一旦JOS启用了MMU（内存管理单元），情况就不再是这样。
        readsect((uint8_t*) pa, offset);
        pa += SECTSIZE; // 移动到下一个扇区
        offset++;       // 扇区偏移量增加
    }
}
```

这段代码是一个名为 `readseg` 的函数，它从磁盘的特定位置（由 `offset` 参数指定）读取一定数量的数据（由 `count` 参数指定），并将这些数据加载到物理内存的特定位置（由 `pa` 参数指定）。

函数首先计算出数据读取的结束地址 `end_pa`，这是通过将开始地址 `pa` 加上要读取的字节数 `count` 来得到的。

然后，函数将开始地址 `pa` 向下舍入到最近的扇区边界。这是通过将 `pa` 与扇区大小 `SECTSIZE` 减 1 的按位取反结果进行按位与操作来实现的。这样做的目的是确保我们总是从一个完整的扇区开始读取数据。

接着，函数将字节偏移量 `offset` 转换为扇区偏移量。这是通过将 `offset` 除以扇区大小 `SECTSIZE`，然后加 1 来实现的。这样做的目的是将字节偏移量转换为扇区偏移量，因为磁盘读取操作通常是以扇区为单位进行的。

然后，函数进入一个循环，从开始地址 `pa` 读取数据，直到达到结束地址 `end_pa`。在每次循环中，函数都会调用 `readsect` 函数从磁盘的 `offset` 扇区读取一个扇区的数据，并将这些数据加载到地址 `pa` 指向的内存位置。然后，函数将 `pa` 和 `offset` 都增加 `SECTSIZE`，以便在下一次循环中读取下一个扇区的数据。

### readsect

readseg 中调用了 readsect ，下面是该函数的函数签名，它从硬盘的指定扇区读取数据。函数接受两个参数：一个指向目标缓冲区的指针`dst`，以及一个表示扇区偏移量的 32 位整数`offset`。

```c
// 从磁盘读取一个扇区的数据到指定内存地址
// 参数:
//   - dst: 目标内存地址，用于存储读取的数据
//   - sec: 扇区号，指定要读取的扇区
void readsect(void*, uint32_t);
```

下面是 readsect 的具体的实现细节，函数首先调用`waitdisk`函数等待磁盘准备好。然后，通过向 I/O 端口写入数据，设置要读取的扇区和数量。这里，端口`0x1F2`用于设置读取扇区的数量，端口`0x1F3`到`0x1F6`用于设置扇区偏移，端口`0x1F7`用于发送读取扇区的命令。

设置完毕后，函数再次调用`waitdisk`等待磁盘准备好。最后，函数使用`insl`指令从端口`0x1F0`读取一个扇区的数据到`dst`指向的地址。这里，`SECTSIZE/4`是因为`insl`一次读取 4 个字节，所以要读取`SECTSIZE/4`次。

```c
void readsect(void *dst, uint32_t offset) {
    // 等待磁盘准备好
    waitdisk();

    // 设置要读取的扇区
    outb(0x1F2, 1);      // 设置读取扇区的数量为1
    outb(0x1F3, offset);         // 设置扇区偏移的低8位
    outb(0x1F4, offset >> 8);    // 设置扇区偏移的第9到16位
    outb(0x1F5, offset >> 16);   // 设置扇区偏移的第17到24位
    outb(0x1F6, (offset >> 24) | 0xE0); // 设置扇区偏移的高8位，并设置驱动器号和其他必要的位
    outb(0x1F7, 0x20);   // 发送读取扇区的命令（0x20）

    // 再次等待磁盘准备好
    waitdisk();

    // 从端口0x1F0读取一个扇区的数据到dst指向的地址
    // SECTSIZE/4是因为insl一次读取4个字节，所以要读取SECTSIZE/4次
    insl(0x1F0, dst, SECTSIZE/4);
}
```

### waitdisk

下面这段代码定义了一个名为`waitdisk`的函数，它的作用是等待磁盘准备好。函数中使用了一个 while 循环，循环的条件是通过`inb`函数读取磁盘控制器的状态（端口 0x1F7），并检查状态寄存器的第 6 位（0x40）是否被设置，同时第 7 位（0x80）是否被清除。只有当这两个条件都满足时，循环才会结束，表示磁盘已经准备好。

```c
void waitdisk(void) {
    // 等待磁盘准备好
	// 使用inb函数读取磁盘控制器的状态（端口0x1F7），并等待磁盘准备好。
	// 磁盘准备好的标志是状态寄存器的第6位（0x40）被设置，同时第7位（0x80）被清除。
    while ((inb(0x1F7) & 0xC0) != 0x40)
        /* 什么也不做 */;
}
```

`inb`函数是一个输入函数，用于从指定的 I/O 端口读取一个字节的数据。在这里，它读取的是磁盘控制器的状态。

这段代码中的`0x1F7`是磁盘控制器状态寄存器的端口号，`0xC0`是一个掩码，用于检查状态寄存器的第 6 位和第 7 位，`0x40`表示磁盘已经准备好。

如果磁盘还没有准备好，那么这个 while 循环会一直执行，直到磁盘准备好为止。在循环体中没有任何操作，这是一种常见的忙等待（busy waiting）技术，也就是说，如果磁盘没有准备好，CPU 会一直空转，直到磁盘准备好为止。

### 加载每个程序段

接下来讲解 bootmain 中将 ELF 文件中加载每个程序段到内存中的细节。ELF 文件的程序头表包含了每个程序段的信息，如在文件中的偏移量、在内存中的位置和大小等。

```c
    // 加载每个程序段（忽略ph标志）
    // ph是指向程序头的指针，e_phoff是程序头表的偏移量
    // 通过ELFHDR和e_phoff计算得到程序头表的起始位置
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
    // eph是指向程序头表末尾的指针，通过程序头数量e_phnum计算得到
    eph = ph + ELFHDR->e_phnum;
	for (; ph < eph; ph++) {
        // p_pa 是该段的加载地址（物理地址）
        // p_memsz 是段在内存中的大小
        // p_offset 是段在文件中的偏移量
        // 从磁盘读取数据并加载到指定地址
        // 从磁盘的p_offset位置开始，读取p_memsz个字节的数据到p_pa指向的内存位置
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
	}
```

首先，通过`ELFHDR->e_phoff`获取程序头表在 ELF 文件中的偏移量，然后将其与`ELFHDR`相加，得到程序头表在内存中的位置，将其赋值给`ph`。

然后，通过`ELFHDR->e_phnum`获取程序头表中的条目数量，将其与`ph`相加，得到程序头表在内存中的结束位置，将其赋值给`eph`。

接下来，使用一个 for 循环遍历程序头表中的每个条目。对于每个条目，它从磁盘的`ph->p_offset`位置开始，读取`ph->p_memsz`个字节的数据，然后将这些数据加载到`ph->p_pa`指向的内存位置。这个过程是通过调用`readseg`函数完成的。

这样，每个程序段都被正确地加载到了内存中的指定位置。

### Boot Loader 结束

下面的代码将控制权最终交给了内核，随后就是执行的内核代码了。

```c
    // 调用ELF头中的入口点
    // e_entry是程序入口点的地址
    // 注意：该函数不会返回！
    // 将控制权转交给ELF文件指定的入口点，开始执行程序
	((void (*)(void)) (ELFHDR->e_entry))();
```

这段代码是在调用 ELF 文件的入口点，也就是程序的起始地址。这个地址是在 ELF 文件的头部中指定的，通常是程序的主函数或者其他类似的起始点。

这里的代码`((void (*)(void)) (ELFHDR->e_entry))();`做了以下几件事情：

1. `ELFHDR->e_entry`获取了 ELF 头部中的`e_entry`字段，这个字段包含了程序入口点的地址。

2. `(void (*)(void))`是一个函数指针的类型转换，它将`e_entry`字段的值转换为一个没有参数也没有返回值的函数指针。

3. `()`是函数调用操作符，它调用了转换后的函数指针，也就是说，它跳转到了程序的入口点并开始执行。

这段代码之所以会将控制权交给内核，是因为在这个过程中，CPU 的指令指针被设置为了内核的入口点，从而开始执行内核的代码。这个过程通常发生在系统引导的过程中，当引导加载程序（bootloader）加载并启动内核时。

### 总结

至此，Boot Loader 的任务完成了，接下来控制权交给内核，后续会讲解跳转到内核后的内容。
