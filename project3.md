Design Document for Project 3: File Systems
======================================

## Group Members

* Ahmed Ali  <ahmedali89@berkeley.edu>
* Denny Hung <dennyhl@berkeley.edu>
* Khang Kieu <ankhangkieu@berkeley.edu>
* Kin Seng Chau <elviskschau@berkeley.edu>

## Design Document:


### Task 1: Buffer Cache

#### Data Structures and Functions
- The global lock used in Project 2 to synchronize file system operations is removed.
- We create a new file (and its corresponding header) in `filesys` directory, `cache.c` and `cache.h` to contain the implementation of the buffer cache.
- In `cache.c`, we add the following:
```
    struct cache_block {
        uint8_t data[BLOCK_SECTOR_SIZE];   /* Contains the actual block to be cached. */
        block_sector_t sector;             /* Disk sector this block belongs to. */
        int ref_count;                     /* Number of threads currently using this block. */
        bool used;                         /* Use-bit for clock algorithm */
        bool dirty;                        /* 1 if block was modified ever since brought from cache, 0 otherwise. */
        bool valid;                        /* 1 if this block is a valid block, 0 otherwise. */
        struct lock block_lock;            /* Lock to be acquired before changing entry's metadata.
    };
    
    struct cache_block cache[NUM_SECTORS];         /* Cache containing a maximum of 64 cache blocks. */
    struct lock cache_lock;               /* Synchronizes block accesses when loading and evicting blocks. */
    static int hand;                      /*  Clock Hand */
```

- In `cache.h`, we have the following API operations:
```
    - void cache_init() /* Initializes the cache and its entries. */`
    - struct cache_block * find_block(block_sector_t sector); /* Finds cache block associated with the disk sector. If not in cache, fetches it from disk. */
    - size_t cache_read(block_sector_t sector, char * buf, size_t size, size_t off); /* Reads SIZE bytes from the cache block into BUF starting at OFFset. */
    - size_t cache_write(block_sector_t sector, char * buf, size_t size, size_t off); /* Writes SIZE bytes to the cache block starting at OFFset from BUF. */
    - void cache_write_back(struct cache_block *block); /* Writes back block of data to the disk. */
``` 
 

#### Algorithms
- Function `void cache_init()`:
    - Initialize the array of cache_block called `cache` with empty entries (valid, use, dirty bits set to 0), call `lock_init(&block_lock)` on each `cache_block` entry.
    - Call lock_init(&cache_lock).
    - Set the hand to 0.

- Function `struct cache_block * find_block(block_sector_t sector)`:
    - Acquire the global lock `cache_lock`.
    - Iterate through the array `cache` and find the `cache_block` that has the corresponding `block_sector`.
    - If the right `cache_block` is found:
        - Release the lock `cache_lock`.
        - Return the pointer to the current `cache_block`.
    - If no `cache_block` is found, meaning it is a cache miss. We need to loop through the array of `cache_block` and run our eviction policy (Clock algorithm), which does the following:
        - If the `valid` bit is 0, meaning we have an empty space:
            - Take a new block from memory and create a new `cache_block` to store this block of memory.
            - Increase the value of `hand`.
            - Break from the loop.
        - If the `valid` bit is 1 and the `used` bit is 1, meaning the current `cache_block` was recently accessed:
            - If the `ref_count` is greater than 0, meaning some other processes are using the current `cache_block`, we increase the `hand` and continue the loop.
            - Otherwise, we set the `used` bit to 0, increase the value of `hand`, and continue the loop.
        - If the `valid` bit is 1 and the `used` bit is 0, meaning the current `cache_block` is an old cache space:
            - Write the data to memory if the `dirty` bit is 1.
            - Take a new block from memory and create a new `cache_block` to store this block of memory.
            - Increase the value of `hand`.
            - Break from the loop.
        - Release the lock `cache_lock`.
        - Return the pointer to the `cache_block`.

- Function `size_t cache_read(block_sector_t sector, char * buf, size_t size)`:
    - Call `find_block(sector)` and set it to cache_block `current`.
    - Acquire the lock `entry_lock` of `current`.
    - Set the `valid` and `used` bits of `current` to 1.
    - Copy the `data` from `current` to `buf`.
    - Release `entry_lock` from `current`.

- Function `size_t cache_write(block_sector_t sector, char * buf, size_t size)`:
    - Call `find_block(sector)` and set it to cache_block `current`.
    - Acquire the lock `entry_lock` of `current`.
    - Set the `valid`, `used` and `dirty` bits of `current` to 1.
    - Copy the data from `current` to `buf`.
    - Release `entry_lock` from `current`.


- Function `void cache_write_back(struct cache_block *block)`:`
    - Assert that the `block->dirty` bit is 1.
    - Acquire the `entry_lock` of the `block`.
    - Put the `block->data` to disk.
    - Set the `block->dirty` bit to 0.
    - Release the `entry_lock` of the `block`.


#### Synchronization Issues

##### When one process is actively reading or writing data in a buffer cache block, how are other processes prevented from evicting that block? 

When a process is actively reading or writing in a buffer cache block, the `ref_count` associated with that cache block will be greater than zero. During the eviction process, we make sure that any block with a `ref_count` greater than zero is not considered for eviction.

##### During the eviction of a block from the cache, how are other processes prevented from attempting to access the block? 

For a process to be able to evict a block from the cache, it must acquire the `cache_lock`. If the `cache_lock` has been acquired, no other process can access the block because they need to wait for `cache_lock` to be released in order to find the block they want to use.

##### If a block is currently being loaded into the cache, how are other processes prevented from also loading it into a different cache entry? How are other processes prevented from accessing the block before it is fully loaded? 

This is handled in our `find_block()` function, which is atomic. Any access to a block needs to call this function, and in order to proceed, they need to acquire `cache_lock`. Thus, if one block is currently being loaded into the cache, this means the process loading this block has not released `cache_lock` yet. Once the block has been fully loaded, and the block's metadata updated, the process will release `cache_lock`, making it possible for other processes to find the block they want to access.

#### Rationale

An alternate design we had in mind was to simply return the `cache_block` corresponding to a particular `block_sector_t`, and let the `inode` functions handle the reads and writes to the cache. We decided to go against this design because we wanted all accesses to the cache data to be done via the provided interface, so that no external files should be able to access cache metadata freely.

We also considered implementing a mapping between `block_sector_t` and `struct cache block` using the provided hash table to decrease the time cost of looking up the block we need. However, we considered it to be a more complex implementation, so we're using an array of `struct cache_block` instead.


 
### Task 2: Extensible Files

#### Data Structure and Functions

- In `inode.c`, we modify the `struct inode_disk` and the `struct inode` by replacing `uint32_t unused[125]` in `struct inode_disk` and adding the following:
```
    struct inode_disk / inode {
        ...
        block_sector_t direct_ptrs[121];     /* The first 121 sectors of the file go here. */
        block_sector_t indirect_ptr;         /* Sector containing the indirect block. */
        block_sector_t doubly_indirect_ptr;  /* Sector containing the doubly indirect block. */
    };
```
- We remove the `struct inode_disk` inside `struct inode` because we can now use the buffer cache to access the `struct inode_disk`.
- In `struct inode` we add the following:
```
    struct inode {
        ...
        struct lock inode_lock;         /* Makes sure extending a file is synchronized. */
        ...
    };
```
- We create `struct indirect_block` which wraps around an array of direct pointers to sectors:
```
    struct indirect_block {
        block_sector_t direct_ptrs[BLOCK_SIZE / sizeof(block_sector_t)]; 
    };
```
- Modify `byte_to_sector(...)` in `inode.c` so that it returns back the actual sector that contains the specific byte, to allow for extending files and files that are not located in contiguous sectors on disk.

- In `userprog/syscall.c`, we add the following function to return the inode number (the sector number of the inode):
```
int inumber (int fd) {
    ...
}
```
- In `free-map.c` we add the following:
```
    struct lock free_map_lock; /* Synchronizes disk allocation/deallocation operations. */
    
    /* Wraps around free_map_allocate and free_map_release
    Sets the pointers to sectors appropriately inside INODE. */
    bool free_map_allocate_helper(size_t cnt, struct inode * inode);
    void free_map_release_helper(struct inode inode);
    
```

#### Algorithms

- On `inode_write_at(...)` and `inode_create(...)`, We check if the offset and the size of data we write to the file pass our current size or not; if the offset + size <= file's size, we call `cache_write` with the data and the size. Otherwise, we verify that the `free_map` has enough free sectors to allocate / extend the file (add more block sectors). If there are not enough free sectors, we abort the operation.
- In `byte_to_sector()`, we calculate the block_sector number. Now that sectors are not necessarily allocated contiguously, we use the index structure to figure out which sector contains the offset we are looking for. 
- In `free-map.c`, we create 2 wrapper functions for `free_map_allocate(...)` and `free_map_release(...)`:
    - Function `free_map_allocate_helper(size_t cnt, struct inode * inode)`:
        - Acquire the global lock on `free_map`.
        - Check if the `free_map` has enough empty block sectors to allocate the `cnt` number of new block sectors. If there are not enough free blocks, we return false, and abort the operation. This takes care of the disk exhaustion case.
        - Create a for loop of 121 times(the number of direct block sectors), each time we decrease the total number of sectors (`cnt`) that need to be allocated, we call `free_map_allocate(1, index)` to allocate a new block in memory and set the pointer of the direct block sector to that new allocated block; if `cnt` reaches 0 before or after finishing the loop, we release the lock and return 1. Otherwise, if `free_map_allocate` return 0 during the process, we release the lock, terminate and return 0.
        - If the function reaches this point, meaning that the number of sectors (`cnt`) is over 121 and we need to use indirect block sectors for the inode.
        - Create a for loop of 128 times(the number of indirect block sectors for single indirect block), and perform the same logic as above.
        - If the function reaches this point, meaning that the number of sectors (`cnt`) is over 128 + 121, we need to use the doubly indirect block sectors.
        - Create a for loop of 128*128 times(the number of doubly indirect block sectors) and perform the same logic as above. At this point, the function should already return since the maximum file size is guaranteed to be less than (121 + 128 + 128 * 128) * 512 Bytes.
    
    - Function `free_map_release_helper(struct inode inode)`:
        - Acquire the global lock on `free_map`.
        - Loop through the direct block sectors in `inode` and call `free_map_release(sector, 1)` on that `sector`.
        - Loop through the singly indirect block sectors in `inode` and perform the same logic.
        - Loop through the doubly indirect block sectors in `inode` and perform the same logic.
        - Release the lock on `free_map`.
        
- In `inumber()` of `userprog/syscall.c`
    - Iterate through the `file_list` from the current thread
    - Get the `file` pointer corresponds to the requested `fd`
    - Return the inode number from the pointer by: `file->inode->sector`


#### Synchronization Issues

Synchronization issues arise when we try to extend a file and another process wants to do an operation on that same file. Depending on the interleaving of the two processes, the result might be different. This is why anytime a file is extended, the process must acquire the `inode_lock` associated with the in-memory `struct inode`, which will be released after the file extension is done. Other processes will need to acquire this lock in order to access the sector. 

Another synchronization issue arises when two processes try to extend the space for a particular file using the wrapper functions we create in the `free-map.c`. If both processes try to allocate at the same time, one process might reserve some of the sectors that the second process needs. Now the second process need to release what has reserved at the beginning. Thus, we must acquire the lock and then make sure every process checks the number of available sectors first before allocating the extra sectors.  

#### Rationale
An alternate design we had in mind was to simply modify `free_map_allocate(...)` and `free_map_release(...)` to do the checking for enough number of sectors during allocation stage, and the freeing during deallocation stage.  However for simplicity we decide not to modify the two functions and abstract their usage by creating two wrapper functions in `free-map.c` which will help us to deal with `free_map_allocate(...)` and `free_map_release(...)` with more cleaner code. We also thought about whether acquiring/releasing the `free_map_lock` should be done within our wrapper functions or in `free_map_allocate(...)/free_map_release(...)`. We decided to make threads acquire a lock before even checking if there are enough disk sectors available in order to prevent race conditions. 

On writes, if a write is done far past the EOF, we decided to allocate the extra blocks even though the blocks will be entirely zero. This decision aligns with the way we are checking if there exists enough blocks in disk to allocate for extending a file. If we were to allocate those blocks only if they are written explicitly, we would need to keep track of extra state indicating that some bits in the free-map are not actually free.


### Task 3: Subdirectories

#### Data Structures and Functions

- In `thread.h`, we add a pointer to the current working directory.
```
    struct thread {
        ...
        struct inode * cwd; /* Current Working Directory */
        ...
    }
    
    struct file_list_elem {
        ...
        struct dir * dir;
    }
```

- In `inode.c`, we add the following to `struct inode` and `struct inode_disk`:
```
    struct inode / inode_disk {
        ...
        bool isdir; /* True if current inode is for a directory, false otherwise. */
        struct *inode parent; /* Points to parent directory (to support `..`)*/
    }
```
- In `userprog/syscalls.c`, we add the following functions:
```
static bool chdir (const char *dir) {...}      
static bool mkdir (const char *dir) {...}
static bool readdir (int fd, char *name) {...}
static bool isdir (int fd) {...}
```

#### Algorithms

- When a process starts another process using `exec (...)` syscall, make sure the child process has a copy of the `struct inode * cwd` of the parent process to guarantee isolation between the child process and the parent process. In other words, what happen in the child directory should not reflect on the parent directory. 

- To ensure we can handle both relative and absolute file paths, we parse the given filenames and handle the following cases:
    - If a file starts with `/`, we look at the rest of the file name relative to the root directory.
    - If a file starts with `.`, then we look at the rest of the file name relative to the current working directory.
    - If a file starts with `..`, then we look at the rest of the file name relative to the parent of the current working directory.

- From our Project 2 code, we are able to obtain the `file *` corresponding to a `fd` because each process keeps track of all the files it has opened. We modify the function so that it can return the file or directory associated with the `fd`.
 

- Modification to syscall.c:
    - Modification to previous functions:
        - In `open(...)` syscall, after obtaining the file pointer for the given fd, we check if the requested `fd` corresponds to a file directory. If so, we use `dir_open (...)`, and if not we proceed the function as before.
        - In `close(...)` syscall, we check if no other processes are using the same directory. If so we won't be able to close the directory. If not then make sure all open files in the directory are closed first to close the directory.
        - In `remove(...)` syscall, acquire the `inode_lock`, and then check if the directory exists using `struct inode * cwd`. Plus, we check if the directory is not used by any other process. If so then check if the directory is empty. In success delete the directory using `dir_remove(...)` function release the `inode_lock` and return true. If the directory does not exist release the `inode_lock` and return false.
    - Added functions:
        - `static bool chdir (const char *dir)`: Call function `dir_open()` to get the dir structure then we call `dir_lookup()` on the argument `dir` to check if it is a valid directory to go to. If it is valid, the process' `cwd` is set to the new inode of `dir`.
        - `static bool mkdir (const char *dir)`: Call function `filesys_create()` to create a new inode and also set the `isdir` of that new inode to true.
        - `static bool readdir (int fd, char *name)`: Iterate through the `file_list` from the current thread, get the `file` pointer corresponds to the requested `fd`, call `dir_open()` to get the dir structure and then call `dir_readdir()` on the file name and the dir from the inode.
        - `static bool isdir (int fd)`: Iterate through the `file_list` from the current thread, get the `file` pointer corresponds to the requested `fd`, get the `inode` pointer from the file structure, return `inode->isdir`.



#### Synchronization Issues

A synchronization issue arises when a user process tries to delete a directory if it is the cwd of a running process. To fix this issue, we use the `inode_lock` from the inode to make sure removing a directory is synchronized. Before we remove the directory, we acquire the `inode_lock`, verify that no file is inside the current directory, and the `in_use` is 0. If all the above are satisfied, we proceed with removing the directory, then release the `inode_lock`


#### Rationale

We thought about keeping track of existing directories by having a directory list in the `thread.h`, but we found out the way we access it repeats the logic of the file list we defined in project 2. We decided to build upon the file list and add the `struct dir` pointer corresponding to the file to achieve the task.


### Additional Questions

1. For this project, there are 2 optional buffer cache features that you can implement: write-behind and read-ahead. A buffer cache with write-behind will periodically flush dirty blocks to the filesystem block device, so that if a power outage occurs, the system will not lose as much data. Without write-behind, a write-back cache only needs to write data to disk when (1) the data is dirty and gets evicted from the cache, or (2) the system shuts down. A cache with read-ahead will predict which block the system will need next and fetch it in the background. A read-ahead cache can greatly improve the performance of sequential file reads and other easily-predictable file access patterns. Please discuss a possible implementation strategy for write-behind and a strategy for read-ahead.


- Function `write_behind()`: 
    - We create a new thread for this part. This thread will have an infinite for loop doing the following:
    - Acquire the global lock `cache_lock`.
    - Iterate through the cache_block array `cache`, if a `cache_block` has the `dirty` bit set to 1:
        - Acquire the lock `entry_lock` of that `cache_block`.
        - Call `cache_write_back()` to write the data back to memory.
        - Set the `dirty` bit to 0.
        - Release the lock `entry_lock`.
    - Release the lock `cache_lock`.
    - Sleep for a period of time then continue the loop.
    
- Function `read_ahead()`:
    - We create a new thread for this part. This thread will do the extra work to look up in memory for the next blocks in the memory.
    - We let this thread run in the background to pre-fetch the required blocks and put them in the cache. These blocks will be evicted or used the same way as in the clock algorithm.
 
