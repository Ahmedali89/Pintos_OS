Design Document for Project 1: Threads
======================================

## Group Members

* Denny Hung <dennyhl@berkeley.edu>
* Kin Seng Chau <elviskschau@berkeley.edu>
* Khang Kieu <ankhangkieu@berkeley.edu>
* Ahmed Ali  <ahmedali89@berkeley.edu>


## Design Document:
=================

### Task 1: Efficient Alarm Clock
---------------------------------

#### Data Structures and Functions
- Inside `struct thread` in (thread.h), add a variable called `time_to_wake_up` that represents the time it should wake up. Whenever a thread is put to sleep, set `time_to_wake_up = timer_ticks() + ticks`.
- Add a variable called `sleep_Queue` in (thread.c) - sorted doubly linked list data structure by ascending `time_to_wake_up`. Since the queue is in ascending order, `schedule()` can be used to unblock and remove all threads that satisfy the condition `time_to_wake_up > timer_ticks()`.

#### Algorithms

*  Changes to `timer_sleep()`:
    - Disable interrupts.
    - Set the current thread’s `time_to_wake_up = timer_ticks() + ticks`.
    - Push the running thread to `sleep_Queue`.
    - Call `thread_block()`.
    - Set `intr_level` to old value.

* Modifications to `schedule()` to handle  `sleep_Queue` as well:
    - Disable interrupts.
    - Check the `sleep_Queue` for any thread to wake up.
    - Call `thread_unblock()`.
    - Pop the awaken thread from `sleep_queue`.
    - Set `intr_level` to old value.


#### Synchronization Issues
 * One big synchronization issue with our algorithm is that while a thread is doing list operations on `sleep_Queue`, `timer_sleep()` and `schedule()` could be accessing the list at the same time.
 * In order to avoid this issue, we disable interrupts whenever we make a thread sleep/wake up.

#### Rationale

* Alternative Approach:
        - Each thread contains a variable that keeps track of how many ticks does it need to wait until it wakes up. When a thread is put to sleep, we update that variable, and set its state to blocked.
        - Each time `schedule()` is called, we check every thread (using for_all) to see which thread needs to wake up.
        - This alternative requires no additional data structure to keep track of sleeping threads, but each time schedule is called, we need to check every single thread, which is inefficient if a large number of threads exist.

 
### Task 2: Priority Scheduler
------------------------------

#### Data Structure and Functions

* Modify `struct thread` by adding:

    - `int base-priority; the original priority of the thread.`
    - `lock *waiting_lock;  // a lock pointer to the lock that is blocking the thread.`
    - `list thread priority_donors; //a list of all priority donors (blocked threads) of this thread.`
```
struct thread {
	...
	int base_priority;
	lock* waiting_lock;
	list thread prority_donors;
}
```

#### Algorithms
* Choosing the next thread to run:
        - The order of the threads in the ready_list.

    * Acquire a lock:
        - When a thread wants to acquire a lock, if that lock has no holder, it will become the holder of the lock; otherwise it goes into the "waiters" list of the lock and we do the "Changing thead's priority" part.

        ~ For example:
            - Thread A's priority  = 5 and has a lock "L"
            - Thread B's priority  = 30
            Here the changing in priorities happens => thread A priority = 30 same as thead B, and A will save its old priority. 
            (This is called Priority Donation)

        - Every time we want to acquire a lock from the "waiters" list of the lock, we sort the "waiters" right before sending the acquisition.
            
        [![Acquiring a Lock](https://s13.postimg.org/ddrzhg0l3/Screen_Shot_2018-02-14_at_12.40.28_AM.png)](https://postimg.org/image/8f4h2wws3/)


    * Releasing a lock:
        - In `sema_up()`, if the list of waiters is not empty, we should first sort by effective priority to decide who gets the lock next.
        
       [![Releasing a Lock](https://s13.postimg.org/xxwtfxo1z/Screen_Shot_2018-02-14_at_12.41.28_AM.png)](https://postimg.org/image/w63ul14oz/)


    * Computing the effective priority:
        - Finding the max priority between threads in priority_donors list and the effective priority of the lock holder.
        `effective_priority = max(priority, for(thread T in priority_donors))`.

    * Priority scheduling for semaphores, lock and condition variables:
        -  Every time we release a lock or increase the semaphore from 0 to 1, there is a new available lock/value for a new thread to take it. We check for the waiters list inside the semaphore and pop out the thread with highest priority in the list if there is one. That thread will be the holder of the lock/value of semaphore.

    * Changing thread's priority:
        - The holder thread of the lock will receive donation of priority from the lock's list of waiters.
        - Regarding recursive modification for threads' priorities see the example below:
           ```
             lock "L1"      lock"L2
             -------        -------        -------
            |       |<---- |      |<----- |       |
            |   A   |      |   B  |       |   C   |
             -------        -------        -------
         ```
    - Assume thread C is waiting for lock "L2" that is taken by thread B, and thread B is waiting for lock "L1" that is taken by thread A.
            - This implies that:
                - `waiting_lock` of C points to B.
                - `waiting_lock` of B points to A.
                - `waiting_lock` of A points to NULL.
                Then the priority of the lock's holder in this case will be `max(priority of thread in the lock, and all of its donors' priority in the donors list)`.
    - The loop is guaranteed to break since there will be no deadlock situation.

#### Synchronization Issues
* Updating priorities of a thread could be racy since an interrupt may be asking for a new highest priority thread to schedule (`next_thread_to_run()`) in the middle of priority updating (when a thread calls `set_priority()`). To prevent this from happening, we can disable interrupts until we are done updating the priority value for threads.
* Pointers to the threads on the `waiters` list are guaranteed to be valid because while they are waiting, the thread is blocked, so it cannot be deallocated.

#### Rationale

* The design we came up with had the recursive priority donation in mind. At first, we considered only adding a pointer to `lock` and the base (old) priority variable. However, if a thread holds multiple locks, it would be very difficult to update its priority (due to priority donation). This is why we needed to add a list of donors, which keeps track of every thread waiting on the current thread.

### Task 3: Multi-level Feedback Queue Scheduler

#### Data Structures and Functions
* Add a variable to `struct thread` declared as `int nice` which keeps track of a thread's niceness.
* Add a variable to `struct thread` initialized to `fixed_point_t recent_cpu = 0` which measures how much CPU time has this thread recently had.
* Add a global variable to `thread.c` initialized to `fixed_point_t load_avg = 0` which estimates the average number of threads ready to run over the past minute.
* In `thread.c`, add an array of list of threads of size 64 (one for each priority) to store threads that are ready to run. Each list at index `i`represents the list of thread with the given priority `i` and is put in order of Round Robin, so when the kernel chooses the thread of priority `i` to run, the first thread element in the list at index `i` will be chosen to run next.
* Implement `thread_set_nice(int nice)` and `thread_get_nice()` as standard getter and setter for a thread's `nice` variable.
```
struct thread {
	...
	int niceness;
	fixed_point_t recent_cpu;
}
```

#### Algorithms
* Calculate priority: `priority = PRI_MAX - (recent_cpu / 4) - (nice * 2)`.
* Calculate `recent_cpu = (2 * load_avg)/(2 * load_avg + 1) * recent_cpu + nice`.
    - Function: `int thread_get_recent_cpu(void);`
* Calculate `load_avg = (59/60) * load_avg + (1/60) * ready_threads`.
    - Function: `int thread_get_load_avg(void);`
            
#### Synchronization Issues
* There may be a race condition when we update the priority of a thread. This happens when a thread is updating its `nice` and `recent_cpu` attributes and another thread interrupts, which will make the calculation for new priority invalid. To fix this problem, we have to disable interrupt during the time that a thread is updating its `nice` and `recent_cpu` until the thread finishes updating its effective priority.
* Since interrupts will be disabled during the execution of the functions above, no two threads will access them at the same time.

#### Rationale
* Alternate approach: Using a Hash Table Data Structures to represent the threads that are in the ready list. The key of the Hash Table will be the priority, and each value will be a list of threads with that priority in a Round Robin order. We chose not to move forward with this approach because Hash Table is usually used to deal with dense data; however, in this case, we know that the threads can only 64 choices for their priority so we decided to use an array of lists instead of hash table.



### Additional Questions

#### 2.1.2.1 According to the project requirements, semaphores (and other synchronization variables) must prefer higher-priority threads over lower-priority threads. However, my implementation chooses the highest-priority thread based on the base priority rather than the effective priority. Essentially, priority donations are not taken into account when the semaphore decides which thread to unblock. Please design a test case that can prove the existence of this bug.

```
#include <stdio.h>
#include "tests/threads/tests.h"
#include "threads/init.h"
#include "threads/synch.h"
#include "threads/thread.h"
#include "threads/synch.h"

static thread_func medium_high_thread_func; //3
static thread_func high_thread_func; //4
static thread_func medium_low_thread_func; //2
static thread_func low_thread_func; //1

struct locks {
    struct lock *a;
    struct lock *b;
};

void
test_highest_efficient_priority(void) {
    struct lock a, b;
    struct locks locks;

    /* This test does not work with the MLFQS. */
    ASSERT (!thread_mlfqs);

    /* Make sure our priority is the default. */
    ASSERT (thread_get_priority () == PRI_DEFAULT);

    lock_init (&a);
    lock_init (&b);

    locks.a = &a;
    locks.b = &b;

    thread_create ("high", 4, high_thread_func, &locks);
    thread_yield();

    thread_create ("low", 1, low_thread_func, &locks);
    thread_yield();

    thread_create ("medium_high", 3, medium_high_thread_func, &locks);
    thread_yield();

    thread_create ("medium_low", 2, medium_low_thread_func, &locks);
    thread_yield();
}

static void
medium_high_thread_func(void *locks_) {
    struct locks *locks = locks_;
    msg("Medium High thread currently waiting on lock `b`");
    lock_acquire (locks->b);
    msg("Medium High is done running!");
}

static void
high_thread_func(void *locks_) {
    struct locks *locks = locks_;

    lock_acquire (locks->a);

    msg("High thread currently holding lock `a`");
    thread_set_priority(0);
    thread_yield();

    lock_release (locks->a);
    msg("High is done running!");
}

static void
medium_low_thread_func(void *locks_) {
    struct locks *locks = locks_;
    msg("Medium Low thread currently waiting on lock `a`");
    lock_acquire (locks->a);
    msg("Medium Low is done running!");
}

static void
low_thread_func(void *locks_) {
    struct locks *locks = locks_;
    msg("Low thread currently holding lock `b`");
    lock_acquire (locks->b);

    msg("Low thread currently waiting on lock `a`");
    lock_acquire (locks->a);
    msg("Low is done running!");

}
```
##### Explanation: Suppose we have 4 threads with priorities low (L), medium low (ML), medium high (MH) and high (H) respectively, and two locks (A and B). The following events occur:
* H acquires lock A.
* L acquires lock B and tries to acquire lock A. It gets put into A's waiting list. L is now blocked.
* MH tries to acquire B, which puts it into B's waiting list. If priority donation and priority scheduling worked correctly, L's priority should be equal to MH's. MH is now blocked.
* ML tries to acquire A. It gets put into A's waiting list, which now contains both L and ML. ML is now blocked.
* H releases lock A, so the priority scheduler should pick between L and ML, which are both in A's waiting list.
* If no bug existed, L would be selected since MH donated its priority to L while waiting for B.
* In this case, if the bug exists, ML should get lock A and continue running.

#### 2.1.2.2 Suppose threads A, B, and C have nice values 0, 1, and 2. Each has a `recent_cpu` value of 0. Fill in the table below showing the scheduling decision and the `recent_cpu` and `priority values` for each thread after each given number of timer ticks. We can use R(A) and P(A) to denote the `recent_cpu` and `priority values` of thread A, for brevity.

```
timer ticks | R(A) | R(B) | R(C) | P(A) | P(B) | P(C) | thread to run
------------|------|------|------|------|------|------|--------------
0           |   0  |   0  |   0  |  63  |  61  |  59  |      A
4           |   4  |   0  |   0  |  62  |  61  |  59  |      A
8           |   8  |   0  |   0  |  61  |  61  |  59  |      B
12          |   8  |   4  |   0  |  61  |  60  |  59  |      A
16          |  12  |   4  |   0  |  60  |  60  |  59  |      B
20          |  12  |   8  |   0  |  60  |  59  |  59  |      A
24          |  16  |   8  |   0  |  59  |  59  |  59  |      C
28          |  16  |   8  |   4  |  59  |  59  |  58  |      B
32          |  16  |  12  |   4  |  59  |  58  |  58  |      A
36          |  20  |  12  |   4  |  58  |  58  |  58  |      C
```
#### 2.1.2.3 Did any ambiguities in the scheduler specification make values in the table (in the previous question) uncertain? If so, what rule did you use to resolve them?

In the previous problem, the "round robin" order the specification proposes causes ambiguity because when two or more threads have the same priority, there needs to be a rule to tiebreak. The ambiguity was resolved by using the rule to run the thread that has been on the queue for the longest amount of time when there are several threads with the same highest priority. In other words, whichever thread is first in the queue will run next.
