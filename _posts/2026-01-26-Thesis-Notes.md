---
layout: post
hide_sidebar: true
title: "Thesis Notes"
---
<style>
  .sidebar, .site-sidebar, .secondary, aside { 
    display: none !important; 
  }

  .main, .content, .post, .wrapper { 
    max-width: 100% !important; 
    margin-left: 0 !important; 
    margin-right: 0 !important; 
    padding: 20px !important;
  }
</style>

I needed to compile the kernel for an x86 system, but I use an ARM-based Mac.

## Orbstack

 - initramfs (**init**ial **RAM** **f**ile**s**ystem) - temporary filesystem (tmpfs, in-memory) that the linux kernel loads into RAM on boot.

It mounts the real root (root = top level directory in any filesystem), especially when there are complex configurations. It loads the drivers (kernel modules) for different filesystems / hardware that aren't built directly into the kernel. 

initramfs images - may be a cpio archive - a stream of all the directories and files concatenated together, with headers containing metadata such as file names. This archive is "unpacked" into the RAM during boot.

initrd was the ancestor of initramfs. It used to be an actual filesystem image (with superblocks, inodes and stuff).

 - RAID (Redundant Array of Independent Disks) - array of independent physical drives as a single logical unit.

#### Linux Memory Stuff

 - physical and virtual memory

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

**How does the kernel organize memory?** 

Physical memory is divided into page frames or pages. The size of a page is arch specific (sometimes you can select the page size at kennel build time).
a Physical page <–> mapped to virtual page(s)
This mapping is done by page tables (arranged as multi-way tree basically).

  So you choose a fixed page size - most often 4096B - but kernel also has:

-   huge pages: when you let the upper levels of the page table map contiguous pages of large sizes
    
-   compound pages: Every “base” page has an associated struct page.
In compound pages: first page struct is marked as “head” and now has compound page info.
The rest are tail pages, have pointer to head page struct.

The problem with compound pages: ambiguity. When a function that works with struct page receives a tail page, shall it work with the tail page itself or with the compound page?
Solution: struct folio 

A folio is a physically, virtually and logically contiguous set of bytes. It is a power-of-two in size, and it is aligned to that same power-of-two. It is at least as large as %PAGE_SIZE. If it is in the page cache, it is at a file offset which is a multiple of that power-of-two. It may be mapped into userspace at an address which is at an arbitrary page offset, but its kernel virtual address is aligned to its size.
A folio is basically a struct page, that is guaranteed to not be a tail page. It's a new struct to replace "compound pages" to resolve the above ambiguity problem.
This LWN article discusses this: [Clarifying memory management with page folios [LWN.net]](https://lwn.net/Articles/849538/)  

A folio and a page are different views of the SAME memory. They are like “overlays” of the same memory. The first union in the folio is a union between a custom struct and a page. This custom struct has basically the same fields as a struct page!

- buffered vs memory-mapped I/O  
In buffered I/O, when a user application wants data, it makes a syscall and the context switches from User to Kernel mode. The VFS checks if the requested pages are present in the page cache - if not, it fetches them via DMA from disk and puts them into the page cache. Then this data is copied to the user buffer.  
In memory-mapped I/O a file (in the memory, basically the page cache) is mapped directly into the process' virtual address space. Once mapped, accessing this memory IS accessing the file. The copying to user-buffer part is skipped.

- file-backed vs anonymous memory  
If there is no file on-disk backing this memory (e.g. program stack, heap). It can also be explicitly allocated using mmap.


**The data structures we are working with**

Every file in a filesystem in the Linux Kernel, is uniquely represented by an inode.  
Each inode has an associated struct address_space ("represents the contents of a cacheable, mappable object"). The inode is the "owner" of this struct address_space - the struct is allocated as part of the inode's memory.    
The file will contain multiple pages / folios, and each of these will have a pointer back to this struct address_space.  

The struct address_space contains:  
> struct xarray		i_pages;
This is the actual Linux page cache.  

An xarray is essentially a sparse array of pointers (internal implementation is a radix tree). 
- An xarray is a small data structure.  

#### Following the read syscall
 
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








