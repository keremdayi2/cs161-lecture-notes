# How Syscall Entry Works
When we enter `syscall_entry`, we initially `swapgs`. What this does is that is switches the `%gs` base pointer to point to `%KERNEL_GS_BASE` which points to the per-cpu data structure, `cpustate`. Then, we do the following 
1. Save the user `%rsp` in the cpustate struct at loc `%gs:(16)` which is a member variable called `syscall_scratch_`
2. Set `%rsp` to the pointer pointing to the per-process data structure of the current running process. `movq %gs:(8), %rsp`.
3. Add proc stack size so that `%rsp` now points to the top of the kernel task stack, which is allocated above `struct proc`: `addq $PROCSTACK_SIZE, %rsp`.

When we complete these operations, `%rsp` is now pointing to the kernel task stack for that process. Then, we push the 5 registers that the hardware would automatically push onto the stack if an exception occurred: `%ss`, User `%rsp`, `%rflags`, `%cs`, `%rcx`
* `%rcx` stores the user `%rip` that points to the instruction right after the system call.

Then, all the user registers are pushed onto the stack (except the ones used by syscall). Then, `%rdi` is set to `%gs:(8)` which points to the per-process struct. `%rsi` stores `%rsp`.

After `syscall(regstate* regs)` returns, we have to first check if the process is runnable (it must be). This is done by `movq %gs:(8), %rdi` and `cmpl $PROC_RUNNABLE, 24(%rdi)` where `24(%rdi)` is `pstate_`. Finally, we restore the stack by moving the stack pointer up. Then, we disable interrupts and call `swapgs`, which restores the old value of `%gs` to stores its old user-level value. 
* Why do we disable interrupts before calling `swapgs`: otherwise, if an interrupt occurred after `swapgs`, the exception entry would think we are in kernel mode (so it wouldn't `swapgs`) but this would cause chaos as the program would start executing on the user stack!!!!.

We finally return with `iretq` which enables interrupts.

# How does Exceptions Work
IST: Interrupt Stack Table