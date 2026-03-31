---
layout: post
title: "Thesis Notes"
---
<style>
  .wrapper {
    width: 100% !important;
    max-width: 100% !important;
    margin: 0 !important;
    padding: 0 !important;
    display: block !important;
  }

  section {
    width: auto !important;
    float: none !important;
    margin: 0 40px !important; /* Adjust this for left/right breathing room */
    padding-top: 60px !important; /* Leaves room at the top for the menu button */
  }

  header {
    position: fixed !important;
    top: 0 !important;
    left: -320px !important; /* Hides it off-screen to the left */
    width: 300px !important;
    height: 100vh !important;
    background-color: #f6f8fa !important; /* Light GitHub-style gray */
    box-shadow: 2px 0 5px rgba(0,0,0,0.1) !important;
    transition: left 0.3s ease !important;
    z-index: 9998 !important;
    overflow-y: auto !important;
    padding: 60px 20px 20px 20px !important;
  }

  header.is-open {
    left: 0 !important;
  }

  footer {
    display: none !important;
  }

  #custom-sidebar-toggle {
    position: fixed;
    top: 15px;
    left: 15px;
    z-index: 9999;
    padding: 8px 12px;
    background: #24292e;
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-family: inherit;
    font-weight: bold;
    box-shadow: 0 2px 4px rgba(0,0,0,0.2);
    transition: background 0.2s;
  }
  
  #custom-sidebar-toggle:hover {
    background: #0366d6;
  }
</style>

<button id="custom-sidebar-toggle">☰ Menu</button>

<script>
  document.getElementById('custom-sidebar-toggle').addEventListener('click', function() {
    // Targets the header tag specifically since that acts as the sidebar in minimal
    const headerSidebar = document.querySelector('header');
    if (headerSidebar) {
      headerSidebar.classList.toggle('is-open');
    }
  });
</script>

# Boot

 - initramfs (**init**ial **RAM** **f**ile**s**ystem) - temporary filesystem (tmpfs, in-memory) that the linux kernel loads into RAM on boot.

It mounts the real root (root = top level directory in any filesystem), especially when there are complex configurations. It loads the drivers (kernel modules) for different filesystems / hardware that aren't built directly into the kernel. 

initramfs images - may be a cpio archive - a stream of all the directories and files concatenated together, with headers containing metadata such as file names. This archive is "unpacked" into the RAM during boot.

initrd was the ancestor of initramfs. It used to be an actual filesystem image (with superblocks, inodes and stuff).

 - RAID (Redundant Array of Independent Disks) - array of independent physical drives as a single logical unit.
 
# Memory Management and Filesystems 

## Physical and Virtual memory

Physical memory is:  
    - a limited resource  
	- not necessarily contiguous (is accessible as a set of distinct address ranges)  
	- different architectures (and even different implementations of the same architecture) may define these address ranges differently  
	- RAM is small and limited. It’s also “random access” (the processor can access all memory locations in the RAM equally fast).  So there’s a bunch of processes all vying for a slice of this RAM. That makes things complicated from the process POV. If it has to constantly track exactly which physical RAM addresses it has access to, which are free, and you have to make sure no processes overwrite each other’s locations and stuff…  
	TL/DR: dealing with physical memory directly is complex.  
	So we offload all that to the kernel, via virtual memory.

Virtual memory is an abstraction of physical memory, that all applications see.
They use virtual addresses that are translated to physical addresses.

Every process gets its own private virtual address space.

![Process Virtual Address Space]({{https://blooberry-muffin.github.io/}}/images/virtual_address_space.png)

Each process lives in a sandbox, believing it has complete, unrestricted access to all addresses from 0x0 - 0xffffffffffffffff (all possible addressable memory). Processes aren't even aware of, and cannot access, the virtual address space of another process.

So processes and their programmers don't have to worry about low-level memory stuff, they interact with this Virtual Address Space and the kernel handles the rest.

Note: There's some confusion between how "virtual memory" sometimes refers to the concept of secondary memory devices such as SSD being seen as a part of the "main memory" by the kernel. 
So: the kernel CAN map virtual addresses to the disk. Less important, less used data can be mapped to slow disk rather that precious, scarce RAM. This is a PART of the virtual memory concept, but the MAIN thing is the abstraction of physical memory for processes. 

For example, if during a running process you see this via strace:

mmap(NULL, 180840, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f9ab51e1000

> mmap: syscall to map a file into a process’ virtual memory space

> arg1: address to map to, if NULL, kernel can choose it

> arg2: length in bytes to map

> arg3: desired memory protection, to decide permissions and stuff

> arg4: flags, stuff about how these mappings are (shared / private) for other processes mapping to this region

> arg5: file descriptor, here it is “3” because a previous syscall assigned that file to “3”

> arg6: offset in file

> 0x7f9ab51e1000 - this is the address the kernel selected in the VAS (because 1st arg was NULL)

The kernel code is also mapped into the process' Virtual Address Space to avoid context-switching overhead (we have plenty of space in the VAS anyway). So the VAS is split into two sections: User VAS and Kernel VAS. 

## File-backed vs Anonymous Memory  
If there is no file on-disk backing this memory (e.g. program stack, heap). It can also be explicitly allocated using mmap.

We'll take a deeper look at reclamation for file-backed memory later on this page.  

- Swapping: When you need to reclaim anon memory, you create a swap space on the disk, where you can place old anon pages temporarily (obviously disk access is very slow, so usually this scenario where a process uses up so much RAM that you need to frequently access the swap space, means you need more RAM). There is a "swap cache", which acts as an intermeditae between the RAM and the swap space on disk, to prevent race conditions and redundant writes. 

- In-Memory filesystems: Sometimes anon memory can present a file-like interface. shm and tmpfs are two “in-memory” filesystems (i.e. they are not actually files stored on disk - these deal exclusively with anon memory - but a file-like interface is provided to anon mem too because it’s super clean and easy)

## Buffered vs Memory-Mapped I/O  
In buffered I/O, when a user application wants data, it makes a syscall and the context switches from User to Kernel mode. The VFS checks if the requested pages are present in the page cache - if not, it fetches them from disk and puts them into the page cache. Then this data is copied to the user buffer.

In memory-mapped I/O a file (in the memory, basically the page cache) is mapped directly into the process' virtual address space. Once mapped, accessing this memory IS accessing the file. The copying to user-buffer part is skipped.

- Direct I/O: Skips the page cache entirely. Data transfer happens directly between the userspace buffer and the storage device. open() with the O_DIRECT flag triggers this.

## Representing Pages

Physical memory is divided into page frames. The size of a page frame is arch-specific (sometimes the size can be selected during kernel build - most commonly 4096B). The page frame number or pfn uniquely represents every physical page frame - it is the physical address of the frame divided by PAGE_SIZE. 

Page frames are mapped to virtual "pages" (contiguous fixed-size blocks of memory in the virtual address space). This mapping is done by page tables (arranged as hierarchical multi-way trees). There's also a hardware cache called Translation Lookaside Buffer that stores recent virtual-to-physical address translations to avoid some slow multi-level tree lookups.

For larger sizes, the kernel offers:
-   huge pages: when you let the upper levels of the page table map contiguous pages of large sizes    
-   compound pages: Every “base” page has an associated struct page. In compound pages: the first page struct is marked as “head” and now has compound page info. The rest are tail pages, have pointer to head page struct.

The problem with compound pages: ambiguity. When a function that works with struct page receives a tail page, shall it work with the tail page itself or with the compound page?
Solution: struct folio 

A folio is a physically, virtually and logically contiguous set of bytes. It is a power-of-two in size, and it is aligned to that same power-of-two. It is at least as large as %PAGE_SIZE. If it is in the page cache, it is at a file offset which is a multiple of that power-of-two. It may be mapped into userspace at an address which is at an arbitrary page offset, but its kernel virtual address is aligned to its size.
A folio is basically a struct page, that is guaranteed to not be a tail page. It's a new struct to replace "compound pages" to resolve the above ambiguity problem.
This LWN article discusses this: [Clarifying memory management with page folios [LWN.net]](https://lwn.net/Articles/849538/)  

A folio and a page are different views of the SAME memory. They are like “overlays” of the same memory. The first union in the folio is a union between a custom struct and a page. This custom struct has basically the same fields as a struct page!

Each physical page in the memory is represented by the **struct page** in the Kernel. The structures for all page frames form an array mem_map[] of type struct page so that you can easily retrieve the structure for a specific page frame by using the page frame number as the index.

## Linux Filesystem Data Structures

- Every file in a filesystem in the Linux Kernel, is uniquely represented by an **inode**.  
- Each inode has an associated **address_space** ("represents the contents of a cacheable, mappable object"). The inode is the "owner" of the address_space - the struct is allocated as part of the inode's memory.    
The file will contain multiple pages / folios, and each of these will have a pointer back to this struct address_space.  
- A **struct file** represents an OPEN file. It is the kernel-side implementation of the file descriptor within a process. A single inode can have multiple associated struct file (the same file may be opened by multiple processes, or the same process may open the file multiple times).  
- The **struct dentry** is an in-memory data structure that represents a component of a file path. It’s what links inodes and filenames (inode doesn’t actually contain filenames). And it’s very important for caching for fast lookup by the VFS (the dentry cache).  

Note: The struct inode has both an address_space and a pointer to it.
> struct address_space	i_data;
> struct address_space	*i_mapping;
Usually, the i_mapping points to the i_data. In specific cases such as block device files, they can be different (where a "master" inode owns the actual struct and the other inodes point to that i_data through their own i_mapping pointers).


## Linux Memory Management Data Structures

- struct mm_struct: There's one unique mm_struct for each Virtual Address Space. It's a descriptor of the memory for this VAS. It has the mm_mt which is a tree of all the Virtual Memory Areas (contiguous ranges in the virtual address spaces) for this VAS. It also has a pointer to the root of the multi-level page tables.
- struct vm_area_struct: Represents one contiguous area in virtual memory.

## The Page Cache
- The struct address_space contains:  
> struct xarray		i_pages;
This is the actual Linux page cache.  

An xarray is essentially a sparse array of pointers (internal implementation is a radix tree). 
- An xarray is a small data structure.  
- It is indexed by an unsigned long. In the page cache, this is the pgoff_t i.e. the "page number" in the file. This is what the i_pages xarray is indexed by - you'll find the pointer to "page #X" at "index #X" in the xarray. 

A note on the confusion about the word "cache":  
In hardware, "cache" is faster, more expensive memory between the processor and main memory. In the Kernel, the "page cache" simply refers to pages that have been fetched from disk into the main memory. The Linux page cache is per-file: each file has an associated "page cache" - the i_pages xarray is the data structure representing it (it stores pointers to the locations of the file's pages, that have been fetched into the main memory).

## Following the read syscall
 
In Linux, filesystem related syscalls (like read) are implemented by each type of filesystem on its own. This is all abstracted by the VFS layer.

There's a large number of dispatchers (functions that route to the correct execution point but don't do the execution themselves).

Here's the path for the read syscall:
“sys_read”  
→   
ksys_read  
→  
vfs_read  
→  
new_sync_read  
→  
xfs_file_read_iter [for the xfs filesystem]  
→   
xfs_file_buffered_read  
→  
generic_file_read_iter  [This is the "read_iter()" routine for all filesystems that can use the page cache directly. It handles direct I/O else sends on to filemap_read for page cache reading]  
→   
filemap_read  
→  
filemap_get_pages [This function waits for the batch to be filled, from the page cache; if not, fetches from disk including readahead]  
→  
filemap_get_read_batch [actual function which walks the page cache and adds pages to the batch]

The read syscall looks like this:  
> SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count) { 	return ksys_read(fd, buf, count); }

What is this file descriptor?  
The open() syscall returns an integer between 0 (empty) and N → this is the “file descriptor” in the user space. This number is mapped via the file descriptor table to a struct file (so basically this table takes the int fd and returns a pointer to the struct file that it refers to). In the kernel space it is a pointer to the struct file + some flags or empty (if 0).  

So the sys_read starts by receiving a file descriptor, then vfs_read() gets the struct file * it actually points to. The new_sync_read() function wraps this struct file * + some other stuff into a struct kiocb. From this point onwards this is what gets sent around!

filemap_get_pages() is the key "entry point" for buffered reads that go through the page cache. The comments below explain what goes on here:

```c

static int filemap_get_pages(struct kiocb *iocb, size_t count,
       struct folio_batch *fbatch, bool need_uptodate)
{
   struct file *filp = iocb->ki_filp;
   struct address_space *mapping = filp->f_mapping;
   struct file_ra_state *ra = &filp->f_ra;
   
   // "index" to "last index" - these are the page numbers in the file, that we want to read
   pgoff_t index = iocb->ki_pos >> PAGE_SHIFT;
   pgoff_t last_index;
   
   struct folio *folio, *f;
   unsigned int flags;
   int err = 0;


   /* "last_index" is the index of the page beyond the end of the read */
   last_index = DIV_ROUND_UP(iocb->ki_pos + count, PAGE_SIZE);
retry:
   if (fatal_signal_pending(current))
       return -EINTR;

// first, try seeing if these pages are all already in the page cache
// at the first cache-miss, we return an empty batch
   filemap_get_read_batch(mapping, index, last_index - 1, fbatch);

// if an empty batch was returned, we know there's a cache miss and try to fetch from disk
   if (!folio_batch_count(fbatch)) { // if the batch wasn’t full
       if (iocb->ki_flags & IOCB_NOIO)
           return -EAGAIN;
       if (iocb->ki_flags & IOCB_NOWAIT)
           flags = memalloc_noio_save();

// This function, through various further function calls, will actually fetch pages from disk into the page cache.
// It also applies "readahead" logic - we don't just fetch the requested pages, we try to predict if this is a sequential read and pre-fetch the next pages in the file.
// First, the readahead control struct is set up in page_cache_sync_readahead(), then page_cache_sync_ra() is called. Here,
// 1. Either "do_forced_ra" is called if, for some reason, we don't actually want to apply readahead logic - it leads to page_cache_ra_unbounded() which allocates new folios via filemap_alloc_folio() and adds them to the page cache via filemap_add_folio().
// 2. Or, if we do want to read ahead, the appropriate calculations are done (we ensure that atleast the requested pages upto last_index are fetched + a readahead window) and page_cache_ra_order() is called. Ultimately, ra_alloc_folio() is called and this also allocates folios via filemap_alloc_folio() and adds them to the page cache via filemap_add_folio().
// In both cases, the pages from disk are then fetched into the newly prepared folios.
 
       page_cache_sync_readahead(mapping, ra, filp, index,
               last_index - index); 
       if (iocb->ki_flags & IOCB_NOWAIT)
           memalloc_noio_restore(flags);
// we expect that the page cache NOW has the pages we wanted - try fetching them from the page cache again
       filemap_get_read_batch(mapping, index, last_index - 1, fbatch);
   }

// if the page cache STILL doesn't have the requested pages, we fall back to filemap_create_folio(). 
// This function allocates exactly one new folio, adds it to cache and fetches it from disk. So, we would return only ONE page to the filemap_read function - which had a loop that fetches all the requested pages batch-by-batch, alling filemap_get_pages as many times as needed. Now we will just fetch one page at a time, but we will ensure that all the requested pages are delivered.
// We do this because if readahead failed, the most likely reason is memory pressure.
   if (!folio_batch_count(fbatch)) {
       if (iocb->ki_flags & (IOCB_NOWAIT | IOCB_WAITQ))
           return -EAGAIN;

       err = filemap_create_folio(filp, mapping, iocb->ki_pos, fbatch);

       if (err == AOP_TRUNCATED_PAGE)
           goto retry;
       return err;
   }
   folio = fbatch->folios[folio_batch_count(fbatch) - 1];
// Here  we check - are we getting close to the end of what we've cached so far? If yes, fire off an asynchronous background request for more data. 
   if (folio_test_readahead(folio)) {
       err = filemap_readahead(iocb, filp, mapping, folio, last_index);
       if (err)
           goto err;
   }
// Is the page we want to read right now currently empty and waiting for the disk? Yes? Put the application to sleep until the disk finishes.
   if (!folio_test_uptodate(folio)) {
       if ((iocb->ki_flags & IOCB_WAITQ) &&
           folio_batch_count(fbatch) > 1)
           iocb->ki_flags |= IOCB_NOWAIT;
       err = filemap_update_page(iocb, mapping, count, folio,
                     need_uptodate);
       if (err)
           goto err;
   }

// And this below is where PaCaR steps in, to intercept all the pages in the batch and verify if the reads are local!
   int nid = numa_node_id();
   for (int i = 0; i < folio_batch_count(fbatch); i++) {
       // replace the recently added folio to the fbatch
       f = fbatch->folios[i];
       fbatch->folios[i] = duplication_find_or_duplicate(f, nid);
   }
   trace_mm_filemap_get_pages(mapping, index, last_index - 1);
   return 0;
err:
   if (err < 0)
       folio_put(folio);
   if (likely(--fbatch->nr))
       return 0;
   if (err == AOP_TRUNCATED_PAGE)
       goto retry;
   return err;
}

```

### Side Note: Understanding the Profiling macros 

Consider these kinds of macros:

```c
#define  __folio_alloc_node(...)        alloc_hooks(__folio_alloc_node_noprof(__VA_ARGS__))
```

Here, __folio_alloc_node() is a wrapper around __folio_alloc_node_noprof. So at compile time, calls to __folio_alloc_node are redirected to __folio_alloc_node_noprof and all its args are passed to it.
“alloc_hooks” adds instrumentation logic - it is used for memory allocation profiling. It adds metadata to a memory allocation call, allowing detailed tracking of memory usage back to the actual code that did the allocation.

Another example is:

```c
#define __alloc_pages(...)			alloc_hooks(__alloc_pages_noprof(__VA_ARGS__))
```

So: when the memory allocation profiling framework was implemented, the original __alloc_pages function was renamed to __alloc_pages_noprof (noprof stands for no profiling). The above macro wraps the function in "alloc_hooks" which does things like: recording the exact file and line number where allocation was requested, and logging memory usage against this "tag" once it succeeds. 


### Memory Reclamation

#### Memory Zones
The Kernel divides RAM into different "zones":
- ZONE_DMA: The lowest physical memory addresses - mostly for backwards compatibility (as legacy devices had much smaller addressing limits)
- ZONE_DMA32: Lower 32 Bits (4GB) of memory - for 32-bit peripherals.
- ZONE_NORMAL
- ZONE_MOVABLE: pseudo-zone containing memory pages that the kernel knows it can safely move, migrate, or reclaim - mostly for RAM hotplugging
- ZONE_DEVICE: persistent memory and specialized device-backed memory (e.g. GPUs) - becoming more important for heterogenous computing and ultra-fast storage devices (DAX)

#### Memory Watermarks
Each memory zone has three "levels" indicating how much free memory it has:
- HIGH: Plenty of free space, background processes for reclamation (like kswapd) are asleep.
- LOW: kswapd wakes up when the level drops below this. It starts reclaiming, until the memory is again above HIGH.
- MIN: Triggers direct reclaim - the process asking for memory is paused and made to reclaim memory first. This happens if kswapd isn't able to keep up with the allocation rate.

#### kswapd
kswapd is the kernel's background garbage collector. Let's look at how allocation of memory actually happens.

Above, in filemap_get_pages, we saw that all the paths use filemap_alloc_folio to allocate a new folio in memory. This wraps around the filemap_alloc_folio_noprof function, which is defined separately for CONFIG_NUMA and without. 
However, in all cases, the allocation is done via __folio_alloc_noprof --> __alloc_pages_noprof.

This function, __alloc_pages_noprof, is the MAIN function in the buddy memory allocator. It first tries allocating via:

```c
page = get_page_from_freelist(alloc_gfp, order, alloc_flags, &ac);
```
This function checks if there are enough free pages above the LOW watermark - if yes, returns these. 

If this fails, the __alloc_pages_noprof function calls:
```c
page = __alloc_pages_slowpath(alloc_gfp, order, &ac);
```
This makes a call to wake_all_kswapds (on NUMA machines, there's a kswapd per node). wake_all_kswapds() iterates through the memory zones being requested and calls wakeup_kswapd() for the appropriate nodes. This finally calls wake_up_interruptible() to actually wake the sleeping thread. 

int kswapd() calls balance_pgdat

#### The classic LRU implementation
In the classic LRU implementation, there are 5 LRU lists:
- LRU_ACTIVE_FILE
- LRU_INACTIVE_FILE
- LRU_ACTIVE_ANON
- LRU_INACTIVE_ANON
- LRU_UNEVICTABLE (not a "real" LRU list, but treated like one)

There are three "shrinker" functions than handle these lists: 
- shrink_active_list() --> "isolates" folios from the tail of the active list and adds them to the inactive list. The only exception: file-backed pages marked VM_EXEC (executable code) that are mapped into at least one process' page table. This protects against use-once streaming I/O (we ignore anon pages here, as they are not vulnerable to this case, plus JVM may create lots of anon VM_EXEC folios).
- shrink_inactive_list() --> isolates folios and sends them to shrink_folio_list(). Doesn't really do any "checks" of its own.
- shrink_folio_list() --> the heavy-lifter. This function does most of the eviction work:


How do we decide which list to prune from? 
get_scan_count() calculates how many pages to scan from each list and stores it. It uses heuristics like:
- Is there no swap space available or is there plenty of inactive file page cache lying around? Scan only the file lists then. It's preferred to evict file-backed pages before anon as they may be clean and don't have to be written back, while anon pages always must be written to swap.
- Is the file cache almost depleted? Scan anon. 
- Are we close to OOM? SCAN. EVERYTHING. 
- If none of the extreme cases above is met, it uses the "swappiness" fraction and refaulting costs to calculate.

The shrink_lruvec() function then takes these calculations, and if any of the lists have > 0 scannable pages, it routes them to the shrink_(active|inactive)_list functions.


 
#### Preventing Thrashing 
Thrashing is when the kernel is continuously evicting pages and then refaulting them back in. The mechanism to prevent this is closely tied to the page cache xarray: the working set and shadow entries.

In the case of file-backed pages, the kernel maintains "shadow entries" to remember (for a while) the existence of pages that have been reclaimed off the inactive list. If those pages are "refaulted" back in, the kernel knows that it's pushing out useful pages and can make adjustments to try to avoid doing that.

When the page cache evicts a file-backed page, it stores a “shadow entry” in the i_pages xarray.
These shadow entries contain: 
- a bit that tells the xarray “I’m a value entry, and not an actual pointer”
- a bit that indicates whether the evicted page was part of the working set or not (a hot page or cold page)
- the NUMA Node ID
- the memory control group
- eviction timestamp

Note: the Shrinker for i_pages xarray itself: 
Suppose a workload is reading / writing massively larger amounts of memory than RAM. Then pages are constantly evicted and shadow entries created. If this workload also prevents the inode from being reclaimed (by keeping for example, the file descriptors open) - then it will create excessive shadow nodes i.e. xarray nodes consisting entirely of shadow entries. For this, the kernel has a "shrinker" mechanism to clean up shadow nodes periodically and not wait for inode reclaim to do so.

A list of shadow nodes is maintained as a global var in workingset.c:
```c
struct list_lru shadow_nodes;
```

The kernel defines a function workingset_update_node() to handle this stuff. This is "attached" to the xarray via xas_set_update(). e.g. both __filemap_add_folio and page_cache_delete call mapping_set_update, which does 
```c 
xas_set_update(xas, workingset_update_node);
```

The xarray structure consists of multiple xa_node, and each xa_node has a private_list member of type list_head, which may be part of the global shadow_nodes list. If it is --> then this node is a valid shadow node. The workingset_update_node() handles all this stuff - adding / removing xa_nodes to the global shadow_nodes list. This function is called by all the functions that change the structure of the xarray: xas_alloc, xas_shrink, xas_delete_node, xas_free_nodes, xas_expand, update_node, xas_split.







### How does this ref thing actually work?

So a page / folio is a metadata structure which the Linux Kernel uses to keep track of physical memory locations. Each page has an associated reference count, which tells us how many functions / data structures are using / storing this page. Whenever a function or data structure does folio_get(folio) --> refcount gets +1 and it's the ref-getter's responsibility to also put the ref when done. As long as refcount > 0, it means SOMEONE is using / taking responsibility for this folio, and this memory location cannot be repurposed.

When the page is first allocated: __folio_alloc_node() wraps around __folio_alloc_node_noprof(), which calls __alloc_pages_noprof() --> the main allocation function. 

Inside this, prep_new_page() is called --> post_alloc_hook() --> set_page_refcounted. 
This function asserts that the refcount is 0 so far, and then forcefully sets it to 1.

So - a newly allocated page has a refcount of 1 - this is the "existence" refcount. 

In filemap_get_read_batch(), we see that every folio added to the batch, has a reference on it via folio_get. In filemap_read, once the contents of the folios is copied over to the userspace buffer, the function explicitly puts each folio.

### Locking in the Linux Kernel
Why do we need locking at all? Because critical regions need to be properly used if accessed concurrently (race conditions).
- “spinlock” → try to acquire a lock, keep spinning until you do
- mutex → can also block until lock is acquired, so the processor can go do other stuff meanwhile

If you have a kernel without SMP, without pre-emption: you don’t need spinlocks at all (because nobody else can run at the same time!)

softirq: software interrupt request, the second half of interrupt processing. They cannot sleep. 
So, if a critical section may be accessed by a softirq, it must have two conditions on the lock:
- It cannot be a “sleepable” lock (therefore, must be a spinlock)
- 
That’s why we use: spin_lock_bh

### Appendix 

Some closer looks at specific functions. 

The struct readahaed_control provides a good idea of what readahead does. The DEFINE_READAHEAD macro sets it up:

```c

struct readahead_control {
    struct file *file; // the open file whose data is being read ahead
    struct address_space *mapping; // this file’s address_space
    struct file_ra_state *ra; //the “readahead state”
/* private: use the readahead_* accessors instead */
// fun fact: this privacy is not enforced by C itself, but through this comment!
    pgoff_t _index; // current page index in the readahead
    unsigned int _nr_pages; // how many pages to read ahead? 
    unsigned int _batch_count;
    bool dropbehind;
    bool _workingset;
    unsigned long _pflags;
};

```

And a closer look at page_cache_sync_ra, to better understand the "forced readahead" vs the normal readahead case:

```c
void page_cache_sync_ra(struct readahead_control *ractl,
        unsigned long req_count)
{
    pgoff_t index = readahead_index(ractl); // this is the “first” page index
    // if we access the file RANDOMLY, then of course we don't predict and use sequential read patterns
    bool do_forced_ra = ractl->file && (ractl->file->f_mode & FMODE_RANDOM); 
    struct file_ra_state *ra = ractl->ra;
    unsigned long max_pages, contig_count;
    pgoff_t prev_index, miss;

    /*
     * Even if readahead is disabled, issue this request as readahead
     * as we'll need it to satisfy the requested range. The forced
     * readahead will do the right thing and limit the read to just the
     * requested range, which we'll set to 1 page for this case.
     */
    // is readahead disabled? Or is the device busy? We then fetch only the one page the user requested.
    if (!ra->ra_pages || blk_cgroup_congested()) {
        if (!ractl->file)
            return;
        req_count = 1;
        do_forced_ra = true;
    }

    /* be dumb */
    // no fancy prediction tricks - just fetch exactly what the user asked for
    if (do_forced_ra) {
        force_page_cache_ra(ractl, req_count);
        return;
    }
    // actual number of pages to prefetch based on system limits, readahead window etc
    max_pages = ractl_max_pages(ractl, req_count);
    // the page number at which the previous read ended
    prev_index = (unsigned long long)ra->prev_pos >> PAGE_SHIFT;

    /*
     * A start of file, oversized read, or sequential cache miss:
     * trivial case: (index - prev_index) == 1
     * unaligned reads: (index - prev_index) == 0
     */
    
    // index = 0 --> start of file 
    // req_count > max_pages --> oversized read
    // index - prev_index <= 1UL --> sequential read, but we are here so there must have been a cache miss.
    // in all these casesL new readahead window
    if (!index || req_count > max_pages || index - prev_index <= 1UL) {
        // set the start of readahead window as the current index
        ra->start = index;
      // set the size of the readahead window
      // small reads x4, medium reads x2, large reads capped at max
        ra->size = get_init_ra_size(req_count, max_pages);
      // how many pages to fetch asynchronously?
        ra->async_size = ra->size > req_count ? ra->size - req_count :
                            ra->size >> 1;
        goto readit;
    }

    /*
     * Query the page cache and look for the traces(cached history pages)
     * that a sequential stream would leave behind.
     */

     // literally walk backward from the current index, in the i_pages, find the first "hole"
     // this is used to estimate sequential read
    rcu_read_lock();
    miss = page_cache_prev_miss(ractl->mapping, index - 1, max_pages);
    rcu_read_unlock();
    contig_count = index - miss - 1;
    /*
     * Standalone, small random read. Read as is, and do not pollute the
     * readahead state.
     */

     // if the prev continuous string of pages is shorter than the current request, then it's random-ish
     // no readahead, then
    if (contig_count <= req_count) {
        do_page_cache_ra(ractl, req_count, 0);
        return;
    }
    /*
     * File cached from the beginning:
     * it is a strong indication of long-run stream (or whole-file-read)
     */

     // if we’ve been reading this file from the start → readahead aggressively
    if (miss == ULONG_MAX)
        contig_count *= 2;
    ra->start = index;
    ra->size = min(contig_count + req_count, max_pages);
    ra->async_size = 1;
    // only 1 page fetched async beyond the window
readit:
// finally: the actual read!
    ra->order = 0;
    ractl->_index = ra->start;
    page_cache_ra_order(ractl, ra);
}
EXPORT_SYMBOL_GPL(page_cache_sync_ra);

```

And continuing down the "predict and read" path:

```c
void page_cache_ra_order(struct readahead_control *ractl,
        struct file_ra_state *ra)
{
    struct address_space *mapping = ractl->mapping;
    pgoff_t start = readahead_index(ractl);
    pgoff_t index = start;
    unsigned int min_order = mapping_min_folio_order(mapping);
    // get the EOF page index
    pgoff_t limit = (i_size_read(mapping->host) - 1) >> PAGE_SHIFT;
    // sets the boundary between sync and async page fetches
    pgoff_t mark = index + ra->size - ra->async_size;
    unsigned int nofs;
    int err = 0;
    // flags for mem alloc behaviour
    gfp_t gfp = readahead_gfp_mask(mapping);
    // how large is each folio we allocate during this fetch?
    unsigned int new_order = ra->order;
    // if large folios are disabled, go to fallback “page by page” path
    if (!mapping_large_folio_support(mapping)) {
        ra->order = 0;
        goto fallback;
    }

    // cap the readahead to EOF!
    limit = min(limit, index + ra->size - 1);

    // set the correct folio order
    new_order = min(mapping_max_folio_order(mapping), new_order);
    new_order = min_t(unsigned int, new_order, ilog2(ra->size));
    new_order = max(new_order, min_order);

    ra->order = new_order;

    /* See comment in page_cache_ra_unbounded() */
    // no FS recurse i.e. don’t call FS while I’m in FS code, for eg, for reclaim
    nofs = memalloc_nofs_save();

    filemap_invalidate_lock_shared(mapping);
    /*
     * If the new_order is greater than min_order and index is
     * already aligned to new_order, then this will be noop as index
     * aligned to new_order should also be aligned to min_order.
     */
    // align the index with folio order
    ractl->_index = mapping_align_index(mapping, index);
    index = readahead_index(ractl);

    while (index <= limit) {
        unsigned int order = new_order;

        /* Align with smaller pages if needed */
        if (index & ((1UL << order) - 1))
            order = __ffs(index);
        /* Don't allocate pages past EOF */
        while (order > min_order && index + (1UL << order) - 1 > limit)
            order--;
        // actually allocating the folios
        err = ra_alloc_folio(ractl, index, mark, order, gfp);
        if (err)
            break;
        index += 1UL << order;
    }

    read_pages(ractl);
    // unlock, restore mem alloc flags
    filemap_invalidate_unlock_shared(mapping);
    memalloc_nofs_restore(nofs);
 
    /*
     * If there were already pages in the page cache, then we may have
     * left some gaps.  Let the regular readahead code take care of this
     * situation below.
     */
    if (!err)
        return;
fallback:
    /*
     * ->readahead() may have updated readahead window size so we have to
     * check there's still something to read.
     */
    if (ra->size > index - start)
        do_page_cache_ra(ractl, ra->size - (index - start),
                 ra->async_size);
}

```

The filemap_add_folio function is the main point where new folios actually get added to the cache, and has several interesting points:

```c

noinline int __filemap_add_folio(struct address_space *mapping,
       struct folio *folio, pgoff_t index, gfp_t gfp, void **shadowp)
{
  // set up the cursor to walk through the i_pages xarray
   XA_STATE(xas, &mapping->i_pages, index);
   void *alloced_shadow = NULL;
   int alloced_order = 0;
   bool huge;
   long nr;

// the folio needs to be locked while being added to cache!
   VM_BUG_ON_FOLIO(!folio_test_locked(folio), folio);
// the swap cache is a COMPLETELY different thing, btw
   VM_BUG_ON_FOLIO(folio_test_swapbacked(folio), folio);
// folio shouldn't be smaller than the minimum size allowed by this mapping
   VM_BUG_ON_FOLIO(folio_order(folio) < mapping_min_folio_order(mapping),
           folio);
// the xarray state iterator should point to the current mapping
   mapping_set_update(&xas, mapping);

// check index alignment
   VM_BUG_ON_FOLIO(index & (folio_nr_pages(folio) - 1), folio);
// very important - since the page cache supports multi-order (large) folios, we store multiple pointers into
// i_pages, and xas_set_order lets us do that in one go
   xas_set_order(&xas, index, folio_order(folio));
// legacy huge TLB pages
   huge = folio_test_hugetlb(folio);
// total number of pages in the folio - this is the number of pointers that will be stored into i_pages 
// and hence the number of refs that i_pages takes for this folio
   nr = folio_nr_pages(folio);

// set memory alloc masks to prevent recursion / deadlock
   gfp &= GFP_RECLAIM_MASK;
// as noted above: increment this folio's ref count by nr_pages 
   folio_ref_add(folio, nr);
   folio->mapping = mapping;
// reset the index, it may have changed during alignment
   folio->index = xas.xa_index;
   for (;;) {
       int order = -1, split_order = 0;
       void *entry, *old = NULL;

// while writing into i_pages, disable interrupts
       xas_lock_irq(&xas);
       xas_for_each_conflict(&xas, entry) {
           old = entry;
          // if there was an actual pointer entry here - something's up! exit
           if (!xa_is_value(entry)) {
               xas_set_err(&xas, -EEXIST);
               goto unlock;
           }
           /*
            * If a larger entry exists,
            * it will be the first and only entry iterated.
            */
           if (order == -1)
               order = xas_get_order(&xas);
       }


       /* entry may have changed before we re-acquire the lock */
       if (alloced_order && (old != alloced_shadow || order != alloced_order)) {
           xas_destroy(&xas);
           alloced_order = 0;
       }

// if there was a non-pointer entry: it's a shadow entry
// first: was the previous entry here a larger folio than we are trying to store now?
// if so, we need to "split" this slot into smaller ones
// we CANNOT allocate memory while holding a spinlock! 
// so, in case we DO need memory allocation, exit and retry
       if (old) {
           if (order > 0 && order > folio_order(folio)) {
               /* How to handle large swap entries? */
               BUG_ON(shmem_mapping(mapping));
               if (!alloced_order) {
                   split_order = order;
                   goto unlock;
               }
               xas_split(&xas, old, order);
               xas_reset(&xas);
           }
           if (shadowp)
               *shadowp = old;
       }
      // now actually store the pointer to the folio
       xas_store(&xas, folio);
       if (xas_error(&xas))
           goto unlock;
      // increment the number of pages this mapping tracks
       mapping->nrpages += nr;
       /* hugetlb pages do not participate in page cache accounting */
       if (!huge) {
           __lruvec_stat_mod_folio(folio, NR_FILE_PAGES, nr);
           if (folio_test_pmd_mappable(folio))
               __lruvec_stat_mod_folio(folio,
                       NR_FILE_THPS, nr);
       }

unlock:
       xas_unlock_irq(&xas);


       /* split needed, alloc here and retry. */
       if (split_order) {
           xas_split_alloc(&xas, old, split_order, gfp);
           if (xas_error(&xas))
               goto error;
           alloced_shadow = old;
           alloced_order = split_order;
           xas_reset(&xas);
           continue;
       }


       if (!xas_nomem(&xas, gfp))
           break;
   }


   if (xas_error(&xas))
       goto error;


   trace_mm_filemap_add_to_page_cache(folio);
   return 0;
error:
   folio->mapping = NULL;
   /* Leave page->index set: truncation relies upon it */
   folio_put_refs(folio, nr);
   return xas_error(&xas);
}

```

#### General
- Pointers in C have a type so they can be directly dereferenced (the compiler knows what they point to). Void pointers can point to anything, but to be dereferenced, must be cast to something.
- cgroups: we can define groups of tasks (processes) and then allocate resources such as memory, CPU time, network bandwidth to these groups
- Fun fact: C structs can't have private members, so you must resort to programmer-enforced privacy. For e.g. in the struct readahead_control.
