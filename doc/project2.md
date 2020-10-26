Design Document for Project 2: Threads
======================================

## Group Members

* Michael Duong mduong15@berkeley.edu
* Nicholas Saquilayan nsaquilayan@berkeley.edu
* Tobey Yang minchu.yang@berkeley.edu
* Sheng Lo shenglo1@berkeley.edu

# Task 1: Efficient Alarm Clock
### 1.1 Data Structures and Functions
__Data Structures:__
- **int64_t done_time** - This variable will be in our TCB. It keeps track of the time (in kernel ticks)
when the thread needs to wake up, which is the current `timer ticks` + `sleeping time`. 
- **struct list sleep_queue** - We will use a Pintos list to store all our sleeping threads, sorted by `done_time` in ascending order. 

__Functions:__
- **bool timer_elapsed_comparator(int64_t, int64_t)** - This is the comparator function that gets passed in to `list_insert_ordered()` to make sure `sleep_queue` is sorted by `done_time`. For comparisons we use `done_time`.
### 1.2 Algorithms
Calculate the `done_time`, which is the time the `timer_sleep()` function was called plus the duration of the sleep. Assign the `done_time` to the `done_time` member of the TCB. Disable interrupts in `timer_sleep()` to atomically add the thread to the `sleep_queue` and call `thread_block()`. Re-enable interrupts right afterwards. Note that the thread is inserted into `sleep_queue` using `list_insert_ordered`, so order is maintained. In the `thread_tick()` function, check if the front element is done sleeping by checking if its `done_time` is greater than or equal to `ticks`. If it is, then remove it from `sleep_queue` and unblock it, and repeat the check for the next front element. Otherwise, return.
### 1.3 Synchronization
In the `timer_sleep()` function the only potential concurrent access to a shared resource is accessing the `sleep_queue`. To prevent this we disable interrupts while adding to the queue. Similarly in `thread_tick()` we access the `sleep_queue`, but since the `thread_tick()` function is called from `timer_interrupt()`, which requires interrupts to be disabled, we are protected from concurrent accesses and modifications to the list.
### 1.4 Rationale
We decided to use a sorted linked list instead of an unsorted linked list to prevent unnecessary iteration through the list to check for ready threads. Suppose there are m elements that are ready to be removed from the sleeping queue and put on the ready queue. Then with a sorted linked list we only have to check the first m elements. With an unsorted linked list we have to check the whole list every time, which is wasteful considering interrupts are disabled.

# Task 2: Priority Scheduler
### 2.1 Data Structures and Functions
**Data Structures:**
- __struct lock *waiting_on__ - This pointer variable will be in our TCB. It points to the lock that the current thread is waiting on if it's waiting on something. Otherwise, it will be `NULL`.
- **int effective_priority** - This variable will be in TCB to store the priority value donated by the thread which has a higher priority and is waiting on the lock.
- **struct list owned_locks** - The `owned_locks` list resides in TCB which keeps track of all the locks that the thread owns.
- **struct list_elem elem** - We will initialize this variable in `struct lock` and will be used in `owned_list` in TCB.
- **int priority** - A priority added to struct `semaphore_elem` in synch.c to sort the `waiters` for a condition variable.

**Functions:**
- **bool scheduler_priority_comparator(int64_t, int64_t)** - This is the comparator function that gets passed in to `list_insert_ordered()` to make sure `ready_list` in threads.c and `waiters` in `locks.semaphore` are sorted by `priority`. For comparisons, we use `priority`.
- __void priority_donation (struct lock*)__ - Given a lock, check if the effective priority of the running thread is greater than the lock’s `holder`, and if so, set the priority of the holder to this thread’s effective priority. Additionally, if the holder is also waiting on a lock, we maintain the `waiting` list in sorted order by removing and re-inserting the holder’s TCB argument from the `waiter` list of the lock on which it’s waiting. We then recursively call `priority_donation` on that lock. There are two base cases: when the thread holding the lock is not waiting on another lock, and when the effective priority of the current thread is less than or equal to the lock’s holder.
### 2.2 Algorithms
**Choosing the next thread to run**  
We will maintain the `ready_list` as sorted by effective priority. Whenever choosing the next thread to run, the first thread in the ready list should have the highest priority and will be chosen by calling `list_pop_front()`. Just like the current implementation of `next_thread_to_run`, if the `ready_list` is empty, then it will choose `idle_thread`.
In `thread_yield()` we change `list_push_back()` to `list_insert_ordered()` for the `ready_list`.

**Acquiring a lock**  
If a thread attempts to acquire a lock but the lock is already held, then the thread calls `priority_donation()` on the lock and inserts itself into the `waiters` list in sorted order in `sema_down`. Otherwise, the lock is granted to the thread and we add that lock to the `owned_locks` list and set the `holder` of the lock to the current thread.

**Releasing a lock**  
When a thread releases a lock, it sets its `effective_priority` to its `base_priority` if it holds no more locks with donors. Otherwise, it iterates through its `owned_locks` list and sets its `effective_priority` to the highest `effective_priority` of its donors. To do this, for each lock, it looks at the first element in the lock’s `semaphore.waiting` list and sets its effective priority to the highest of these waiting TCB’s `effective_priority`, or its `base_priority` if that’s higher. We can correctly look at just the first element because we make sure the `waiting` list is sorted by priority. If there are other threads waiting on the lock, we simply look at the first TCB in the `waiting` list because it will already be sorted by priority. That one will be given the lock. (Just like the current implementation, this will actually be handled in `sema_up`.) Call `thread_yield()` to make sure the highest priority thread is run.

**Priority scheduling for semaphores and locks**  
Priority scheduling is implemented for locks and semaphores by keeping the `waiters` list in the semaphore struct sorted in descending order by priority. When a lock is released or a semaphore is upped, then the front element of the `waiters` list (with the highest priority) is removed and put on the ready queue. If any thread is added to the `waiters` list, we will use the given list insertion function that will insert the TCB in sorted order, maintaining the sorted property.

**Priority scheduling for condition variables**  
First, we add `priority` to `semaphore_elem`. Then, this can be used to keep the `waiters` list in the condition struct in descending order by priority. Since we have the list sorted, the original implementation of the condition variable’s `signal()` and `broadcast()` will enforce priority scheduling. In `wait()`, we change `list_push_back()` to `list_insert_ordered()`.

**Changing thread’s priority**  
In `set_priority()`, assign the `base_priority` to the argument. Set `effective_priority` to the max of the current effective priority and the base priority. Call `thread_yield()` after setting both priorities to make sure the highest priority thread is run.

**Edge cases:**  
Suppose a thread A has lock 1 with waiters B (priority 10) and C (priority 5). Thread C has lock 2. Now a new thread D (priority 11) attempts to acquire lock 2, which donates its priority to C. Now C has a higher priority than B, but in the sorted list the order is not maintained, since the priority changes the contents of the list internally after insertion. This is why in our solution we re-insert the thread with `list_insert_ordered()`, to maintain order in the `waiting` list even after internal modification.

### 2.3 Synchronization
For the synchronization, we consider each data structure or variable that can be shared between threads, the functions that will access those variables, and what we are doing with each function to prevent race conditions.
* `ready_list`
  - Case: modifications to `schedule()`, `thread_yield()`, `thread_unblock()`  
        - All of these functions disable interrupts, allowing the running thread to run until completion without being preempted or interrupted, so the access to `ready_list` is guaranteed to be atomic. `ready_list` is not directly changed in any other function.
- `struct semaphore.waiters`
   - Case: modifications to `sema_up()`, `sema_down()`  
         - Since interrupts are disabled in both the `sema_up()` and `sema_down()` functions, writes to `value` and calls to `thread_block()` and `thread_unblock()` are atomic.
   - Case: modifications to `lock_acquire()`, `lock_release()`, `priority_donation()`  
         - To prevent concurrent accesses to the `waiters` member, we disable interrupts before doing priority donation in `lock_acquire()` and before looping through the locks to find the current thread’s new `effective_priority` in `lock_release()`. We keep interrupts disabled when calling `sema_up` and `sema_down` so that no other changes will make the `effective_value` inconsistent.
- `struct condition.waiters`
   - Case: modifications to `wait()`  
         - Since the `waiters` list in the condition struct is shared data, and lists are not thread safe, we need to disable interrupts before accessing the list and re-enable them afterwards.
   - Case: modifications to `signal()`  
          - It is possible for a malicious thread to directly access and modify the condition struct’s `waiters` list, which could lead to a time of check to time of use error. To prevent this we disable interrupts before the check and re-enable them after the function completes.
   - Case: modifications to `broadcast()`  
         - To ensure that all waiting threads are unblocked we disable interrupts before iterating through the list and re-enable them after.
- `effective_priority`
   - Case: modifications to `schedule()`, `thread_yield()`, `thread_unblock()`, `lock_release()`, `priority_donation()`  
            - These functions have already been made thread safe for other shared variables, so the concurrent accesses are prevented for this shared resource as well.
   - Case: `thread_get_priority()`, `thread_set_priority()`  
         - Since multiple threads may request or set priority concurrently, we need to ensure the priority does not change during the function calls. To do this we disable interrupts.
- `waiting_on` and `owned_locks`  
Both of these resources are used specifically for priority donation, so they are specific to dealing with locks. Hence, they are not used outside of `acquire()` and `release()`.
    - Case: modifications to `acquire()`, `release()`  
            - These functions are the only instances where the resources are accessed, and we have made them thread safe previously for other shared resources, making concurrent accesses to the resources thread safe.

### 2.4 Rationale
Our reasoning for using a sorted list for both the `ready_list` and the `waiting` lists is to make common operations, such as `schedule()` and `release_lock()` more time efficient. This is because in a sorted list the highest priority thread is at the front of the list. This allows us to simply remove the front element of the list when we want to run the thread with the highest priority in `schedule()`. It also allows us to more quickly and efficiently identify which thread has the highest priority out of all the waiting threads in the current thread’s list of locks. This saves time in the `release_lock()` operation, which will run in O(m) time when m locks are held, as opposed to O(mn) time in an unsorted list, with n waiters in each lock. 

Another list we are adding is the `owned_locks` list in the TCB, but unlike the ones mentioned above, it is not sorted. Leaving this list unsorted is more efficient than keeping it sorted (on the holder’s priority) in several ways. Inserting locks into this list is considerably faster, since it simply adds the lock to the front instead of finding the right place to insert. It does not have any overhead of maintaining the order of the list, i.e. priority donation forces the list to re-insert the updated lock element. Lastly, we cannot determine which lock will be released first, so maintaining an order does not speed up any operations in particular.
 
In the `semaphore_elem` struct we add a priority member so we can keep the `waiters` list sorted in the `condition` struct. However, unlike the `semaphore`’s `waiters` list, the threads’ priorities in `semaphore_elem` cannot not change after they are added to the list. This is because the threads are blocked on a semaphore, which does not implement priority donation. Because of this we can simply use the `list_insert_ordered()` and `list_pop_front()` functions to maintain order in the list.


# Additional Questions
1. When we create a new thread, we allocate a 4 KB memory to store our TCB and kernel stack. The thread structure itself sits at the very bottom of the page (at offset 0) and will never grow. The rest of the page is reserved for the thread's kernel stack, which grows downward from the top of the page (at offset 4 kB). We initialize `kernel_thread_frame` to store our program counter, TCB to store stack pointer, and the kernel stack to store the register state.

2. The page containing a kernel thread’s stack and TCB is freed at the end of a call to `thread_schedule_tail()`, which is called in `schedule()`. We cannot call `palloc_free_page()` in `thread_exit()` because we still need data from the TCB in the `schedule()` function. This is why we need to call `schedule()` to switch to another thread and free the page of the previous thread if it’s dying. A thread needs to be freed by another thread and cannot free its own TCB. Additionally, we cannot call `palloc_free_page()` after the `schedule()` call because it will never be reached.

3. Since the `thread_tick` function is called by an interrupt handler, it is being handled by the kernel, so it uses the kernel stack.

4. The test case:
```
/*
 * Initial priorities:  
 * priority_donor: 4  
 * higher_effective_priority: 5  
 * higher_base_priority: 6   
 * broken_sema_up_call: 8  
 */

struct lock springBreak; // pintos lock. Already initialized.
struct lock isComing; // pintos lock. Already initialized.
struct Semaphore virus; // pintos Semaphore. Already initialized. Value = 0

void priority_donor() { 
    thread_set_priority(12); 
    lock_acquire(&isComing); 
    lock_acquire(&springBreak);
    lock_release(&isComing);
    lock_release(&springBreak); 
    thread_exit(); 
} 

void higher_effective_priority() {
    lock_acquire(&springBreak); 
    sema_down(&virus); 
    printf(“should print first”);
    lock_release(&springBreak); 
    sema_up(&virus); 
    thread_exit(); 
} 

void higher_base_priority() {
    sema_down(&virus); 
    printf(“should print second”); 
    sema_up(&virus); 
    thread_exit(); 
} 

void broken_sema_up_call() { 
    lock_acquire(&isComing); 
    while (thread_get_priority() < 11) { 
        timer_sleep(20); 
    } 
    lock_release(&isComing); 
    sema_up(&virus);
    thread_exit(); 
}
```
**How test works:**  
1. All of the functions are queued on the priority scheduler.
2. `broken_sema_up_call()` executes first because it has highest priority, which acquires the `isComing` lock and loops until its priority is greater than or equal to 11.
3. `higher_base_priority()` executes, which waits on the `virus` semaphore on a `sema_down()` call.
4. `higher_effective_priority()` executes, which acquires the `springBreak` lock and also waits on the `virus` semaphore.
5. `priority_donor()` executes, which sets its priority to 12 and then waits on the `isComing` lock, donating its priority (12) to `broken_sema_up_call()`.
6. `broken_sema_up_call()` exits the while loop since its priority is greater than 11. It then releases the `springBreak` lock.
7. Since `broken_sema_up_call()` released the lock, it lost the donated priority from `priority_donor()` and got put back to sleep.
8. `priority_donor()` now has the highest priority, but it is waiting on `isComing` lock, donating its priority (12) to `higher_effective_priority()`.
9. `broken_sema_up_call()` is now the running thread because all other threads are in the waiting queue. `broken_sema_up_call()` calls `sema_up()` on virus semaphore.
10.Since the semaphore is upped, the thread with the highest base priority is given the semaphore. If up chooses based on the base priority, `higher_base_priority()` is given the semaphore since its base priority is 6 while the other is 5. If up properly chooses based on effective priority, `higher_effective_priority()` should be given the semaphore instead since it has an effective priority of 12.
11. If `sema_up` is not properly implemented, then `higher_base_priority()` executes first, and prints “should print second,” and ups the `virus` semaphore, giving the semaphore to `higher_effective_priority()` thread, which prints “should print first”.
12. If `sema_up` is properly implemented, then `higher_effective_priority()` executes first, and prints “should print first,” and ups the `virus` semaphore, giving the semaphore to `higher_base_priority()` thread, which prints “should print second”.
13. The possible test outcomes vary based on how `sema_up` implements its priority selection, so the test highlights whether or not `sema_up` is using base or effective priority.

**Expected output:**  
should print first  
should print second

**Actual output on failure:**  
should print second  
should print first
