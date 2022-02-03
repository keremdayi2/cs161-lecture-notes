# **CS 161 Lecture Notes**
## **Monday, January 24th**
## **Processor design**
Instruction set architectures are what the compiler compiles high level code (c++, go, rust) into. The processor executes ISAs, it cannot execute high level c++ code. There are 3 different types of instructions:
* Arithmetic/bitwise logic
* Data movement between registers and/or memory
* Control flow -> conditional & unconditional jumps (eg. function calls, loops)
## **RISC vs CISC: ISA Wars**
RISC (Reduced Instruction Set Compiler) and CISC (Complex Instruction Set Compiler) are the two types of ISAs.
### **CISC: Complex Instruction Set Compiler**
In CISC, single instructions do many things. Instructions also vary in size. 
> This makes life easier for compiler writers as they can use single instructions that can do very complicated tasks instead of trying to combine multiple instructions to accomplish a small task.

However, it is complex to create hardware for CISC, as executing a single instructions is much more complicated than RISC. x86_64 is the classic ISA for CISC. For instance, consider the instructions

`mov %rax, %rbx` and `mov 16(%rsp), %rbx`

Though the first instruction is simple, the second instruction needs to do a lot of things. It has to add 16 to the value of `%rsp` and then do a memory lookup. After that, it must perform the `mov` operation. Consider the even worse instruction `movsb`. It has to do a condition test, memory read, memory write, and two register increments and decrements!

### **RISC: Reduced Instruction Set Compiler**
In RISC, each instruction does a simple thing. Instructions are fixed size, and uses the load/store architectures (except loads and stores, instructions only manipulate registers). 
> This makes is easier to design fast, power-efficient hardware

Two examples of RISC are MIPS and ARM.

### **x86_64 Registers**
* There are 16 general purpose registers in x86-64. 
* `%rbp`: break pointer, `%rsp`: stack pointer, others `%rax, %rbx`, instruction pointer: `%rip`
* There are also special purpose registers: `%cr3` for page-table pointer, `xmm0-xmm15` for floating point arithmetic

### **x86-64 System V Calling Convention**

When the arguments are integers and pointers, return value is pointer/integer:

1. Caller stores first 6 arguments `%rdi, %rsi, %rdx, %rcx, %r8, %r9`.
2. Remaining arguments are passed via stack
3. Return value is in `%rax`

Caller Saved Registers: `%rdi, %rsi, %rdx, %rcx, %r8, %r9, %r10, %r11, %rax`\
Callee Saved Registers: `%rbp, %rbx, %r12, %r13, %r14, %r15`

Code at the start of a function ensures these conventions. Ex:
```
pushq %rbp // save rbp (callee saved)
movq %rsp %rbp // set break point

pushq %rbx // callee saved
pushq %r12 // callee saved

subq $0x10 %rsp // allocate space for local vars.
```
### **Pipelining a Processor**
Instructions in a processor execute in 5 steps:\
1. Fetch: pull the instruction from RAM into the processor
2. Decode: Determine the type of the instructions and extract the operands
3. Execute: Perform any arithmetic operation
4. Memory: If necessary, read/write memory
5. Writeback: update the register with the result

### **Processors from the view of a Master Programmer**
### **The Problem With Branches**

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
