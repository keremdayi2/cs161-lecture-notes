# Feb 14
### Implementing `sleep(ms)` inefficiently:
In chickadee:
* `#define HZ 100`. Number of ticks per second. 
* `std::atomic<unsigned long> ticks;` # of timer interrupts so far on CPU 0
*  Define a wakeup time for the process, `wakeup_time = ticks + sleep_ticks`
* Then, while `long(wakeup_time - ticks) > 0`, yield.
### The challenges of Blocking:
* Kernel tasks often need to block until an event occurs
    * Ex: a certain amount of time has elapsed
    * Ex: keyboard data arrives
    * Ex: A (non-spinlock) lock is released
* Efficient mechanism for handling these blockign events
### Wait Queues to the Rescue
* Have a queue for each type of blocking event
    * Threads that wait for the same event will sleep on the same queue
    * When the event in question is noticed by a running thread, this wakes the sleepers in the relative queue.

*True blocking is now possible*!!!!: However, correctly implementing a wait queue is tricky.
### Formalizing the Blocking Problem
* Let $x$ be a subset of program state
    * A sleeping thread blocks until $f(x)$ is true.
    * Running thread modifies $x$ to  make $f(x)$ true.
### Sleep/wakeup attempt 1
1. Sleeper thread computes the predicate false (but hasn't slept yet).
2. The wakeup thread modifies $x$ so that predicate $f(x)$ is true. But cannot notify anyone!! (no-one is sleeping)
3. The other cpu sleeper thread goes to sleep. BLOCKS FOREVER!!!
### Sleep/wakeup attempt 2
* sleeper thread calls`x_lock.lock()`
* checks false, goes to sleep
* Tries to `x_lock.lock()` and blocks.
* Sleeper thread sleeps forever, because the wakeup thread is sleeping!!.
### Sleep/wakeup attempt 3

### Sleep/wakeup solution 1
* Sleeper thread checks the predicate, sees false. Decides to block. 
* Calls threading api `cond_wait(&x_cond, &x_lock)`. This atomically releases lock and blocks thread; when it returns, lock held by caller.
* Wakeup thread locks `x_lock.lock()`. Does stuff so that $f(x)$ is true. `cond_signal(&x_cond)` wakes up sleeping thread.
### Case study: OS/161
* Implements a wait channel (queue)
* `wcan_sleep` puts thread on sleep queue and releases lock
### What do other OSes Do?
* FreeBSD and xv6 implement blocking in a similar way to OS/161
    * Scheduler must be aware of wait queue abstractions
* Linux and Chickadee employ a different wait queue design.
    * Goal: Loosen the binding between the scheduler and the condition/wait abstraction.
    * We do not need couple shceduler design and wait queues
    * Maintain wait queues outside of scheduler 
* Chickadee style wait queues enable more flexibility
### Chickadee Wait Queues
**Wakeup Thread**:
* wakeup function will acquire the lock (via guard)
* Then will wake all in wait queue
* release lock when the function ends

**Sleeper Thread**:
* acquire `x_lock.lock()`
* `w.prepare(waitq);` prepare to block if necessary
* Check predicate
* If true, break out of the loop, `w.clear()` no longer on wait queue. and unlock `x_lock`.

