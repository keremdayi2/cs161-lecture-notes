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

