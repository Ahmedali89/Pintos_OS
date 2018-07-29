Design Document for Project 2: User Programs
======================================

## Group Members

* Ahmed Ali  <ahmedali89@berkeley.edu>
* Denny Hung <dennyhl@berkeley.edu>
* Khang Kieu <ankhangkieu@berkeley.edu>
* Kin Seng Chau <elviskschau@berkeley.edu>

## Design Document:


### Task 1: Argument Passing
---------------------------------

#### Data Structures and Functions

- Helper function to check if the argument size is larger than the page allocated.
- `char* parse_cmdline(char* cmd_line, char* filename)` - Helper function to parse the user command line using `strtok_r(...)` from `lib/string.c` (Breaks the user command to executable file and arguments)
- `void push_args(char ** args, size_t num_args, void **esp)` - Helper function to push parsed arguments to the stack. 

#### Algorithms

* During `load()`, before we attempt to open the file associated with `file_name`, we parse the arguments.
*  Following the 80x86 Calling Convention (3.1.8, 3.1.9), we push the parsed arguments by calling `push_args(...)` during the `load()` function after `setup_stack(esp)` returns successfully.


#### Synchronization Issues
 * Regarding the case where two different threads are calling` process_execute()`, there is no synchronization issue in this part because if two process calling `process_execute()` the two processes have to go through `thread_create(...)` where interrupts are disabled at the beginning and enabled at the end. 
    * For example, let’s says we have two processes P1 and P2 both are calling `process_execute()` then on `thread_create()` one process will move on in execution while interrupt is disabled as a result P2 can not do anything until the P1 enables the interrupt and continue in executing other lines of code. At that moment P2 will continue into `thread_create()` and similarly will do the same thing as P1.
* There is a synchronization issue when a child process is created
    * We should not allow the parent process to exit `exec()` until the child has loaded its executable into the stack (or failed to do so). 
    * For this reason, we use a binary semaphore to allow the parent to wait for the children to finish loading. (related to part 2).


#### Rationale

* We considered pushing the exec arguments in `setup_stack(..)` but this requires additional parameters, since we don’t pass neither the filename nor the tokens corresponding to the arguments to the process.
* Not doing it in start_process? 
    * It is more intuitive to push arguments into the stack during `load(...)` rather than doing so during `start_process()`.


 
### Task 2: Process Control Syscalls
------------------------------

#### Data Structure and Functions

* In `struct thread` add `struct list children_status` to keep track of the current thread’s children. 

* In `thread.h`, we declare the following `struct` to synchronize the `wait` operation, because the parent process should always know the exit status of its child.

```
struct wait_status {
    struct list_elem status_elem;   /* ‘children_status’ list element. */
    struct lock lock;
    int ref_count                   /* 0, 1, 2: Number of threads alive in the relationship.
    tid_t tid;                      /* Child thread id. */
    int exit_code;                  /* Child exit code, if dead. */
    struct semaphore dead;          /* Value is 1 when child is alive, and 0 if it is dead. */       
};

struct thread {

    ... 
    struct thread *parent             /* This thread’s parent process. */
    semaphore load_child              /* Waits for child to finish loading. */           
    struct wait_status *wait_status;  /* This process’ completion state. */
    struct list children_status; /* Completion status of children. */
    ...
};

userprog/syscall.c
#include “devices/shutdown.h”

…

static void syscall_exit (void) {
    shutdown_power_off();
}
```

#### Algorithms
* Wait: 				
    - In `process_wait (tid_t child_tid)`:						
        - Iterate through list of child processes:
            - If no child process tid matches tid parameter, return -1
                - If child process tid matches tid parameter, call sema_down on the `dead` semaphore associated with that child process (when child process exits, it will sema_up on that same semaphore, waking up the parent process). 
        - After waking up:
            - Set exit code to terminated child process’ exit code from `struct wait_status`
            - Destroy the shared data structure and remove it from the list.
            - Return exit code.		

* Exit: 
    - Acquire lock corresponding to the `struct wait_status` shared between this child thread and its parent. (to change ref_count).
    - Save the exit code in the shared data. 
    - Decrement `wait_status->ref_count` by 1.
    - If wait_status -> ref_count == 0: free the wait_status struct
    - Otherwise, `sema_up` the `dead` semaphore in the data shared with our parent process. 
    - Iterate through the list of children, for each wait_status corresponding to them:
        - Acquire lock from wait_status shared with child
        - Reduce `ref_count` by 1
        - `If child is dead already, free the wait_status struct
        - Release lock from `wait_status` shared with child.
    - Release lock from `wait_status` shared with parent
    - Terminate the thread.

* Exec: 
    - Modify `process_execute()` function. Check if the given pointer is valid (not NULL and doesn’t violate the user’s address space bounds).
    - Inside `thread_create()`, call `malloc` to initialize the `wait_status` corresponding to this pair.
    - In `process_execute()` after calling `thread_create`, down the `load_child` semaphore, so the parent has to wait until the child load the executable into the stack before exiting `exec`.
    - After the child finishes loading and setting up the stack, (or it errors) `up` the parent’s `load_child` semaphore. (This can occur before or after the previous bullet point).


* Halt: 
    - Create a static function in `syscall.c` that calls `shutdown_power_off()` from ‘devices/shutdown.h’

* Practice: 
    - Create a static function in `syscall.c` that takes an integer `i` and returns `++i`.

#### Synchronization Issues
* The synchronization issue happens when a thread tries to call `process_exit`, yet there is a possibility that the parent thread wants to exit as well. Therefore, at this moment the race condition might happens when a child and parent try to exit at the same time. Thus, Locks must be applied because in the existing situation the share object `wait_status` is being modified, and we do not want any conflict during editing the `wait_status`.
* Another synchronization issue occurs when the parent process calls `thread_create()` to create the child thread the parent must wait until the child loads its executable. This is done by using `load_child` semaphore. The parent will decrements the `load_child`(inside the struct wait_status which is initialized in during the `thread_create()` call), and this will make the parent waits for the child to load. Then when the child increments the parent process shall proceed in execution.   

#### Rationale

* Initially, we consider putting a lock for parent processes to wait for their child to finish loading inside struct `wait_status`, but it makes more sense to put it in struct `thread` because a process may only be waiting for one child thread to load its executable at the same time.
* We considered using two booleans `child_alive` and `parent_alive` instead of `ref_cnt` to keep track of the number of threads alive, and free the shared memory with both booleans are false. However, we found that condition checking and implementation will be a bit easier if we use one single variable `ref_cnt` for the freeing shared memory logic.

### Task 3: File Operation Syscalls

#### Data Structures and Functions

* In our `struct thread`, we add the following:
```
struct thread {
    ...
    file * exec_file;       /* Pointer to file this process is executing (used for deny/allow writes) */
    int fds[128];           /* A list of all the file descriptors that this thread open */
    ...
};
```

* In `syscall.c`, add 
```
static lock sys_lock;           /* A lock to ensure only one syscall is allowed at a time */
static struct list file_list;   /* A list of file_list_elem defined below. */

/* Wrapper to iterate through the file_list. 
Contains mapping between fd and file* */
struct file_list_elem {
    int fd;
    file * file;
    struct list_elem file_elem;
};

/* Increment when we make a new file descriptor.
static unsigned int nextfd = 2; 
```
#### Algorithms
* During `load`, we save a pointer to the executable file inside `struct thread` after it is opened successfully. Next, we call `file_deny_write()` to deny writes to the file being executed. 

* During `process_exit()`
    * Before we destroy the page, call `file_allow_write()` to allow writes to the file this process was executing. 
    * Iterate through all open file descriptors, closing each of them and deallocate the memory for the `file_list_elem` associated with them.

- Helper function to validate the arguments on the stack before calling the kernel filesys functions. Checks for NULL pointers and for addresses that are out of range for the user.
- For all the following syscall handler functions, we always acquire `sys_lock` and validate **all** the arguments before doing any operation. `sys_lock` is released right before returning from the function. In each function, "return" means putting the return value to the EAX register, as stated in the specs.

					
    * Create: 
        * Return`filesys_create(filename, size)`

    * Remove:
        * Return`filesys_remove(filename)`
    * Open: 
        * Get `file *` using `filesys_open(filename)`
        * Allocate memory for a file_list_elem
        * Let `fd = nextfd++`.
        * Add `fd` to `fds` in `struct thread` of the current process.
        * Add the `file_list_elem` to `file_list`
        * Return `fd`

    * Filesize: 
        * Find `file *` associated with `fd`
        * Return `file_length(file *)`

    * Read: 
        * Find `file *` associated with `fd`
        * Return`file_read(file *, buf[], size)`

    * Write: 
        * Find `file *` associated with `fd`
        * Check if file->deny_write is false.
        * If false, call `file_write(file *, buf[], size)`

    * Seek: 
        * Find `file *` associated with `fd`
        * Call `file_seek(file *, pos)`
    
    * Tell: 
        * Find `file *` associated with `fd`
        * Return`file_tell(file *)`

    * Close: 
        * Find `file *` associated with `fd`
        * Call`file_close(file *)`
        * Remove `file_list_elem` corresponding to `fd`
        * Remove `fd` from the `fds` array in `struct thread`
	
    * Exit:
    	* Right before lock release: If there is any files opened by this process (check the list of fds from task 3) that haven't been closed, iterate through the list of `fds` and call `file_close()` function from `syscall.c` to close the files.

#### Synchronization Issues

* For this task, the synchronization issue is taken care of by acquiring a global `sys_lock` for all file operations. This ensures that at most one process is doing a file operation at any given time.



#### Rationale



* At the beginning we thought that one file could be shared by multiple users. In fact we were thinking about make a struct as the following:
    
```
struct file_lock {
    inode* fd = NIL;
    int AS: Number of active seekers; initially = 0  
    int WS: Number of waiting seekers; initially = 0
    int AR: Number of active readers; initially = 0
    int WR: Number of waiting readers; initially = 0
    int AW: Number of active writers; initially = 0
    int WW: Number of waiting writers; initially = 0
    Condition okToRead = NIL
    Condition okToWrite = NIL
    Condition okToSeek = NIL
}
```

However, it is mentioned in the specs that only one thread may do a file operation syscall at the same time. We thought that there could be more than one active readers for one file, yet this is not the case.

### Additional Questions

#### 1. Take a look at the Project 2 test suite in pintos/src/tests/userprog. Some of the test cases will intentionally provide invalid pointers as syscall arguments, in order to test whether your implementation safely handles the reading and writing of user process memory. Please identify a test case that uses an invalid stack pointer ($esp) when making a syscall. Provide the name of the test and explain how the test works. (Your explanation should be very specific: use line numbers and the actual names of variables when explaining the test case.)

Test : `sc-bad-sp test`, (line 18)
How it works? The test tries to invoke a system call with the stack pointer (%esp) set to a bad address: -(64\*1024\*1024).  As a result, the process must be terminated with -1 exit code.

#### 2. Please identify a test case that uses a valid stack pointer when making a syscall, but the stack pointer is too close to a page boundary, so some of the syscall arguments are located in invalid memory. (Your implementation should kill the user process in this case.) Provide the name of the test and explain how the test works. (Your explanation should be very specific: use line numbers and the actual names of variables when explaining the test case.)

Answer: Test: `sc-bad-arg`, (line 14). 
How it works: Sticks a system call number (SYS_EXIT) at the very top of the stack, then invokes a system call with the stack pointer (%esp) set to its address (0xbffffffc).  However, the argument to the system call, located at `0xbffffffc + 4` would be above the top of the user address space.


#### 3. Identify one part of the project requirements which is not fully tested by the existing test suite. Explain what kind of test needs to be added to the test suite, in order to provide coverage for that part of the project. (There are multiple good answers for this question.)

Answer: The existing test suite does not test the case when we remove an opened file and try to read/write to the file. According to the specs, any process that owns a file descriptor to the removed file should still be able to access it. We only delete the file after the last file descriptor to that file has been closed or the machine shuts down. To test this case, we can have two processes open the same file, then have one of the user programs call `remove`. The other process should still be able to use its `fd` to read/write to that file.


#### 4. GDB Questions
##### a. Set a breakpoint at `process_execute` and continue to that point. What is the name and address of the thread running this function? What other threads are present in pintos at this time? Copy their struct threads. (Hint: for the last part `dumplist &all_list thread allelem` may be useful.)
Answer: 

name : main
address: 0xc000e000

Other thread: `idle`


##### b. What is the backtrace for the current thread? Copy the backtrace from gdb as your answer and also copy down the line of c code corresponding to each function call.
Answer:
```
#0  process_execute (file_name=file_name@entry=0xc0007d50 "args-none") at ../../userprog/process.c:32
#1  0xc002025e in run_task (argv=0xc0034cac <argv+12>) at ../../threads/init.c:288
#2  0xc00208e4 in run_actions (argv=0xc0034cac <argv+12>) at ../../threads/init.c:340
#3  main () at ../../threads/init.c:133
```
Lines of code:
```
#0:   tid_t process_execute (const char *file_name) {
#1:   process_wait (process_execute (task));
#2:   a->function (argv);
#3:   run_actions (argv);
```

##### c. Set a breakpoint at start_process and continue to that point. What is the name and address of the thread running this function? What other threads are present in pintos at this time? Copy their struct threads.

Answer:
Name: `args-none`
Address: 0xc010a000
Other threads: `main` and `idle`


##### d. Where is the thread running start_process created? Copy down this line of code.
Answer:
```
../../threads/thread.c:462
function (aux);       /* Execute the thread function. */
```

##### e. Continue one more time. The userprogram should cause a page fault and thus cause the page fault handler to be executed. It’ll look something like
```
[Thread <main>] #1 stopped.
pintos-debug: a page fault exception occurred in user mode
pintos-debug: hit ’c’ to continue, or ’s’ to step to intr_handler
0xc0021ab7 in intr0e_stub ()
``` 
##### Please find out what line of our user program caused the page fault. Don’t worry if it’s just an hex address. (Hint: `btpagefault` may be useful)

Answer: 
0x0804870c


##### f. The reason why btpagefault returns an hex address is because pintos-gdb build/kernel.o only loads in the symbols from the kernel. The instruction that caused the page fault is in our userprogram so we have to load these symbols into gdb. To do this use `loadusersymbols build/tests/userprog/args-none`. Now do `btpagefault` again and copy down the results.

Answer:
```
#0  _start (argc=<error reading variable: can't compute CFA for this frame>, argv=<error reading variable: can't compute CFA for this frame>) at ../../lib/user/entry.c:9
```

##### g. Why did our user program page fault on this line?

Answer:
User program tries to access unmapped memory space.
