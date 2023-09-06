# User-Level Thread Library

## Summary

This report details the creation of a static library, **libuthread.a** that can 
implement user level threads. This static library provides public functions 
within **uthread.c** that create, run, and manage the user threads. Mainly, 
`uthread_run` to initialize the first two user-level threads, `uthread_create` 
for further thread creation, and `uthread_yield` for user requests to yield the 
currently running thread. In order to manage these threads the public library 
implements a queue in **queue.c** which manages a given process' user threads 
represented through instances of `struct uthread_tcb`. The library further 
provides access to one of the most popular synchronization primitives, 
semaphores, within **sem.c**. The library also provides access to preemption 
through **preempt.c** and a boolean passed to `uthread_run` which guarantees 
any given thread will yield through an interrupt timer sending a preemption 
signal every 100hz. All of the aforementioned implementations follow 
from a private context API distributed through **private.h**. 

## Implementation

>The user-level thread library is split into two sections: public functions 
>accessible to processes creating threads and private preemption, API, and 
>helper functions.

### Public

1. Design a means to store a given thread's TID, context, running state, and 
stack. 
2. Design a helper class that lets the thread library store and access ready to 
run threads in a first in first out manner.
3. Save the user's currently running process thread as an idle thread, and 
initialize the first user-level thread.
4. Allow the user to initialize as many user-level threads as allocated memory 
permits.
5. Provide a means for the user to request the currently running user-thread 
yield to another user-level thread or the idle thread.
6. Provide a public class available to the user to creation of semaphores for 
thread synchronization.

### Private

1. Utilize context API functions while maintaining their private status.
3. Handle proper preemption interrupting and thread yielding with helper 
functions that are hidden to user-level threads.

#### Thread Storage

In order to keep track of any given thread's thread id, context, running state, 
and stack we created a `struct uthread_tcb`. This data structure will hold all 
the information a thread requires to switch into its previously running 
context, or initialize its previously running context at the top of its code.
To manage a thread state we utilized an int and the following defines:
1. READY = 1
2. RUNNING = 2
3. BLOCKED = 3
4. ZOMBIE = 4

#### First in First out thread storage

To hold our threads we made a first in first out data structure which 
initializes a queue of READY threads from **queue.c's** `queue_create()`. This 
queue implementation makes a singly linked list of type defined `queue_nodes` 
and provides useful helper functions: `queue_enqueue()` to push a node onto the 
back of the stack, `queue_dequeue()` to pop the top of the stack and return it 
using a `void** data`, `queue_destroy()` to empty and free an entire queue, and 
`queue_delete()` to search for a given node within the queue; delete that item; 
and return the deleted node's data through a `void* data`, and further 
functions to return queue variables and iterate over the list given a function. 

#### Initialize user-level thread library

To do this, we utilize `uthread_run(bool, uthread_func_t, void* )`, within the 
**uthread.c**. This function takes in a boolean value to decide if preemption 
will be required and a function pointer and void* to initialize the first 
user-level thread. Before creating the first user-level thread this function 
will save all the required information to create a 
`struct uthread_tcb* tcb_idle` to store the current process' information for 
use as an idle thread and initialize the previously discussed queue called
`thread_lib`. Once idle is saved, `thread_create` is called with the function 
and void* as  arguments and enqueues the new thread into `thread_lib`. The idle 
thread then locks itself into a while loop based on if `thread_lib` has a next 
READY thread, set the currently running thread to idle, and call yield to enter 
the first user-level thread.

At the beginning of `uthread_run` preemption will have been decided based on 
the passed boolean and throughout the function we utilize `preempt_disable()`
and `preempt_enable()` to protect sensitive parts of the user-thread library 
against preemption that can result in lost or corrupted initialization.

#### Further thread creation

Once `uthread_run` has been called by the user's process the currently running 
thread will be the first of as many user-level threads as is required. Thus, 
`uthread_run` will need to create new user-level threads properly regardless of 
previous execution and must run to completion so we also protect it from 
preemption. To do this it will utilize the private context functions described 
below to continually initialize valid memory chunks such that a new thread's 
stack and context are valid before setting its state to RUNNING and giving it a 
thread ID value that is unique, starting at 0 for idle and incriminating by 1.  

#### Allow the user-level threads to yield

This is done through `uthread_yield` which accurately switches context and acts 
as a simple scheduler based on the age of threads in the queue `thread_lib`. To 
do this it will set the currently running thread's state to READY from RUNNING 
and enqueues it to `thread_lib` if it's not a zombie or exiting to allow it to 
return and eventually run to completion. An important note is that no thread of 
any state other that READY should or will be added to `thread_lib` thus making 
it a queue of only READY threads. It then acts as a simple scheduler by 
dequeuing the oldest READY thread and setting it to `curr_thread` before 
changing its state to RUNNING. It then uses private context functions to 
context switch and into the new thread. All of this function will be protected 
from preemption due to the sensitive nature of `curr_thread` and the dequeued 
`next_thread` being lost.

#### User-level thread semaphores

A simple binary semaphore class is provided in **sem.c** to allow user-level 
threads to prevent starvation and deadlocking. It provides a `semUp()` and 
`semDown()` function which add blocked threads to a `blocked_queue` which will 
all eventually be released before completion. This semaphore class uses helper 
functions within **uthread.c** to block and unblock threads and make 
`uthread_yield` calls. Semaphores have small pieces of preemption sensitive 
code that must be protected to ensure threads don't get lost while being 
blocked or in the process of creating the semaphore.

#### Private context API functions 

These sensitive functions provided through **ucontext.h** and distributed to 
**private.h** are implemented in **context.c** and **preempt.c**. These private 
files are not to be accessible to user_threads outside of a passed boolean to 
`uthread_run` for preemption. The functions within context.c are implemented 
**ucontext.h** functions that allow uthread and it's functions to allocate 
valid stacks, contexts, and make the subsequent context switches through 
`uthread_ctx_switch`.

#### Preemption

Our implementation of preemption uses an `itimerval` that runs at 100hz which 
is equivalent to 10000 microseconds. Every 100hz this timer will send a 
`SIGVTALRM` which gets handled by a private function in **preempt.c**. This 
preemption timer can be enabled and disabled by calling `preempt_enable()` or 
`preempt_disable()` within public classes but both of which are still private 
to user-level threads. These functions are used to ensure proper data 
continuity during points where a preemption can cause of a thread to be lost 
from a queue or an illegal context change once it resumes after the preempt.

### Testing

To test our user-level thread library we first used the provided test **queue_
tester_example.c** and made many different test cases for edge scenarios of our 
queue. Once we had a robust queue implementation it made implementing our 
working `thread_lib` and subsequent uthread functions easy. We then tested it 
with the provided testing files for uthread. We also made our own uthread test 
cases to track down thread switching errors. We then tested with the provided 
semaphore test cases and modified them to further ensure they were effective 
locks. The **preempt_test.c** we made to demonstrate preemption does this 
through a hello thread which prints hello world iteration: # in a while loop  
with no `uthread_yield` call which runs until a preempt signal is received from 
the timer causing it to escape into `thread2`. This `thread2` then sets the 
while loop's boolean to be false and once the `hello` thread is returned into 
it will print the escaped through preemption `printf()`.

### Limitations and peculiarities

Throughout our project we attempt to guarantee that every `malloc()` call has a 
matching free call before the user process returns out of the idle while loop. 
We have a small block of memory leakage that we believe stems from 
`queue_delete()` but were unable to find a solution based on our implementation 
and where we call the aforementioned function. Moreover, our use of 
`preempt_disable()` is too extensive in certain functions such as 
`uthread_yield` in an attempt to play it extra safe when preemption interacts 
with semaphores. 

### Sources

1. https://www.ibm.com/docs/en/i/7.3?topic=ssw_ibm_i_73/apis/sigpmsk.html
2. https://www.gnu.org/software/libc/manual/2.36/html_mono/libc.html#Signal-Actions
3. https://www.gnu.org/software/libc/manual/2.36/html_mono/libc.html#Setting-an-Alarm
4. https://www.gnu.org/software/libc/manual/2.36/html_mono/libc.html#Blocking-Signals
5. Lecture 08.sync -slide 21
6. Discussion makefile.pdf -slide 22
7. Referenced signal.c -Credit to Joël Porquet-Lupine
