Final Report for Project 3: File Systems
========================================
## Changes made from initial design document
### Task 1
The type of eviction policy we originally were going to implement was a Clock approximation of LRU, but during our review session we learned it would be more efficient and easier to implement actual LRU. This is because reads and writes to files are not as often as a TLB, so LRU would not be a huge loss to performance.

Our original design was that `get_cache_entry` was used to return a sector number, and `replace` would be called by `cache_read` and `cache_write` to do the replacement. Instead, we decided to simplify `cache_read` and `cache_write` by having `get_cache_entry` do the work of calling `replace` for cache misses, meaning that reads and writes only need to call `get_cache_entry` and that will make sure the entry is in the cache for reading/writing.

For synchronization we originally planned to keep a monitor for the cache, but this approach turned out to be too complicated. Instead we used double check locking by suggestion of our TA, Neil. This was much easier to implement and simpler to understand and reason its correctness.

### Task 2
We removed the `inode_lock` in `struct inode` which was originally used to serialize all read and writes to a given inode since it did not follow the project specification. We replaced it with a `resize_lock` and a `metadata_lock`. The `resize_lock` prevents multiple threads from extending the same file concurrently. This is done by protecting the `length` member of the `struct inode_disk` until the end of the write operation. The `metadata_lock` serves a dual purpose. Firstly, it prevents multiple threads from accessing an inode’s metadata concurrently. Secondly, it serves as the lock to a monitor surrounding `inode_deny_write` and `inode_write_at`. We introduced a condition variable dependent on `no_writers` to ensure that threads that have started writing before the `inode_deny_write` function is called finish writing, but new threads cannot write.   

Originally we were going to perform resizing in `byte_to_sector`, but we found that the task was too complex to have in one function. We also realized that it would be better to refactor resizing into its own function that is called specifically in `inode_write_at` and `inode_create` since these are the only instances where resizing occurs. During task 3, we found a bug forcing us to relocate some of the functionality from `inode_resize_check` to `inode_write_at` since we were assigning the length of the `struct inode` before the write was done to the extended blocks, allowing for reads before the writes finished.

### Task 3
In our design doc we made the assumption that `chdir` is the only function that can have the root directory as a valid argument. We found that `filesys_open` also has this edge case, so we added the same check for root before calling `get_dir_from_path`.

Initially we intended to make the lock for directory operations global in the directory.c file, but we found out this wasn’t allowed in our design review. We relocated this lock to the `struct inode` which is only allowed to be acquired by inodes referring to directories. We changed our synchronization to the granularity of specific directories rather than any directory operation.


## Reflection
An improvement we made on this project from previous projects is the ability to parallelize our work. Although we still mainly utilized mob programming, and that helped to make sure everyone was on the same page, when we came to an agreement that a change needed to be made, we were able to split up the work to make that change. Because of mob programming, we all knew the codebase pretty well and what’s necessary to change it. For instance, when we needed to add syscalls for the test cases, we knew what files needed to be changed and all just started working on the necessary parts, getting the change fully completed sooner.

Another improvement we made for this project that went well was working around the synchronization before implementing the algorithms portion of a task. This was especially helpful for task 2, since there are a lot of places where good synchronization needs to be enforced to prevent any heisenbugs. It helped tremendously to think of synchronization without having the complexity of the algorithms distract us, and then just add the algorithmic implementation above the already reasoned synchronization.

Something that could have been improved was sticking to the roles of mob programming a little more strictly. Although the parallelization definitely helped, we kind of all became both pilots and drivers. What we needed was someone who had the role of looking up documentation and references online because whenever an issue came up, we all kind of scrambled to find it out rather than one person being ready to search it up. Having one person in charge might have helped for us to find solutions more quickly. In the end, each group member played an equal role in the project and we all pretty much did the work together, taking on the same roles of designers, programmers, and debuggers. We felt that our group did a great job of staying organized throughout the entire project, and we all agree that we felt like we tackled this huge codebase and project in an organized, effective manner.


## Student Testing Report
### cache-no-read

__Description:__ The cache-no-read test writes a new 200 block file and checks that there are no extraneous read operations performed.  
__Overview:__ We do this by creating an empty file and then flush the cache to make sure that everything is evicted. Then we write 256 bytes at a time until we reach 100KB to make sure that newly allocated blocks from file extension do not get flushed out of the cache before being written to. Once all writes have finished, flush the cache to make sure entries still in the cache are flushed to disk. Finally, check that the `write_cnt` is greater than or equal to 200 and that the `read_cnt` is less than or equal to 2. The `read_cnt` can be 2 at most since it includes reading in the free map file and the file’s metadata since it was flushed out of the cache before the test.
- Bug 1: If your kernel handles cache writes to existing blocks and newly allocated blocks (from extending or creating a file) the same way instead of treating them separately, then the test case would output more than 2 reads and fail. More than 2 reads would occur because on write for existing blocks, the `block_read` function gets called to make sure existing data on the disk is put in the cache before being written to. A proper implementation should know that if it’s a new block, `block_read` doesn’t need to be called because there is no data to retrieve from the disk. 

- Bug 2: If your kernel implements file extension such that blocks are allocated for a file in the metadata but is not read into the cache, then on each new block write, it would require a read before the write. In this case, the number of writes would be the same, but there would be many more `block_read`s. A proper implementation should set up a cache entry for the newly extended block without doing a `block_read`. 

__cache-no-read.output__  
Copying tests/filesys/extended/cache-no-read to scratch partition...  
Copying tests/filesys/extended/tar to scratch partition...  
qemu-system-i386 -device isa-debug-exit -hda /tmp/wrCVjtD0Ge.dsk -hdb tmp.dsk -m 4 -net none -nographic -monitor null
PiLo hda1  
Loading............  
Kernel command line: -q -f extract run cache-no-read. 
Pintos booting with 3,968 kB RAM...  
367 pages available in kernel pool.  
367 pages available in user pool.  
Calibrating timer...  117,350,400 loops/s.  
hda: 1,008 sectors (504 kB), model "QM00001", serial "QEMU HARDDISK". 
hda1: 200 sectors (100 kB), Pintos OS kernel (20)  
hda2: 241 sectors (120 kB), Pintos scratch (22)  
hdb: 5,040 sectors (2 MB), model "QM00002", serial "QEMU HARDDISK"  
hdb1: 4,096 sectors (2 MB), Pintos file system (21)  
filesys: using hdb1   
scratch: using hda2  
Formatting file system...done.  
Boot complete.  
Extracting ustar archive from scratch device into file system...  
Putting 'cache-no-read' into the file system...  
Putting 'tar' into the file system...  
Erasing ustar archive...  
Executing 'cache-no-read':  
(cache-no-read) begin. 
(cache-no-read) create "newfile"  
(cache-no-read) open "newfile"   
(cache-no-read) write_cnt: 203   
(cache-no-read) read_cnt: 2   
(cache-no-read) end   
cache-no-read: exit(0)   
Execution of 'cache-no-read' complete.   
Timer: 177 ticks   
Thread: 102 idle ticks, 66 kernel ticks, 9 user ticks   
hdb1 (filesys): 284 reads, 691 writes   
hda2 (scratch): 240 reads, 2 writes   
Console: 1149 characters output   
Keyboard: 0 keys pressed   
Exception: 0 page faults   
Powering off...   

__cache-no-read.result__  
PASS   


### cache-hit-rate
__Description:__ This tests that the hit rate of the buffer cache performs better the second time reading a file than the first time when it is cold.  
__Overview:__ We do this by creating a new file, writing random bytes into it, and flushing the cache to disk. Next, we reset the counters `num_hits` and `total_calls` in the `struct cache` to 0 so we can calculate the hit rate for the first read. After reading the entire file, calculate the hit rate and close the file. The file is reopened and we recalculate the hit rate following the same steps as before, starting with the reset. Lastly we check if the second hit rate is higher than the first hit rate. The output should say that the hit rate improved.

- Bug 1: If your kernel invalidated dirty entries during a cache flush instead of only writing them back to disk and resetting the dirty bit, then the test would output that the hit rate did not improve. In this case the hit rate would be exactly the same as a cold cache. After reading the entirety of the file the first time, flushing the cache would effectively lead to an empty cache since none of the entries are valid. Since none of the entries are valid, no cache hits can occur, just like in a cold cache. 

- Bug 2: If your kernel invalidates cache entries relating to a file once it is closed instead of keeping them in the cache, then the test would output that the cache rate did not improve. This is because the cache, after closing the file, is similar to a cold cache in regards to the file’s sectors. Only one file is being written to in this test case, so all of the entries in the cache that refer to the file are invalidated. Cache entries for a file should only be invalidated when the file is removed, not just when it’s closed. 

__cache-hit-rate.output__  
Copying tests/filesys/extended/cache-hit-rate to scratch partition...   
Copying tests/filesys/extended/tar to scratch partition...   
qemu-system-i386 -device isa-debug-exit -hda /tmp/m4r7oc3Zl1.dsk -hdb tmp.dsk -m 4 -net none -nographic -monitor null    
PiLo hda1   
Loading............   
Kernel command line: -q -f extract run cache-hit-rate   
Pintos booting with 3,968 kB RAM...   
367 pages available in kernel pool.   
367 pages available in user pool.   
Calibrating timer...  212,172,800 loops/s.   
hda: 1,008 sectors (504 kB), model "QM00001", serial "QEMU HARDDISK"   
hda1: 200 sectors (100 kB), Pintos OS kernel (20)   
hda2: 243 sectors (121 kB), Pintos scratch (22)   
hdb: 5,040 sectors (2 MB), model "QM00002", serial "QEMU HARDDISK"   
hdb1: 4,096 sectors (2 MB), Pintos file system (21)   
filesys: using hdb1   
scratch: using hda2   
Formatting file system...done.   
Boot complete.   
Extracting ustar archive from scratch device into file system...   
Putting 'cache-hit-rate' into the file system...   
Putting 'tar' into the file system...   
Erasing ustar archive...   
Executing 'cache-hit-rate':   
(cache-hit-rate) begin   
(cache-hit-rate) create "newfile"   
(cache-hit-rate) open "newfile"   
(cache-hit-rate) open "newfile"   
(cache-hit-rate) open "newfile"      
(cache-hit-rate) hit rate improved   
(cache-hit-rate) end   
cache-hit-rate: exit(0)   
Execution of 'cache-hit-rate' complete.   
Timer: 132 ticks   
Thread: 63 idle ticks, 66 kernel ticks, 3 user ticks   
hdb1 (filesys): 297 reads, 502 writes   
hda2 (scratch): 242 reads, 2 writes   
Console: 1197 characters output   
Keyboard: 0 keys pressed   
Exception: 0 page faults   
Powering off...   

__cache-hit-rate.result__     
PASS   

## Tell us about your experience writing tests for Pintos. What can be improved about the Pintos testing system? (There’s a lot of room for improvement.) What did you learn from writing test cases? 

Something that can be improved is the ability to have kernel programs. While writing tests for Pintos, we found that in order to implement the tests we had to create new syscalls. Some of the syscalls we created should not be available to the user, such as flushing the buffer cache, it would be nicer if there was a way to test things without having to create new syscalls. Something we learned is that test cases can become part of the code rather than being its own separate entity, requiring changes spread throughout the codebase just to keep track of details necessary for tests.
