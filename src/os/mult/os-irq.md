时钟中断是由计算机的硬件时钟（通常是一个计数器）产生的。这个硬件时钟会在固定的时间间隔（通常是几毫秒）向 CPU 发送一个中断信号。这个时间间隔被称为时钟滴答（tick）。

当 CPU 接收到这个中断信号时，它会暂停当前正在执行的任务，保存当前任务的状态，然后跳转到一个预先定义的中断处理程序（Interrupt Service Routine，ISR）去处理这个中断。在这个中断处理程序中，操作系统可以做一些定期需要做的事情，比如更新系统时间，调度其他任务运行等。

### 时钟中断有什么用？

目前 JOS 内核不支持来自时钟硬件的外部硬件中断，如果用户程序是一个死循环的话，那么这个程序会永远不归还 CPU 进而使得整个系统陷入停顿。所以为了让内核能够抢占正在运行的环境，需要强制从它那里重新获得 CPU 的控制权，接下来需要扩展 JOS 内核以支持来自时钟硬件的外部硬件中断。

时钟中断是一种硬件中断，由计算机的时钟产生。它通常用于操作系统的调度器，以实现抢占式多任务。当时钟中断发生时，当前正在执行的进程会被暂停，操作系统的调度器会选择另一个进程来执行。这样，即使一个进程进入了无限循环，也不会导致整个系统停止响应，因为时钟中断会强制切换到其他进程。此外，时钟中断也常用于实现定时器功能，例如在一定时间后唤醒一个进程，或者在特定的时间点执行某个任务。

IRQ，全称为中断请求（Interrupt Request），是一种硬件设备向处理器发送中断信号的机制。当硬件设备需要处理器的注意时，它会发送一个 IRQ 信号。处理器会响应这个中断请求，暂停当前的任务，保存当前的状态，然后执行与该 IRQ 关联的中断处理程序。当中断处理程序完成后，处理器会恢复被中断的任务。IRQ 是实现硬件设备与处理器之间异步通信的重要机制。

IRQ（中断请求）和时钟中断之间的关系是，时钟中断是一种特殊类型的 IRQ。在计算机系统中，硬件设备通过发送 IRQ 信号来通知处理器需要处理某些事件。时钟中断是由计算机的时钟系统产生的 IRQ，用于告知处理器已经过去了一定的时间。

当时钟中断发生时，处理器会暂停当前正在执行的任务，保存当前的状态，然后执行与时钟中断关联的中断处理程序。这个中断处理程序通常会做一些与时间相关的处理，比如更新系统时间，或者检查是否有需要在此时执行的定时任务。

### IRQ 初始化

IRQ 有 16 个，编号为 0 到 15。IRQ 号到 IDT 条目的映射不是固定的。`pic_init` 在 `picirq.c` 中将 IRQs 0-15 映射到 IDT 条目 IRQ_OFFSET 到 IRQ_OFFSET+15 。

在 `inc/trap.h` 文件中，`IRQ_OFFSET` 被定义为 32。这是因为在 x86 架构中，前 32 个中断向量（0-31）被保留用于处理器异常。因此，硬件中断（IRQs）从 32 开始编号，以避免与处理器异常冲突。

在 `kern/trap.c` 文件中，我们可以看到如何初始化 IDT 条目以处理 IRQs。例如，考虑以下代码行：

```cpp
SETGATE(idt[T_SYSCALL], 0, GD_KT, th_syscall, 3);
```

这行代码设置了系统调用（`T_SYSCALL`）的中断描述符表（IDT）条目。`SETGATE` 宏用于设置 IDT 条目的各个字段。

- `idt[T_SYSCALL]` 是要设置的 IDT 条目。
- 第二个参数 `0` 表示这不是一个陷阱门（trap gate）。陷阱门在处理中断后不会自动关闭中断，而中断门会。这里，我们希望在处理系统调用时自动关闭中断，所以这不是一个陷阱门。
- `GD_KT` 是中断处理程序的代码段选择子。这告诉处理器中断处理程序在哪个代码段中。
- `th_syscall` 是中断处理程序的地址。当发生系统调用中断时，处理器会跳转到这个地址开始执行代码。
- 最后一个参数 `3` 是描述符特权级（Descriptor Privilege Level，DPL）。这决定了哪些特权级的代码可以调用这个中断。在这里，我们设置为 3，这意味着用户模式的代码可以发起系统调用。

同样的方法也用于设置 IRQ 的 IDT 条目。例如，时钟中断（IRQ 0）的 IDT 条目是 `idt[IRQ_OFFSET + 0]` 或 `idt[32]`。当时钟中断发生时，处理器会查找 `idt[32]` 条目，然后跳转到该条目指向的中断处理程序。

在 JOS 中，外部设备中断在内核模式下总是被禁用，这是一个关键的简化，这与 xv6 Unix 操作系统的行为相同。在用户空间中，外部中断是启用的。这种中断的启用和禁用是由 `%eflags` 寄存器的 `FL_IF` 标志位控制的。

当 `FL_IF` 标志位被设置时，外部中断就会被启用。虽然有几种方法可以修改这个标志位，但是由于我们的简化，我们只通过保存和恢复 `%eflags` 寄存器来在进入和离开用户模式时处理它。

例如，当我们从内核模式切换到用户模式时，我们会保存当前的 `%eflags` 寄存器的值，然后设置 `FL_IF` 标志位，以启用外部中断。当我们从用户模式切换回内核模式时，我们会恢复保存的 `%eflags` 寄存器的值，从而禁用外部中断。

这种处理方式简化了中断处理的逻辑，因为我们知道在内核模式下，我们不需要处理外部设备中断。这使得我们可以专注于处理内核的任务，而不需要担心被外部设备中断打断。在用户模式下，我们允许外部设备中断，因为用户程序可能需要响应外部设备的事件。

### 为什么硬件中断不会提供错误码？

这句话的意思是，当处理器响应础件中断时，它不会将错误代码（error code）推送到堆栈。错误代码是一种特殊的值，用于提供关于异常或中断的更多信息。例如，当发生页错误（Page Fault）时，处理器会将一个错误代码推送到堆栈，以指示导致页错误的具体原因。

然而，对于硬件中断，处理器不会这样做。这是因为硬件中断通常不是由错误引起的，而是由外部设备发出的信号，表示需要处理器的注意。例如，当硬盘完成了数据传输，或者网络卡接收到了一个数据包时，它们会发出一个硬件中断，让处理器知道现在可以处理这些数据了。

### 时钟中断是如何产生的？

在多处理器系统中，每个处理器都有一个本地 APIC（Advanced Programmable Interrupt Controller，高级可编程中断控制器）。`lapic_init()`函数用于初始化这个本地 APIC。本地 APIC 可以接收来自系统其他部分的中断，并将其传递给相应的处理器。

在早期的单处理器系统中，PIC（Programmable Interrupt Controller，可编程中断控制器）是唯一的中断控制器。`pic_init()`函数用于初始化 PIC。PIC 可以接收来自系统其他部分的中断，并将其传递给 CPU。

这两个函数的主要目的是设置时钟中断控制器以产生中断。时钟中断是由硬件时钟（通常是一个计数器）产生的。这个硬件时钟会在固定的时间间隔（通常是几毫秒）向 CPU 发送一个中断信号。这个时间间隔被称为时钟滴答（tick）。

当 CPU 接收到这个中断信号时，它会暂停当前正在执行的任务，保存当前任务的状态，然后跳转到一个预先定义的中断处理程序（Interrupt Service Routine，ISR）去处理这个中断。在这个中断处理程序中，操作系统可以做一些定期需要做的事情，比如更新系统时间，调度其他任务运行等。

### 时钟中断是如何处理的？

当中断号 `tf->tf_trapno` 等于 `IRQ_OFFSET + IRQ_TIMER` 时，表示发生了时钟中断。

```c
    // ...
	if (tf->tf_trapno == IRQ_OFFSET + IRQ_TIMER) {
		lapic_eoi();
		sched_yield();
		return;
	}
    // ...
```

首先，调用 `lapic_eoi()` 函数来确认这个中断。在 x86 架构中，当中断处理程序完成时，需要向本地 APIC 发送一个 EOI（End of Interrupt）信号，表示中断已经被处理完毕。如果不发送 EOI，那么 APIC 将不会发送更多的中断。

然后，调用 `sched_yield()` 函数来进行进程调度。这是因为时钟中断通常用于实现抢占式多任务，即当一个进程运行一段时间后（由时钟中断决定），操作系统会强制该进程让出 CPU，切换到其他进程运行。`sched_yield()` 函数的作用就是选择一个新的进程来运行。

最后，使用 `return` 语句结束这个中断处理程序。

### 总结

通过上面的设置使得 JOS 支持时钟中断，进而使得 OS 可以进行调用切换不同的进程，随后可以在此基础上实现 IPC 机制。下篇文章讲解如何 JOS 是如何实现 IPC 的。
