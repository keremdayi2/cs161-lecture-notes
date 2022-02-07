# CS 161 February 7 Lecture Notes
### Outline
* Overview of concurrency
* kernel threads vs user threads
* the lifecycle of a proces
* syncing between kernel tasks

## Concurrency on a single core machine
* Quasi-concurrency: different applications share a single pipeline
* Context switches happen during interrupts, exceptions, and traps.

## Concurrency on a multicore machine
* True concurrency: different pipelines are simultaneously executing independent instruction streams
* Each pipeline might be executing instructions from different address spaces
    * OR: pipelines might be executing instructions from the same address space but different thread contexts (different values of IP and other regs.)
## Kernel Visible Threads
Many OSes allow a single address space to contain multiple threads of execution
* Each thread has a separate visible stack
* OS associated each thread with
    * an address space (shared with other threads)
    * register state, including an `%rsp` (unique per thread)
    * a kernel stack and `struct proc`
    * one kernel level stack for each user level thread in the kernel text segment
* A multi-threaded address space can be simultaneously active on multiple cores
    * Ex: a web server that spawns a new thread for each connection
    * Ex: A database that uses multiple threads to wait for parallel IO requests to complete
## Making Kernel-visible Threads on Linux: clone()
* `fork()` creates a new child process from a parent process
    * The child has a copy of the memory space of parent
* `clone()` creates a new thread that might only share *some* state with the original thread
`int clone(int (*fn)(void\*), void \* child_stack, int flags, void \*arg)
    * Ex for `child_stack`: A malloc'd region in the calling process
    * `CLONE_VM`: should the new thread use the same address space?
    * `CLONE_PARENT`: Should the new thread's parent be the parent or the parent's parent
## Linux: Processes and Kernel-visible threads
* A process is a set of resources like
    * an address space
    * A PID, UID, GID
* A kernel-visible thread is an OS-schedulable entity
    * A user-mode stack and a kernel stack
    * A set of register values
    * Scheduler metadata (eg. priority)
* A newly-created process has a single thread that executes `main()`
## Pure User-Level Threads: No Kernel Assistance
* User-level code allocated multiple stakcs in low canonical memory
    * However, the address space only has one kernel-level stacks
* User-level assembly defines `user_level_thread_switch()`
    * Pushes callee-saved registers onto the stack
    * Stores current thread's `%rip`-to-resume-at in a per-thread structure
    * Jumps to the `%rip` for the new thread
* A thread might invoke `user_level_thread_switch()` when:
    * An IO call would block: `if(read(fd, buf, 128)) == EWOULDBLOCK) {...}`
    * The thread yields
* What if you want preemption?
    * Thread library could do stuff like this:
        ```
        signal(SIGALRM, force_thread_switch);
        alarm(1); // will call force_thread_switch in 1 sec
        ```
    * Linux: `setcontext(), getcontext(), makecontext(), swapcontext()` automates some of this
### A single kernel-visible thread with multiple user-level threads
* At any given moment, one of three things is true:
    * No user-level thread is executing, and address space's single kernel task isn't running
    * Exactly one user-level thread is executing
    * The address space's kernel task is running
* There can't be truly concurrent execution of the user-level threads, even on a multicore CPU

## Hybrid Threading
* A single application can create multiple kernel threads, and place several user-level threads atop each kernel thread
* Example: goroutines in a single Go program.
    * `GOMAXPROCS` environment variable sets the number of kernel threads to use for a single Go program
    * Calls to the Go runtime (e.g. to perform IO) allow the goroutine scheduler to run
    * Each goroutine gets a 2KB user-mode stack at first
    * Each function preamble (code at the beginning of a pure user level goroutine) checks whether there's enough stack space to execute the function. If not, runtime doubles the size of the stack, copies old stack into new space, updates stack pointer.
## The Lifecycle of a process
* Running, Runnable, Blocked, Finished
## Synchronizing between kernel tasks
* A bad thing we should avoid
    * Suppose that, on a single core machine, a kernel task is executing with interrupts enabled
    * Deadlock: when we acquire a lock and an interrupt occurs, another kernel task can try to acquire that lock and deadlock!!!
    * where does kernel code enable/disable interrupts?
* `adjust_this_cpu_spinlock_depth` is used to keep track of the number of spinlocks a process has 
* per CPU stacks for interrupts:
    * Unlike chickadee, Linux doesn't immediately move state to a kernel stack!
    * Instead, initial interrupt handling totally occurs using the per-CPU interrupt stack
* Linux has two basic types of interrupt handlers
    * A top-half handler executes quickly, with interrupts disabled, doing initial interrupt handling and then reenabling interrupts