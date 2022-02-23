# CS 286 Lecture notes: February 23
## Interacting with Hardware Devices: Storage Devices, file systems, DMA, memory mapped IO
* A naive fork implementation uses synchronous copying
    * The OS performs repeated, synchronous page allocations and `memcpy()` operations to duplicate every valid pafe in the parent's low canonical memory.
    * Child gets a copy of parent's pagetable (updated to point to newly created copies).
## Making Fork Efficent
* The kernel gives the child a copy of the parent's page tables, but the OS does not allocate + `memcpy()` raw page data for low canonical state.
* Instead, mark the pages as read only in both parent and child.
* A page fault will occur when the child attempts to write to the part of the memory marked as read-only.

**Even with this optimization, physical RAM may be too small to hold all valid virtual pages!**

## Swapping Pages between RAM and Disk
* Physical RAM may be oversubscribed: aggregate number of pages in Virtual Address spaces is larger than number of physical pages
    * OS can swap pages between a swap device and memory (Swap device: SSD, or even remote network storage)
    * How does the OS associate a virtual page with its location on the swap device
## Case Study: Linux x86
**File-backed pages vs. Anonymous pages**:
* A file-backed page is an in-memory copy of an on-disk region
    * A page containing `libc` code
    * A page containing data from `/home/alice/file.txt`
* An anonymous page holds dynamic execution state for a process
    * Ex: A stack page
    * Ex: A heap page
* Both file-backed an anonymous pages can be evicted
    * A clean file-backed page can just be dropped from memory
    * A dirty file-backed page can be swapped to the associated location in the file (a file that contains modifications)
    * An evicted anonymous page must always be written to a special swap file.
## What about kernel pages?
* In theory, kernel pages can be swapped too!
    * However, this makes life so much harder!!!! So, we don't swap generally.
    * If kernel pages cannot be swapped, we get less interrupts such as page faults.
    * Some types of kernel code must execute quickly, so we cannot wait to swap it from a slow IO device
* Linux never swaps out kernel pages!
* Windows *does* allow some kernel pages to be swapped! :((

### How does linux set up swap:
* Swap info struct

How does linux distinguish between:
    * A non-present page that is swapped out to disk
    * A non-present unmapped page
### Linux x86-64: Virtual Memory Areas (VMAs)
`task_struct`: Per kernel thread data structure in linux. Inside this we have `mm_struct`, memory map structure. Holds the permissions for memory areas
* From the perspective of user level code, a virtual address is valid to access if the address is covered by a VMA
* However, the "Present?" bit in the relevant PTE indicates whether the relevant page is actually in physical memory.
### Demand Paging
* In most OSes, when a new program is loaded, the associated process gets:
    * No/very few virtual pages in physical RAM; thus, the process gets ...
    * A page table with almost all present bits turned off
* As the process executes, its instructions touch virtual addresses which are not resident in physical RAM
    * Page faults induce the OS to bring the missing virtual pages into physical RAM on-demand.
## Working Sets
* Working set: the set of virtual pages that the process is currently accessing
* If there is enough free RAM to hold the working set: 
    * The process generates page faults until the working set resides in RAM
    * The process won't generate additional page faults until
        * The working set changes
        * The OS proactively swaps out pages in the working set (process priority)
* If the working set doesn't fit in memory ... there is thrashing
    * Most memory accesses will cause page faults
    * The system will spend most of its time reading and writing to disk, not performing useful computation
* Thrashing can also occur if the OS has a bad eviction strategy, bad prefetching strategy.
* There is inherently too much demand for physical memory space

