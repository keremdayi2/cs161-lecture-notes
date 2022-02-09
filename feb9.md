# February 7 Lecture Notes
* Your `fork()` implementation is probably a very long God function (which is ok in this scenario)
* In OS code, `goto` statements are a common way to unwind from a deeply-nested failure

### Today: Outline
* Nested interrupts and exceptions
* Locking: synchronizing kernel code
* Locking: synchronizing user code
* nested exceptions and interrupts

## Kernel Code Should (alsmost never) cause exceptions
* In real OSes, exceptions can nest at most two deep
    * E1: user mode divide by zero or whatever
    * E2: A page fault exception triggered by the kernel's handler for exception 1
## A potential thing: nested interrupt handlers
* An OS can allow interrupts to nest
    * For example, an interrupt handler could reenable interrupts as soon as old register state has been saved on the stack
* Advantage: Nesting interrupts can improve system responsiveness
* Disadvantage: Writing correct kernel code (and debugging that code) becomes more difficult!
    * If an interrupt can interrupt itself, then handler code must be reentrant
    * Stack overflows become more likely
    * Doing stack backtraces become harder.
## Linux's solution to nested interrupts
* Top-half handlers
    * Run with interrupts disabled
    * Are fast (i.e. run small amounts of code)
    * Can mark a kernel data structure to indicate that a softirq needs to run later
* Bottom half softirqs
    * Run with interrupts enabled
    * Do stuff that takes a medium amount of time, but does not require sleeping (acquiring a non-spinlock lock, performing io, allocating large amounts of memory)
    * Pending softirqs are checked
        * When returning from a top-half handler
        * By per-cpu `ksoftirq` kernel threads
* Bottom half work-queue kernel threads
    * Run with interrupts enabled
    * Can do computations that are slow-running of might sleep
## Chickadee's proc::exception()
* Timer interrupt: Suspend the currenly running proces
```
    regs_ = regs;
    yield_noreturn();
    break; // won't be reached
```
* **Page Fault**: Mark the faulting process as broken, because stock Chickadee doesn't swap pages between RAM and disk
```
    pstate_ = proc::ps_broken;
    yield(); // Won't run again because cpustate will notice the state
```
* **Keyboard interrupt**: Read a character then immediately return to the interrupted execution context
```
    keyboardstate::get().handle_interrupt();
    break; Kernel goes to k-exception.S::restore_and_iret
``` 
## Locking: Synchronizing Kernel Code
**Sacrificing Parallelism to Ensure Correctness**: 
* OSes were originally designed for single-processor machines
    * Disabling interrupts was sufficient for kernel to prevent race conditions
* Spinlock acquires just save the old interrupt state and disable interrupts-- there's no spinning!
* Spinlock releases just restore the old interrupt state
* When Linux first added multicore support in 1996, Linux used a Big Kernel Lock that was acquired at the start of any context switch into the kernel
    * Advantage: simple and still allowed user-level code to be parallel
    * Disadvantage: only one core can execute in the kernel at a time
## Case Study: Locks in the xv6 Teaching OS
* Locks for many different variables
* Impact when two distinct CPUs execute `fork()`:
    * The parallel `fork()`s may serialize
    * However, paralel forks will be able to modify pagetables simultaneously
## A partial list of Chickadee locks
* `ptable_lock`: Protects the process table
* `page_lock`: Protects memory allocation metadata
* Per-CPU `runq_lock` protects a cpu's list of runnable processes
* `cursor_lock`: console input

## Designing a locking strategy
* A lock is used to protect a collection of events that must be atomic
    * Usually, a single function will acquire and release a lock
* Atomicity reduces the difficulty of defining invariants
    * Invariants are properties that must be true before and after an operation on a data structure
    * Invariants may only be false in the middle of an atomic update
* Locks are typically needed when an invariant requires multiple memory locations to be read and written as a single atomic action
## Case Study: getppid()
* The `fork()` system call allows a parent process to create a child process
* The `getppid()` system call allows a child to get the pid of its parent
* In Unix-derived systems, a special undying `init` process is the first process created.
    * `init` has `getpid() ==1` and `getppid() == 1`
    * When a parent dies, its children are reparented to `init`
## A possible getppid() implementation
* We could add a `ppid_` field to `struct proc`:
    * Initialize it during `fork()`
    * Set it to `1` if the parent `exit()`s before the child
* Are we done?
    * Initializing `ppid_` is fast
    * The parent process can directly access its own `id_` and assign it to the child's `ppid_`
* The parent, however, the does not keep track

**Problem 1: Scalability**

**Problem 2: Synchronization: access to ppid_**: parent calls `exit()` and child calls `getppid()`
## A Different implementation of getppid()
* We could protect `ppid_` with a per-process `ppid_lock_` instead of the global `ptable_lock`.
* Bottlenecks: why is my program slow? too few CPUs? syncronization? 

## Synchronizing User Code
* So far, kernel code synchronizes with itself
* However, user-level code may want to synchronize itself!
    * Two threads in a multithreaded process need a producer/consumer queue
    * With user-level threads, there is no true concurrency
    * So, `lock()` just checks a user-level flag indicating whether the lock is held
* What about kernel visible threads?
    * Solution 1: `lock()` could atomically spin on a lock that lives in low-canonical memory
    * Solution 2: `lock()` and `unlock()` be system calls
        * `sys_lock()` implementation would atomically test-and-sets. If this fails, put on some kernel level lock queue and block.
* Other solutions: futexes
