## Overview and global variables

For every experiment in eg. `nachos/exp1` there is a `main.cc` which is the file that is run when you run `./nachos` from ternimal. In `main()`, the `Initialize()` which is found in `threads/system.cc` is called, which initalised ** SIX ** global variables, which can be accessed anywhere:
* `currentThread` - which is a pointer to the current "running" thread
* `threadToBeDestroyed` - which will be set when a thread is finished (thread calls `Finish()`)
* `scheduler` - a `Scheduler` object which is in charge of maintaining the ready queue
* `interrupt` - a `Interrupt` object which has a list of `PendingInterrupt` objects
* `stats` - which stores the global ticks in `stats->totalTicks`
* `timer` - a Timer object which which uses the interrupt to schedule new interrupts 

## Scheduler
Located in `threads/scheduler.cc`. Contains all the logic in controlling the threads. Current implementation is **FCFS**
<br/>Attributes and methods:
* `readyList` - a `List` of `Thread *`, all of which are in the `READY` state
* `ReadyToRun(Thread * thread)` - add `thread` to `readyList` and sets the state of `thread` as `READY`
* `FindNextToRun()` - returns pointer to next thread (or NULL if list is empty). The thread is removed from `readyList`. This is where the scheduling algorithm would be, but in this case it just take the head of the list
* `Run(Thread *nextThread)` 
    * save state of `currentThread`
    * set state of `nextThread` to `RUNNING`
    * if `currentThread` gave up CPU because it called `Finish()`, the it would have set `threadToBeDestroyed` to be itself. `Run()` will delete this thread and deallocate memory

## Threads
Located in `threads/thread.cc`. the pointer to the thread is treated as the thread control block(TCB) and is the one being passed around and queued. 
<br/> Attributes and Methods:
* `Thread(char *name)` - constructor, literally just to give it a name. Holds no other logic
* `Fork(voidFnPointer *function, int args, bool joinP)` - gives the thread a function to run. calls `scheduler->ReadyToRun(this)` to add itself to the ready queue. The `scheduler` is the one that sets its state to `RUNNING`
    * `joinP` is a boolean flag which indicates that there is another thread which has/will call `Join()` on this thread, and indicates that when this thread is finished, then another thread will continue
* `Yield()` - **The most important** method in all of nachOS because it causes a context switch
    * calls `nextThread = scheduler->FindNextToRun()`. If it returns smth that is not NULL, then context switch
    * calls `scheduler->ReadyToRun(this)` to add itself to the back of the ready queue
    * calls `scheduler->Run(nextThread)` to start running the next thread. All state changes are handled by `scheduler` 
* `Sleep()` - set its own status to `BLOCKED` (waiting), and calls `nextThread = scheduler->FindNextToRun()`. calls `scheduler->Run(nextThread)` if not NULL. If NULL then the CPU will idle
* `Finish()` - is automatically called with the thread's forked function is complete. Takes up one cycle of Ticks (TODO). Sets global variable `threadToBeDestroyed` as itself and calls `Sleep()` to trigger context switch, in which `scheduler->Run()` will destroy this thread. If `joinP` is `TRUE`, then it'll also let the thread waiting for it to continue. **In exp2, this is modified so that it resets the pending interrupt in `interrupt`**
* `Join(thread *forked)` - the thread will wait for `forked` to call `Finish()` before it reenters the ready queue

## Timer
Located in `machine/Timer.cc`. Not the actual timer which counts down. Only job is to come up with numbers (fixed or random) for the next interrupt, the actual countdown is within the PendingInterrupt objects initialised by the interrupt object. 
<br/> Attributes and Methods:
* `Timer(voidFnPointer handler, int arg, bool doRandom)` - constructor, called in `Initialize()`, calls `interrupt->Schedule(TimerHandler, this, TimeOfNextInterrupt(), TimerInt)`, which basically adds the first `PendingInterrupt` object. The `handler` is a by default `TimerInterruptHandler()` defined in `threads/system.cc`
* TimerExpired() - starts the next timer by calling `interrupt->Schedule(TimerHandler, this, TimeOfNextInterrupt(), TimerINt)`, and then calls `handler` function with given `args`
* `TimeOfNextInterrupt()` - returns a number, either random or fixed based on `doRandom` boolean flag. If fixed (and `handler` is `TimerInterruptHandler`), then the scheduler follows the round-robin schedulling algorithm.  This is because `TimerInterruptHandler` calls `currentThread->Yield()` meaning that the interrupts are pre-emptive.
* TimerHandler(int *dummy) - a handler called in `CheckIfDue()` that calls `TimerExpired()` which then calls the actual `TimerInterruptHandler()` 

## TimerInterruptHandler()
Defined in `threads/system.cc`. Calls `interrupt->YieldOnReturn()` which sets the YieldOnReturn flag to be `TRUE` (HONESTLY NO FKING CLUE WHAT THIS IS FOR)

## Interrupt
Defined in `machine/interrupt.cc`. Used in Timer class. 
<br/> Attributes and Methods:
* `PendingInterrupts[] pending` - a `List` of `PendingInterrupts`, sorted in increasing order, such that the system only needs to check the start of the list to see if an interrupt is needed. 
* `Interrupt()` - constructor to initialise empty `pending` list
* 'Schedule(handler, arg, fromNow, type)' - also ** important**. Initialises a `PendingInterrupt` object with `PendingInterrupt(handler, arg, when, type)`. where `when =  stats->totalTicks + fromNow `. This object is inserted into `pending` based on `when`
* `OneTick()` - increments the global variable `stats->totalTicks` and calls `CheckIfDue()`. If `YieldOnReturn` boolean flag is `TRUE`, then call `currentThread->Yield()` at the end.
* 'CheckIfDue(boolean advanceClock)' - remove first `PendingInterrupt` and check it's when attribute. If not due, add it back to `pending` and return `FALSE`. If it is due, call the `TimerHandler()` in `Timer`, which in turn calls `TimerExpired()` and the actual `TimerInterruptHandler()`
* `YieldOnReturn()` - set `YieldOnReturn` boolean flag to be `TRUE`
<br/><br/> Other misc methods:
* `Idle() `- to wait for the next thread to enter ready queue
* `SetLevel()` and `ChangeLevel()` - to stop interrupts and start interrupts

## Overall Interrupt Sequence 
Overall, all the steps during an interrupt takes up **one cycle on the CPU (10 Ticks in exp3). This is the reason that in exp2, the first experiment takes more ticks.** The steps are
* `interrupt->OneTick() `
    * increment `stats->totalTicks`
    * `CheckIfDue()`
        * `pendingInterrupt->TimerHandler()` (which is a method of Timer)
            * `timer->TimerExpired()`
                * `interrupt->Schedule()` to schedule the next pending interrupt
                * `TimerInterruptHandler()` defined in `system.cc`
                    * `interrupt->YieldOnReturn()` to set `YieldOnReturn` flag to `TRUE`
    * check `YieldOnReturn` flag
    *` currentThread->Yield()`
    * `YieldOnReturn = FALSE`


## Semaphores
Defined in `threads/synch.cc`. The implementation is similar to blocking in the slides, with some small implementation details that are different. 
<br/> Attributes and Methods: 
* `int value` - the semaphore value
* `thread[] queue` - the waiting queue for the semaphore
* `Semaphore(name, initialValue)` - constructor
* `P()` - short for Probeer meaning try. Check if `value==0`. If so, then the thread has to sleep and will call` currentThread->Sleep()`. If `value>0`, it means that the thread can enter the critical section, and value will be decremented to indicate what one unit of resource has been allocated. 
* `V()` - short for Verhoog meaning increment. Called when thread exits the critical section. removes next thread in `queue` and if it's not NULL, call `scheduler->ReadyToRun(nextThread)` to move it back to the ready queue to enter the critical section, increment `value` to indicate that one unit of resource has been deallocated. 

## Lock 
Defined in `threads/synch.cc`. Wraps a Semaphore, but with an added attribute `lock_owner`.
<br/> Attributes and Methods:
* `lock_owner`
* `status` - a Semaphore
* `Lock()` - calls `status->P()`
* `Acquire()` - calls `status->V()`

## Race Conditions
If two threads are modifying a shared `value`, the `value` must be wrapped carefully, to prevent a scenario where a context switch during a critical section will not allow another thread to enter its critical section. 