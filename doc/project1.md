Design Document for Project 1: User Programs
============================================

## Group Members

* Michael Duong <mduong15@berkeley.edu>
* Nicholas Saquilayan <nsaquilayan@berkeley.edu>
* Tobey Yang <minchu.yang@berkeley.edu>
* Sheng Lo <shenglo1@berkeley.edu>

# Task 1: Argument Passing
### 1.1 Data Structures and Functions

__Data Structures:__
- **char * arr[]** - We will use a dynamically allocated array of C string pointers to hold the file name and arguments as we're parsing them, before putting them on the stack. Each pointer will point to a complete, space-delimited argument. If the number of arguments are larger than the allocated array size, we will realloc the memory and double its current size. This will allow for an arbitrary number of arguments to a program. It will be implemented in the function, 
```bool load (...)```, in process.c.
   
**Functions:**

- __void push_args_to_stack(char* arr[], void **esp)__ - In this function, we will take in the previously parsed char array, and copy the memory from arr to the current address to which the stack pointer points. After pushing each argument into the user stack, we will update the stack pointer according to the size of each argument so it correctly points to the bottom of the stack.

### 1.2 Algorithms

First, we will parse the file string by using the ```strtok_r``` function; we will use a while loop to add each space delimited word 
into the dynamically allocated array of C strings (named _**arr**_) which will have an initial size of 10. Meanwhile, we will keep a 
counter _**argc**_ that has the total number of arguments once iteration is over. If there are more words than the size of the 
array, then we will realloc _**arr**_ to twice its size. After the while loop, we realloc _**arr**_ to _**argc**_ elements if there is extra space not being used to conserve the amount of memory used. Lastly, push the C strings in reverse order onto the stack, effectively in right to left order to follow the 80x86 calling convention, and free _**arr**_.

**Edge cases:**
- If the argument _**file_name**_ to ```process_execute``` is an empty string, the parsing array _**arr**_ won't be filled. However, it is possible that the memory initially allocated on the heap at _**arr[0]**_ is a coincidentally valid executable, and at the very least would be garbage, but ```filesys_open``` would attempt to open that executable, which is undesired behavior. We solve this by additionally checking if _**argc**_ is zero, and if so, print an error message and avoid starting the process.

### 1.3 Synchronization

Since there is no data shared across threads, we do not need to use synchronization primitives to maintain correctness. The _**file_name**_ passed into ```process_execute``` is specific to the process attempted to be started and is not shared with any other process, so there is no need for synchronization.

### 1.4 Rationale

Even though this section is supposed to be for the implementation of ```process_execute(char *filename)```, through tracing the calls within the function and figuring out how it works, we realized that a better location to actually do the argument parsing is within ```load(...)```. The rationale behind this is that what happens in ```process_execute``` is the creation of the new thread. If we were to parse the arguments before that and there was an issue with thread creation, we would have wasted time parsing the arguments. By pushing it within the loading process, i.e. inside ```load(...)```, we don’t have to do the argument passing unless we know the thread was at least created. 

In regards to why we decided to use an array instead of the given list API, the increased overhead required to implement the list was not necessary. We need to create a new struct and call the list functions, but our only goal is to parse the arguments and put them into a stack. By using an array, we use less space since we avoid having to store the _**prev**_ and _**next**_ pointers, and we have better efficiency because we only need to call ```realloc``` in the cases where there are more arguments than we can fit, as opposed to a function every time to add an element to the list.

# Task 2: Process Control Syscalls
### 2.1 Data Structures and Functions
__Data Structures:__

In thread.h, struct thread, we are adding the following things to the USERPROG region that represents the PCB:
- __struct list* children__: It stores a list of children of the current thread/process (list of thread struct).
- __struct list_elem child_elem__: Used to be able to traverse the children of a process.
- __struct semaphore* parent_child_sema__: Used in two cases. (1) in ```exec```, this will make sure that the process gets loaded before the call to exec returns. (2) If a parent calls ```wait``` on its child, the semaphore is used to make sure a child is finished before the parent unblocks.
- __int exit_code__: It stores the exit code of the thread to which the PCB belongs when it is finished.
- __bool call_wait__: Checks if the parent has already called ```wait``` on this child.
- __bool parent_dead__: Used by parents to notify that they have died so that when the child is done, it knows to free its own resources.
- __bool success__: Used for the parent in ```exec``` to determine if loading the executable succeeded.
- __struct lock* dying_lock__: Ensures there is no race condition between parent and children when they are dying.

**Functions:**
- __void exit_handler (int status)__: In ```exec``` a parent process needs its child’s exit status, so we modify it to save the exit status in the thread struct.

- __int practice_handler (int i)__: In syscall ```practice```, we will implement the function so that it increments the passed in integer argument by 1 and returns it to the user.

- __void halt_handler (void)__: We will call ```shutdown_power_off(void)``` to power down the machine we are running.

- __pid_t exec_handler (const char *cmd line)__: When a user calls this function, ```process_execute``` gets called to create a child process. Then call ```add_child``` to keep track of all the direct children this process has. 

- __int wait_handler (pid_t pid)__: We will add a couple data structures and variables in thread.c, struct thread to make sure that parent will wait for its child.

- __void copy_args_to_kernel (uint32_t* user_args, int number_of_args, uint32_t kernel_args[])__: This function copies the arguments in user stack to kernel stack. ```pagedir_get_page``` will be used to map the user virtual memory space to kernel virtual memory space.

- __struct thread * get_child_from_tid(tid_t tid)__: There is no ```get``` or ```search``` function in the Pintos list API that searches on thread id. Given a thread id, return the thread with the matching thread id if it exists, otherwise return a null pointer. This will only look at children processes of a current process and return null if a tid is not one of the children.

- __void add_child(pid_t *child)__: When ```process_execute``` gets called, it returns a child tid. We will add the TCB corresponding to _**child**_ into the current thread’s children list.

### 2.2 Algorithms
In the ```syscall_handler```, we first retrieve the syscall number from the user arguments and copy it to the kernel stack. Then we use the sys number to call ```copy_args_to_kernel```, which puts all of the syscall arguments on the kernel stack. Lastly, call the appropriate function based on the syscall number.
- wait(): Get the thread struct of the pid that is passed in as an argument by looping through the child list of the current thread. Set the _**parent_is_dead**_ boolean value in the child to false, and call ```sema_down``` on _**parent_child_sema**_. When control is returned to the parent, return the child’s exit status.

    **Edge cases:**
    - ```Wait``` is called twice. Our solution was to add a _**called_wait**_ boolean variable to the thread struct, so that calling wait twice is not possible.
    - ```Wait``` is called on a thread that is not a direct child. Since we designed our solution around a list of direct children, this is not an issue.
- exec(): First call ```process_execute``` and save the child’s tid. Add the child to the current thread’s array of children. Call ```sema_down``` on _**parent_child_sema**_. In the child process, at the very end of the load function in process.c, save success to the thread’s success field and call ```sema_up``` on _**parent_child_sema**_. Once control is returned to the parent, access the child’s success field. If success is false, then return -1, otherwise return the child’s tid.

### 2.3 Synchronization
The main resource that will be shared between threads is the struct thread or TCB/PCB. This is shared between parent and child processes, and the two synchronization primitives we’re using are _**parent_child_sema**_ and _**dying_lock**_ that we add to the thread.h struct thread. Below are their uses:

- __parent_child_sema__: When a system call to exec occurs to create the new process, since ```process_execute``` only sets up the thread and does not load the new process, we need to wait until the process is loaded. Within the exec syscall, we will call ```sema_down``` to make the thread block until load gets called, in which the process actually gets loaded, and only then do we call ```sema_up``` and allow the exec syscall to finish. By using a semaphore, this ensures that regardless of the order in which the semaphore functions are called, exec will continue when the child process is loaded. We then reuse this semaphore in the case that the parent calls wait on the child. Because the ```exec``` syscall blocks until the child process is loaded, that means that a wait syscall for that child will not occur until the child is loaded and the semaphore is reset to 0, making it reusable. We reuse the semaphore in a similar fashion: parent calls ```sema_down``` and ```sema_up``` is called by the child only when it is finished in ```thread_exit``` to let the parent continue. 

- __dying_lock__: As a process is exiting, it will loop through its children and either free up the child’s resources if it’s already dead, or let the child know to free up its own resources upon completion (via the _**parent_is_dead**_ variable). To make sure that there’s no race condition between the child and parent dying, the child will attempt acquire its own _**dying_lock**_ for its entire exit process, and the parent will attempt to acquire the specific child’s _**dying_lock**_ of which it wants to check the status and possibly update the _**parent_is_dead**_ variable. That way, either the child is completely dead and the parent knows, or the parent was able to successfully let the child know to clean up its own resources before the child enters the exit process. Deadlock cannot occur because a process’s child cannot be one of its ancestors, so there must be some thread that is free to obtain a lock.

We believe these primitives to be at a reasonable granularity because they are only between parents and their immediate children, as it should be. These are used to maintain atomicity and correctness of order, so the only time these will cause threads to wait are when it’s intended (i.e. the wait syscall), or it is still dependent on other threads (i.e. when a new process hasn’t been loaded yet). The memory usage is very low since only two variables are added and we even reuse the _**parent_child_sema**_ for two purposes. In terms of runtime, this is O(n) because the only lookup that occurs is when a child loops through its children to find a TCB/PCB. We believe that with these primitives, this is the best way to maintain correctness with minimal wait time.


### 2.4 Rationale
We chose to implement the children as a list in the thread struct instead of an array because it is more memory efficient and time efficient. Using a dynamically allocated array would require a call to realloc every time the number of children exceeded the size, additionally when we expand the array there would likely be extra memory that is not being utilized for the duration of the thread’s or thread’s parent’s life. The list also works well with the tree-like structure of parent and children threads. One setback however is that implementing a list is considerably more difficult than using an array.

Another design choice we made was when to clean up memory from a thread that has been exited. To ensure that our ```exec(...)``` and ```wait(...)``` calls work correctly, we check the _**parent_is_dead**_ boolean, and if it is true then the thread frees its memory at the end of a ```thread_exit``` call. If it is false, it will wait until its parent calls ```thread_exit```. This logic follows because a thread only needs to persist its data past ```thread_exit``` if its parent is still running and needs its data.




# Task 3: File Operation Syscalls
### 3.1 Data Structures and Functions
**Data structures:**
- __struct *file fdt[]__ - A dynamic array of file struct pointers that represents the file descriptor table. Many of the syscall functions we need to implement for this task depend on a file descriptor, so we need to store each file for later reference. Many of the functions in file.c and filesys.c take a file struct pointer as an argument; this table is necessary to know which files each process currently has open to pass into these functions.

- __int empty_fdt[]__ - This dynamic array of integers keep track of the indices of empty spots in the file descriptor table.

**Functions:**
Note: Functions that have an integer argument fd refer to the _**fdt**_ data structure to get a file struct if the associated function has a file struct argument.

- __bool create_file_handler(const char *name, off_t initial_size)__: In this function, we will call the function, ```filesys_create (...)```, in filesys.c to create a new file initially _**initial_size**_ bytes in size.

- __bool remove_file_handler(char *name)__: When trying to remove the file, we call ```filesys_remove()``` to remove the file.

- __int open_file_handler(char *name)__: We will utilize the _**fdt**_ data structure to store the file struct and return the corresponding index, which is the file descriptor.  	

- __int filesize_file_handler(int fd)__: Call ```file_length(...)``` to return the size of the file in bytes.

- __int read_file_handler(int fd, void *buffer, unsigned size)__:  Calls ```file_read``` from file.c. Puts the bytes actually read into _**buffer**_ and returns the number of bytes read.

- __int write_file_handler(int fd, const void *buffer, unsigned size)__: Since ```file_write()``` has already checked if the file is locked or not, we just have to return the call to ```file_write()```.

- __void seek_file_handler(int fd, unsigned position)__: We will call ```file_seek(...)``` function in file.c to set the current position in file to _**position**_ bytes from the start of the file.

- __unsigned tell_file_handler(int fd)__: We return the call to ```file_tell(...)```.

- __void close_file_handler(int fd)__: When ```close_file_handler()``` gets called, it passes a file descriptor id. Then the pointer that points to the file gets removed from the file descriptor table and set to null. The empty space array, _**empty_fdt**_, gets updated.

### 3.2 Algorithms

When ```syscall_handler()``` gets called, check the syscall number and determine which function to call. If it is a filesys number, obtain the global lock. Call the corresponding filesys function. Filesys functions that take a file descriptor as an argument refer to the file descriptor table data structure. When a file descriptor is closed, add it to _**empty_fdt**_ to be reused. If opening a file and _**empty_fdt**_ is non-empty, remove the first  valid entry from the array and use that file descriptor. Once the filesys function has returned, release the global lock and return control to the user.

**Edge cases:**

- For functions that take a file descriptor as an argument, if the user passes in null pointer it will cause a segmentation fault. To prevent this we have a null check before accessing the file descriptor table.
- For functions that take a file descriptor as an argument, if the user passes in a file descriptor whose entry has been removed from the array, the process will try to dereference a null pointer and cause a segmentation fault. To prevent this we terminate the process.
- For functions that take in a file descriptor as an argument, if the user passes in a recycled file descriptor, i.e. a file descriptor that was closed before, they may expect that they are working with the same file, but they are actually working with whatever file was most recently opened. If the user is intending to work on the original file with the same file descriptor, this is a logic error on the user’s side and we allow it to happen since it is valid behavior.

### 3.3 Synchronization
Resources shared across threads include files in disk and memory, which are accessed from an interrupt context. 
Our synchronization strategy for executable files is to obtain a global lock on the file system in the ```load``` function in process.c before calling ```filesys_open```. Then if the ```filesys_open``` call was successful, call ```file_deny_write()``` on the executable and release the global lock. This ensures atomicity when opening an executable, that is a process cannot get interrupted between opening the file and changing its write permissions.
For other files, we obtain a global lock before calling the associated filesys function, and release the lock after the filesys function returns.

### 3.4 Rationale
Our design ensures correctness by enforcing atomicity when opening and closing files. However, given that the grain of our lock is very coarse, concurrency may be limited which we will aim to improve in project 3. 
We chose to use a dynamically allocated array instead of a list for _**fdt**_ because of the quick lookup time and low overhead. We had similar reasoning for _**empty_fdt**_, but additionally we treat it as a circular queue to ensure fragmentation is not an issue. With these two data structures however, if the user opens many files and closes most or all of them, we don’t have a way to free up the memory from _**fdt**_ or _**empty_fdt**_.

# Additional Questions
-  In sc-bad-arg.c line 14, `movl $0xbffffffc, %%esp;` the stack pointer is set to 0xbffffffc, which is above the _**PHYS_BASE**_ = 0xc0000000. Instead of pointing to a user virtual address, it points to a kernel address. The next instruction, `movl %0, (%%esp);` puts _**SYS_EXIT**_ into the top of the stack. And then “int $0x30” gets executed, it switches to kernel mode and passing the invalid address to kernel. Thus, the process should exit with error code.


- In sc-boundary-3.c line 13-14, p is set to be the highest possible user virtual address, which is one byte away from the invalid address. `“movl %0, %%esp; int $0x30" : : "g" (p));` the instruction puts the valid user address into stack pointer, then shifts to kernel mode. Although the first byte of the address is valid, the other 3 bytes of address are not valid user addresses. Thus, the process should exit with error code.

-  A type of test that should be added to the test suite is checking for unusual and empty string inputs, for example calling ```exec(“”)``` on the empty string would produce a -1 since opening a non-existent file would fail.
