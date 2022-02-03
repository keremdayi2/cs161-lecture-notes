# Notes: Monday January 31st: Multitasking and Preemption
A user level execution context consists of:
* VA space (pointer to by `%cr3`)
* The next execution to execute (pointed to by `%rip`)
* A stack (pointed to by `%rsp`)
* Other register state

Multitasking allows us to run multiple context to run atop a single processor.
* Divide time into slices
* At the beginning of a slice, pick a task (ie, a context) to run
* Run the task until, 
    1. Volutarily yields the CPU
    2. It is interrupted (eg. timer interrupt) and must involuntarily release the CPU

Kernel runs in response to external stimuli:
* **Asynchronous interrupts**: timer interrupt, nic (network interface card) interrupt
* **Synchronous exceptions**: (eg. dividing by zero, dereferncing a null pointer, attempting to write a read only page)
* **Synchronous traps**: system calls via syscall, debugging traps via int3

During boot, the kernel must configure the processor to invoke the appropriate kernel code when exceptions, traps, and interrupts occur.

## **Handling a system call**
x86-64 descriptor tables:
* An OS assigns to `%cr3` to configure how a core handles virtual memory
* An OS sets up descriptor tables to configure other processor behavior
    - each table is located somewhere in kernel memory (only configurable by kernel mode)
    - each table contains multiple entries called descriptors

Each core of a multi-core processor has:
* A global decsriptor table register (`%gdtr`)
* A global descriptor table (GDT) that is:
    * Located in kernel memory

x86-64: The `%fs` and `%gs` Segment Registers:
* Accessible by user and kernel code
* Visible part: segment selector that can be directly read or written by normal register instructions like `mov` and `push` (they manipulate the visible part)
* Hidden part: Contains the base address that's used in base+offset address calculations like `mov %fs:(28), %rax`

The visible and hidden parts of `%fs` and `%gs` must be properly saved and restored during context switches!

x86-64: Leveraging `%gs` for kernel fun
* An OS needs to store a per-CPU data structure that is only accessible by the kernel
* Each core has its own `%KERNEL_GS_BASE` register
    * 
