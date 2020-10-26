Design Document for Project 3: File Systems
===========================================

## Group Members

* Michael Duong mduong15@berkeley.edu
* Nicholas Saquilayan nsaquilayan@berkeley.edu
* Tobey Yang minchu.yang@berkeley.edu
* Sheng Lo shenglo1@berkeley.edu

# Task 1: Buffer Cache
### 1.1 Data Structures and Functions
__Data Structures:__
```
struct cache_entry {
	bool valid_bit;			// a bit that is set if this entry is still valid
	bool dirty_bit;			// a bit that is set if writes need to be pushed to disk
	bool ref_bit;			// a bit that is set if this entry was recently accessed
	block_sector_t sector;		// sector on disk this entry has data for
	void* data;			// a pointer to the data from disk copied to the heap
        struct lock entry_lock;		// lock for each cache entry
        struct condition entry_cond;	// condition variable for the cache entry monitor
}
```
```
struct cache {
	 struct cache_entry entries[64];	// the entries of the cache
         unsigned clock_hand;		        // index to `entries` for the clock algorithm
         struct lock cache_lock;	        // lock for the cache entry monitor
}
```
__Functions:__

__int get_cache_entry (block_sector_t sector)__ - Check if `sector` exists in the cache already. If it does, returns the cache entry index for the sector, else return -1.  

__int replace (block_sector_t new_sector)__ - First, it loops through the cache to find an entry where the `valid_bit` is 0 and places the `new_sector` into that entry. If no entry is found, evict the less recently used cache entry using the clock algorithm and replace it with a cache entry for `new_sector`. Also performs write back for the cache. Returns the index of the new entry.  

__void cache_read (block_sector_t sector, void *buffer, off_t offset, off_t size)__ - Read `sector` starting at `offset` for `size` bytes from `cache` into start of `buffer`. If `sector` is not in the `cache`, then `replace()` will be called to place the desired sector into the cache before the read.

__void cache_write (block_sector_t sector, const void *buffer, off_t offset, off_t size)__ - Write `sector` starting at `offset` for `size bytes` to `cache` from start of `buffer`. If `sector` is not in the `cache`, then `replace()` will be called to place the desired sector into the cache before the write.
### 1.2 Algorithms
Replace every instance of `block_read` and `block_write` in inode.c with `cache_read` and `cache_write` respectively. Also, every instance of `inode->data` needs to be replaced with a `cache_read` to the `inode->sector`.  

Initialize every cache entry `valid bit` to 0. In `cache_read()` and `cache_write()`, call the `get_cache_entry` function to check if the sector exists in the cache. If it does and the valid bit is set to 1, set the `ref_bit` to 1 and perform the read/write operation to the cache. If not, call `replace` to find the cache entry that is empty or should be evicted. If the evicted entry’s `dirty_bit` is set, then write the data back to disk and reset the `dirty_bit` to 0. Then replace it with the newly read data from disk and update the `cache_entry->sector`, `ref_bit` and `valid_bit`. After replacing it, perform the read and write operation on the data in the cache.  

On system shutdown which is handled by `filesys_done` in filesys.c, scan through the cache and write back any dirty blocks to the disk.  

In `inode_close`, if `inode->removed` is true then for each sector belonging to the inode, iterate through the cache and invalidate any entries that belong to that sector.

__Edge cases:__

Suppose a thread, A is waiting on the condition variable of a cache entry, since the entry and the desired file share the same sector number. It is possible that there is another thread, B, waiting on the condition variable that plans to evict the entry given that it was selected by the clock algorithm. After B evicts the entry and has finished performing its operations on the new entry, thread A gets the `entry_lock`, and expects the entry to be data from the sector number it searched the cache with. However, this is not the case because thread B replaced the contents of the cache entry with its own data. Our design handles this by adding an additional check to the monitor which checks if the sector number still matches. If so, then the thread can perform the operations it intended to. Otherwise, it has to perform a search of the cache again, which can lead to finding another matching cache entry or evicting another entry.

### 1.3 Synchronization
__struct cache_entry__  
- Any time a sector contained in a cache entry wants to be used, the `cache_entry->entry_lock` needs to be obtained. This means we will not allow concurrent reads.
- If the clock algorithm chooses a cache entry, the `cache_entry->entry_lock` has to be obtained first before eviction can occur. This prevents other processes from attempting to access the block during eviction, and it also prevents a block being used from being evicted.

__struct cache__  
- A `cache->cache_lock` has to be used for any activity involving the cache. This makes sure no block will attempt to access or load the same block until it is fully loaded into the cache. Additionally, we will use a Monitor to check for `cache_entry` availability. The cases for the `struct cache` are:
  - Looping through the cache to see if a sector is in the cache  
        __-__ When looping through all of the cache entries, we will have the `cache->cache_lock` acquired so that no other thread can be looping through to possibly do an eviction.  
        __-__ If no cache entry is found, then we move onto eviction (the bullet point below).  
        __-__ Otherwise, if there is a cache entry, we will use a `while` loop to call `lock_try_acquire` on the `cache_entry->entry_lock`. If the acquire fails, we enter the loop and call `cond_wait(cache_entry->entry_cond, cache->cache_lock)` to release the `cache_lock` for another thread. Once signalled, we call `lock_try_acquire` and loop again until we obtain the lock.  
        __-__ Once the lock is obtained, we have to check again that the `cache_entry->sector` is the sector we’re looking for (in case it got evicted). If it’s not the desired one, then we release the `cache_entry->entry_lock` and repeat the entire looping through the cache from the start to see if the entry is somewhere else.   
  - Evicting a cache entry (done in the `replace()` function)  
        __-__ While the clock algorithm is being run to find an entry, the `cache->cache_lock` will be acquired. When the entry is chosen, we will use a `while` loop to call `lock_try_acquire` on the `cache_entry->entry_lock`. If the acquire fails, we enter the loop and call `cond_wait(cache_entry->entry_cond, cache->cache_lock)` to release the `cache_lock` for another thread. Once signalled, we call `lock_try_acquire` and loop again until we obtain the lock. Once the lock is obtained, then we actually evict.  
  - Once we have the desired entry in the cache, we should also have the corresponding `cache_entry->entry_lock`, so we can then release the `cache->cache_lock` so that another read/write in a different sector can start concurrently.  
- Monitor signalling:  
  - At the end of `cache_read` and `cache_write`, once everything is done, the thread attempts to reacquire `cache->cache_lock`. Once it has the lock, it will then release its `cache_entry->entry_lock`, signal a thread to wake up using `cache_entry->entry_cond`, and then release the `cache->cache_lock`.
  
The space used for this synchronization is O(n) since there is one lock per `cache_entry`, but this allows for more parallelism because this allows for each `cache_entry` to be utilized in parallel. The only time there will be blocking is if multiple threads are trying to access the same sector, in which case blocking should occur to maintain correctness, or a block chosen by the clock algorithm for eviction just happens to be in use, which should not occur as often due to the LRU approximation of the clock algorithm. By using a monitor and calling `cond_wait`, this allows for independent sectors to still be accessed even when multiple threads may be waiting on the same sector. 

### 1.4 Rationale
We decided to implement our cache as a fully associative cache because data can be placed in any unused blocks when data is fetched from the disk. Because of this, the cache will never have a conflict between memory addresses that maps to the same cache entry. Moreover, it will have a high hit rate when we repeat access to the same memory or when the cache is nearly full. While the direct-mapped cache has faster access time for its entries, a file system prioritizes hit rate over performance, which makes a better case for a fully associative cache.

For our replacement algorithm, we decided to use the NRU (clock) algorithm as our implementation. Although LRU will be ideal in a way that it evicts the least recently used one, it’s too expensive to achieve. Thus, we use the clock algorithm which approximates LRU. The `ref_bit` partitions the cache into 2 groups, one for the recently used, the other for the not recently used. Whenever a block in the cache needs to be evicted, the clock algorithm advances the clock hand to either find the not recently used block to replace or set the recently used block to not recently used. This also is fairly simple to implement when writing the code for the algorithm.
# Task 2: Extensible Files
### 2.1 Data Structures and Functions
__Data Structures:__
```
struct inode_disk {
	...
	/* data blocks pointers */
	block_sector_t directs[124];
        block_sector_t indirect;		
        block_sector_t doubly_indirect;
}
```
```
struct inode {
	...
	struct lock inode_lock;
}
```
```
/* variables */
struct lock open_inodes_lock;
struct lock freemap_lock;
```
__Functions:__

__block_sector_t byte_to_sector (const struct inode *inode, off_t pos, bool extend_eof)__ - Returns the `block_sector_t` mapping to the pos in the inode. If `extend_eof` is true and pos is beyond EOF, the function will add the sectors in between the EOF and pos and fill it with zeros.  

__int inumber (int fd)__ - Gets the file descriptor table for the current thread to access the file struct given `fd` and returns the inode sector as the unique inode identifier.
### 2.2 Algorithms
The main function for this Task is `byte_to_sector`. Since this function’s job is to take a position and return the corresponding sector for the inode based on that position, this is where file extensibility comes in.  
To start, as mentioned in Task 1, we need to load the inode’s corresponding `inode_disk` from cache, so we call `cache_read` on sector `inode->sector`.   
Then, we check to see if the `pos` is beyond the file’s EOF using the `inode_disk->length`. If it is beyond and `extend_eof` is `false`, then we return -1.  

Else if `extend_eof` is `true`, we do the work of extending the file as follows:  
1. We calculate how many sectors need to be added, accounting for the fact that there may still be some remaining space in the current last sector.  
2. Use `free_map_allocate` to allocate the number of sectors needed to extend this file.
3. To do the mapping, we first think of the file’s memory as contiguous. We can then get a `contiguous_idx` for the end of the file by doing `contiguous_idx = inode_disk->length / BLOCK_SECTOR_SIZE`. This would be the index of the sector if the memory was contiguous. Thinking in this fashion, we start adding sectors to the end of the file. The next sector to add would be `contiguous_idx + 1` and the following is `contiguous_idx + 2` and so forth.
4. Loop through and assign each of the allocated sectors to the correct pointer in `inode_disk`. If we take the `contiguous_idx` for each sector to add, we use that index to get the correct pointer with the following scheme:  
    - 0 <= contiguous_idx < 124  
          - We use the direct pointers and assign the new sectors to `directs[contiguous_idx]`  
    - 124 <= contiguous_idx < (124 + 128)  
          - We use the (singly) indirect pointers. If `contiguous_idx == 124`, we need to allocate a block of all zeros and assign `indirect` to it.   
          - To access the actual pointers, we first do a `cache_read` on `indirect` and fill a buffer of `block_sector_t indirect_sectors[128]`.   
          - Then, we assign the new sector to `indirect_sectors[contiguous_idx - 124]`.  
    - (124 + 128) <= contiguous_idx  
          - For all larger indices, we use doubly indirect pointers. Similar to the singly, if `contiguous_idx == (124 + 128)`, we need to allocate a block of all zeros and assign `doubly_indirect` to it.   
          - To access the actual pointers, we first do a `cache_read` on `doubly_indrect` with a buffer of `block_sector_t doubly[128]`.   
          - We then do another `cache_read` on `doubly[((contiguous_idx - 124) /  128 ) - 1]` with a buffer of `block_sector_t singly[128]` to get the singly indirect corresponding to the index if the entry exists. If it doesn't exist, we  need to allocate a block for it and assign the entry to the new block.   
          - Then, we can finally assign the new sectors to `singly[(contiguous_idx - 124) % 128]`   
5. If we reach disk space exhaustion and are unable to allocate new blocks, then we return -1, and any code that calls `byte_to_sector` will have to do an error check to avoid doing any operation if memory can’t be allocated. In `inode_read_at` and `inode_write_at`, we immediately return `bytes_read` and `bytes_written` respectively to let the user know what has already been read/written, and it also prevents any other operations from occurring.

At this point, we know that the sectors up to `pos` for the file will exist. We then calculate the contiguous index for the position with `pos_idx = pos / BLOCK_SECTOR_SIZE`. Knowing where the index lies if the file was contiguous, we can then find and return the value of the true `block_sector_t` using the accessing scheme described in step 4 of the extension algorithm above (ignoring the parts about assignment).

__Edge cases:__
- Suppose a user tries to write or read to a file that is past the file’s length, but the offset is within the same sector. In that case, we detect that the number of sectors to be added is 0, so nothing is done for extension.  
- If `inode_close` is called on the `free_map_file` and the file is to be removed, then we do not call `free_map_release` and instead only clear the sectors from the cache by making the `valid_bit = 0`. The reason for this is that in `free_map_release`, it will call `bitmap_write`, and that will attempt to acquire the inode lock again and cause errors. This is acceptable because since the `free_map_file` is being deleted from disk, there is no need to write to the `free_map` to update it.
### 2.3 Synchronization
__struct inode__  
- Inodes are accessed concurrently whenever a file access operation is being performed. To ensure correctness, we added a lock member to each inode. This ensures that only one file is worked on by a single thread at a time.
- Correctness: The inode lock is acquired at the beginning of the file operation and released at the end of it, so a process can only own one inode lock at a time. Because of this there is no deadlock possible between inode locks.
- Time and space costs: Because the granularity of the lock is per-file, this results in slower performance for reading and writing to the same file than if the granularity was per-block. It also adds O(n) space where n is the number of inodes in memory since each inode has to allocate space for a lock struct.
- Frequency: The number of threads contending for each inode lock depends on the popularity of the file the inode describes, and also the frequency at which the user is accessing files.  

__open_inodes__  
- `open_inodes` is accessed whenever a thread opens a new file or is the last thread to close a file. Since Pintos list operations are not thread-safe, we need to ensure that multiple threads can open and close files asynchronously.
- Correctness  
  - Consider a scenario where a thread A acquires an `inode_lock` in `inode_close` and, before it is able to acquire the `open_inodes` lock to remove it from the list, is preempted by thread B which acquires the `open_inodes` lock. Thread B then attempts to acquire the `inode_lock` held by A on `inode_reopen`, but waits on A. Then A resumes, and attempts to acquire the `open_inodes` lock but waits on B, which results in deadlock.  
  - To avoid this we have threads acquire the `open_inodes` lock before acquiring any `inode_lock` in `inode_open` and `inode_close` to ensure correctness.  
- Time and space costs: Serializing list operations may slightly slow down requests to open files since it involves iterating through the `open_inodes` list and checking if an inode already exists for that sector. For space this is a single lock, so constant memory is added.  
- The frequency of threads contending for the `open_inodes` lock depends on the frequency of calls to `inode_open` and `inode_close`.  
- The only time a thread will have the `open_inodes` lock and an `inode_lock` at the same time is in `inode_reopen`.  

__free_map__  
- In free-map.c, there is a bitmap `free_map` that gets modified in two places: `free_map_allocate` and `free_map_release`. Since the `free_map` dictates what changes are written to the `free_map_file` in memory we need to ensure that it is thread safe to maintain correctness. We avoid using this lock for `bitmap_read` and `bitmap_write` because the `inode_lock` for the `free_map_file` will take care of synchronization.  
- Correctness: To do this we added a lock around any modifications to `free_map`. The goal of this lock is to serialize changes made to `free_map`. Consider two threads A and B which want to write to `free_map`. Without the lock it is possible that A and B both write to it simultaneously and one overwrites the other’s changes. With the lock, the most recent changes are always written back to the `free_map_file`.  
- Time and space costs: The time cost does not heavily affect the overall runtime as locks only surround a single call to bitmap operations. Additionally, since there is only one lock the space consumed is constant.
- The frequency of threads contending for the `free_map` lock is likely low since it only changes whenever files are added or removed from disk.
### 2.4 Rationale
The current implementation relies on contiguous free blocks to be able to extend files. However, it can’t be guaranteed that there’s always free blocks that come after the original one. Thus, our design should not be restricted by using contiguous blocks. Both the FAT and FFS system use pointers to allow data in files to reside in noncontiguous blocks for extensibility. However, the FAT system makes random access to data blocks inefficient because it requires traversing through the data blocks by following the pointers. Eventually, we decided to use direct, indirect and doubly-indirect pointers like the Unix FFS file system to allow file extensibility. That way, the data block can be retrieved by converting the bytes to the correct pointer using mathematical calculation. Since 128 pointers can fit in each data block, we choose to use one indirect pointer which could hold up to 64KB of file data and another doubly indirect pointer that can hold up to 8MB to satisfy the 8MB file size requirement. The rest are reserved for direct pointers for faster access if the file is small.

When a user writes data far beyond EOF, our system will allocate the data blocks in between and zeroed them out. This will be easier to implement and will make accessing sectors more efficient because we do not need to check every sector that is getting accessed to see if it has been allocated or not. 
# Task 3: Subdirectories
### 3.1 Data Structures and Functions
__Data structures:__
```
struct fdt_entry {
	bool is_dir;		// True if the file is a directory, false otherwise
	void *entry;		// A pointer to the actual file 
}
```
```
struct tcreate_arg {
	struct dir *cwd;	// The current working directory of the parent
				// process for the child to inherit
}
```
```
struct thread {
	...
	struct dir *cwd;	// The current working directory of this thread
}
```
```
struct inode_disk {
	...
	bool is_dir;		// True if the file described by this inode is
				// a directory, false otherwise
}
```
```
/* variables */
struct lock dir_lock 		// Lock that makes lookup operations followed by
				// inode operations atomic in directory.c
struct fdt_entry** fdt   	// We modify our fdt from “struct file** ” to “struct fdt_entry** ” 
				// to account for both file struct and dir struct can be in our fdt
```

__Functions:__

__inode_create (block_sector_t sector, off_t length, bool is_dir)__ - Overloaded version of `inode_create` that, on creation of an `struct inode_disk`, specifies whether the file in `sector` is a directory or not. This is done by assigning the `is_dir` data member of the `struct inode_disk` to the argument `is_dir`.  

__static int get_next_part(char part[NAME_MAX + 1], const char **srcp)__ - Extracts a file name from `*srcp` into `part`, and updates `*srcp`  so that the next call will return the next file name part. Returns 1 if successful, 0 at the end of string, -1 for a too-long file name part.

__struct dir* get_dir_from_path(const char *path, char *last_arg)__ - Returns the parent directory of the `last_arg` (`last_arg` can be a directory or a file). `path` can be an absolute path or a relative path. The last argument of `path`, i.e. the file or directory name following the last slash, is stored in `last_arg`.

__bool chdir (const char *dir)__ - Uses `get_dir_from_path` to get the parent path. With `last_arg`, it checks that it’s a directory of the parent path and sets `cwd` to that directory if it is. Returns true if successful, false otherwise.

__bool mkdir (const char *dir)__ - Uses `get_dir_from_path` to get the parent path. With `last_arg`, it creates a directory in the parent directory and sets `.` and `..` as the first entries. Returns true if successful, false otherwise.  

__bool readdir (int fd, char *name)__ - We will use the file descriptor `fd` to get the `fdt_entry` from our fdt. Then we check if the `fdt_entry` struct is a valid directory. If so, store the filename in name and return ture; otherwise, return false.

__bool isdir (int fd)__ - Go to entry in fdt and return true if it is a directory, false otherwise.

__int open (const char *file)__ - Change the signature of `filesys_open` to return a `struct fdt_entry*`.

__void close (int fd)__ - Check if file described by `fd` is a dir (use `dir_close`) or a file (use `file_close`).

__pid_t exec (const char *file)__ - Pass `thread_current()->cwd` as an argument to `start_process`, so the child’s `cwd` member is set in `start_process`.  
In `load()` in process.c check if file is a directory after null check following call to `filesys_open()`. If it is a directory, print an error message and goto done. 

__bool remove (const char *file)__ - We check if the `file` is a directory. If yes, then obtain the `inode_lock` and then check `open_cnt`. We check the `open_cnt` and   
- If non-zero, don’t remove the directory. __This means that a user cannot delete a directory if it is a cwd of a running process__.
- Else make sure that all entries after `.` and `..` have `in_use` as false. If so, remove the directory.

If it’s not a directory, then we just remove the file.

__bool create(const char *file, unsigned initial size)__ - Change to allow for relative path resolution.

__int read (int fd, void *buffer, unsigned size)__ - Check if the result of `fd_to_file` is a directory, if so then return -1.

__int write (int fd, const void *buffer, unsigned size)__ - Check if the result of `fd_to_file` is a directory, if so then return -1.

__int inumber (int fd)__ - We get the `fdt_entry` from fdt by using the `fd` as in index to the table. Then use `is_dir` on `fdt_entry` to cast the struct `fdt_entry*` to a `struct dir*` or `struct file*`. Return that pointer’s `inode->sector`.

### 3.2 Algorithms
To support both relative path and root path, every function that takes in a path name should call `get_dir_from_path` to get the directory that it is intended to operate on. To accomplish this, first check whether the path ends in `/`, if so return NULL. If not, we check if the first character is a `/`, and if it is, we start the search at the root. If it’s not, we start the search at the current thread’s `cwd`. Then a while loop is run on `get_next_part` to get the second tokenized path until there’s no further path to be parsed. In the while loop, call `dir_lookup` to see if the first tokenized path is valid and whether it’s a directory, if so close the previous directory, open the new one, update the parsed directory pointer, set the first tokenized path to the second and continue through the loop. If it’s not a valid directory, update the directory to null. In the end, check if the directory pointer is null, if not copy the second tokenized path to `last_arg`. Finally return directory pointer.

To support `.` and `..` directories, we will have one `dir_entry` for each of  `.` and `..` inside a directory. That way the two special notions would be treated like a normal `dir_entry`. Wherever a user wants to create a new directory, we first use `get_dir_from_path` to get the parent directory and the new directory name. Then use `free_map_allocate` to get a sector and call `dir_create` on the sector. Then call `inode_open` on the sector to get the pointer to the inode, call `dir_open` on the inode which gives a pointer to the directory. Next, call `dir_add` to add both `.` and `..` to the newly created directory. For the `.` sector we can pass in the sector returned from `free_map_allocate`, and for the `..` sector we would pass in the `parent_dir->inode->sector` where `parent_dir` is the `struct dir*` returned from `get_dir_from_path`. Finally, add the new directory to the parent directory.

__Edge cases:__

- For `chdir`, if the argument is just `/`, then we just set the `cwd` to the root. This is the only time something ending with a `/` is considered valid. We do not need to account for this in `get_dir_from_path` because no other operation is allowed to have just the root as an argument.
- In `do_format` in `filesys.c`, this is where the root directory is created. After creation, in that function, we open the root directory and then add the root as `.` and `..` entries for itself. After finishing, we close the directory.

### 3.3 Synchronization
For synchronization, anything in the lower levels of reading and writing to the file representing the directory is already taken care of by the synchronization done in inode. 

Above that though, we need to ensure that changes to a directory are safe. We notice that a common pattern is reading in from inode to lookup something, doing some action, and then writing back to the inode. However, in the time from reading in to writing back, things can change, so we need a lock around those critical sections. These occur in:
- `dir_lookup`
- `dir_add`
- `dir_remove`

Each process would have its own `struct dir *`, so we cannot place the lock inside the struct. Since we have no way of sharing a lock for a specific directory, we decided to use a global lock for the entire `directory.c` file. 
- Correctness: This ensures that any operation that modifies a directory, like adding or removing, will run to completion, so a directory file cannot get corrupted.
- Time and space costs: The time of this is O(1) since it is just a lock acquire and release, and the space is O(1) since there is just one global lock for the entire directory. This will limit directory operations, but does not limit the parallelism of file operations.
- The frequency of this will depend on how often directory operations occur. Most programs work on files and directories that are already present, so only directory and file creations/removals will be affected, which should not be too often.

### 3.4 Rationale
For the `get_dir_from_path` function we decided to return the parent of the last argument instead of writing another function to split the path string. Initially we intended to have `get_dir_from_path` return the last directory in the argument string, and to leave the rest of the functionality involving the last argument to the syscall. However, we reasoned that this would result in extra time being taken up to traverse the path string only to traverse the individual parts of the string anyways when looking up each directory. Additionally, most syscalls do something special with the last directory in the argument string, so we felt that it was useful to do the parsing in `get_dir_from_path`.

We decided to incorporate the `.` and `..` directories in as directory entries after considering the tradeoffs between having entries or each directory having a pointer to its parent. Without the additional `struct dir_entry`s in each directory, the only way to access the parent directory would be to access the parent pointer in the `struct inode_disk`, which requires special cases to handle arguments that have `.` and `..` as well as more function calls just to open the parent directory from the pointer. By having them as entries, that creates uniformity in how directories are stored, without special cases for `.` and `..`, and it ensures that users cannot create directories or files with those names.

# Additional Questions
 
We can implement read ahead by caching a fixed number of sequential blocks following the desired block into the cache instead of only one whenever a sector has to be retrieved from disk. The number of extra blocks should be small, since a large number of evictions could result in lower hit rate and worse performance. Because of our cache having only 64 entries, we think that one extra block (meaning 2 total blocks per disk fetch) would be good. Due to the principle of spatial locality, we choose the next block to be the following sector in the inode's pointer structure, if it exists. When the cache is full we evict however many entries necessary to cache the desired block and the extra sequential blocks. To make the second block fetch in the background, whenever we find a cache miss, we could create a new thread using `thread_create()` and have that new thread just fetch the second block. Once the block is fetched, we just terminate the thread. This way, the first thread can continue once its desired block is fetched and the second fetch is done asynchronously.  

For write behind, we can create a new thread that periodically looks for the dirty data in cache and writes them back to disk. To achieve this, `timer_sleep` should be called to put the thread to sleep after write operations are done so that it’s not busy waiting. Every time this thread wakes up, it repeats the operation to write dirty data back and set the dirty bit to 0. Although the write operations are performed, the cached data should not be evicted.
