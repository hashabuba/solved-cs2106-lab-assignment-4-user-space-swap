Download Link: https://assignmentchef.com/product/solved-cs2106-lab-assignment-4-user-space-swap
<br>
swap, or paging, is a way of “extending” the memory capacity of a computer by moving less-used <em>pages</em> to secondary storage, thereby making room for new pages, and moving pages back into main memory on-demand.

Although swap is typically implemented and provided by a kernel transparently to user programs, some operating systems provide features that allow programs to emulate swap in a similarly transparent fashion as well.

On Linux and other POSIX systems, when a process performs an invalid memory access, the kernel sends the SIGSEGV signal to the process. The default action for the signal is to terminate the process, and, in most cases, this is the only sensible thing to do: an invalid memory access usually implies a severe bug in the program or that memory corruption has occurred.

However, it is also possible to install a signal handler for SIGSEGV. While the C and POSIX specifications both state that the behaviour upon returning from a SIGSEGV signal handler is undefined, the behaviour on Linux is clear: it will return to the instruction that caused the invalid memory access and re-try the instruction. This means that by installing a SIGSEGV handler and performing actions to move memory, etc., it is possible to implement swap in user space. In effect, our SIGSEGV handler is acting as a page fault handler.

In this lab, you will implement a user space swap library, as will be further described below.

<h2>Frequently asked questions</h2>

Frequently asked questions by students for this assignment will be answered <a href="https://docs.google.com/document/d/e/2PACX-1vRw6yHlTJElEuCF03eTrLRo_0Td_JD3mn5UkK41O4X997RZrzhlhcdNTkyJWYhU2UPkVtBrXT0ZV2U5/pub">here</a><a href="https://docs.google.com/document/d/e/2PACX-1vRw6yHlTJElEuCF03eTrLRo_0Td_JD3mn5UkK41O4X997RZrzhlhcdNTkyJWYhU2UPkVtBrXT0ZV2U5/pub">.</a> The most recent questions will be added at the top; questions will be dated. <strong>Check this file before asking questions.</strong> If you have questions regarding the assignment, please ask on the LumiNUS forum created for this assignment.

<h2>Reminder for Windows, macOS, and other non-Linux POSIX OS users</h2>

This lab uses Linux-specific behaviours, including the ability to return normally from a SIGSEGV handler and certain syscall arguments and operations. Your program may compile successfully on macOS or other POSIX OSes, but is unlikely to behave correctly.

If you are on Windows, the lab should work fine in WSL 2 (given that WSL 2 is just Linux), but it may not work in WSL 1.

Regardless, you are strongly encouraged to test your implementation on the SoC Compute Cluster xcne nodes, as that is where grading will be performed.




<h1>1.   Overall specifications</h1>

The overall specifications for the library are as follows. If you are up for a challenge, you may implement the lab solely based on these specifications; otherwise, they are broken down into exercises below. In either case, please read the specification carefully, as it defines key terms that will be used in the rest of the document.

<h2>Controlled memory regions</h2>

A controlled memory region is a memory region controlled by the user space swap library.

In the rest of the document, we use <strong>page fault</strong> to denote when a memory access to a controlled memory region causes a SIGSEGV. Note that this is <strong>independent</strong> of whether the memory access causes an <em>actual</em> page fault in the kernel.

A page of memory in a controlled memory region has these properties:

<ul>

 <li><strong>Residency</strong>: A <strong>non-resident</strong> page is not present in memory, and accessing it will cause a page fault. A <strong>resident</strong> page is present in memory, and must not cause a page fault on access.</li>

 <li><strong>Contents</strong>: Each page of memory has some contents. These contents are independent of the residency of the page, and must be preserved when a page is evicted and then brought back into memory.</li>

 <li><strong>Dirtiness</strong>: A page is <strong>dirty</strong> if it has been written to since the last time the page was made resident; otherwise, the page is <strong>clean</strong>.</li>

</ul>

A controlled memory region is allocated as private anonymous pages using mmap. The pages should initially be in the <strong>non-resident</strong> state. If the memory region is allocated using userswap_alloc, then the pages’ contents are initialised to zero. If the region is allocated using userswap_map, they should contain the contents of the backing file at the corresponding locations.

Upon a page fault accessing a <strong>non-resident</strong> page, the page should be brought into memory and thereby made resident. If doing so will cause the total size of resident memory in all controlled memory regions to exceed the LORM (Limit Of Resident Memory, defined in a section below), then <em>before</em> the new page is brought into memory, the <strong>minimum</strong> number of pages should be evicted according to the page eviction algorithm until the total size after the new page is brought in will be under or equal to the LORM.

If a dirty page in an allocation by userswap_alloc is evicted from memory, the contents should be saved in a swap file. Each process should have at most one swap file regardless of the number of allocations made. The swap file should be named &lt;pid&gt;.swap, in the current working directory of the process, where &lt;pid&gt; is the PID of the process (e.g., 12345.swap if the PID is 12345). The layout of the swap file is up to you, but its size must not exceed the maximum value since the start of execution of the total amount of memory in all unfreed controlled memory regions that were allocated by userswap_alloc. There is no need to shrink swap files after allocations are freed, <strong>but</strong> the swap file must not be extended if there is sufficient space already, e.g., due to previous allocations that have since been freed. Your implementation should be able to deal with an existing swap file; it is sufficient to just truncate or delete the existing file.

If a dirty page in an allocation by userswap_map is evicted from memory, its contents should be updated in the backing file in the corresponding location.

If a <strong>clean page</strong> is evicted from memory, <strong>no file writes should be done due to that eviction. </strong>This is because if the page is already in a swap file or a backing file and it is clean, its contents in the file should be identical to that  in memory. Hence no writes are needed. Note that a page that was allocated by userswap_alloc and that is read from but never written to, and therefore containing all zeroes, is considered a clean page.

When a page is evicted from memory, the kernel must be advised to actually free the backing physical pages by using madvise with MADV_DONTNEED; see the man pages for madvise.

<h2>SIGSEGV handler</h2>

The signal handler should check if the faulting memory access is to a controlled memory region. If so, the fault should be handled as described above. If not, it should <strong>remove itself as a </strong><strong>SIGSEGV signal handler</strong>, reset the action taken for a SIGSEGV signal, and return immediately, in order to allow the program to crash as it would without the user space swap library.

<h2>Page eviction algorithm</h2>

A first-in-first-out eviction algorithm should be used. That is, when a page is to be evicted, the page that was least recently made resident should be chosen for eviction. This applies <strong>across all</strong> controlled memory regions i.e., this is a global replacement algorithm.

<h2>Limit of resident memory in all controlled memory regions (LORM)</h2>

This is a global setting that can be controlled by userswap_set_size. The limit applies to all controlled memory regions in <strong>total</strong>, not per-region. The default value of the LORM should be 8,626,176 bytes (that is, 2,106 pages :-)), i.e., this is the value of the LORM prior to any calls to userswap_set_size.




<h2>Function specifications</h2>

<ul>

 <li>void *userswap_alloc(size_t size);</li>

</ul>

This function should allocate size bytes of memory that is controlled by the swap scheme described above in the “Controlled memory regions” section, and return a pointer to the start of the memory.

If size is not a multiple of the page size, size should be rounded up to the next multiple of the page size.

This function may be called multiple times without any intervening userswap_free, i.e., there may be multiple memory allocations active at any given time.

If the SIGSEGV handler has not yet been installed when this function is called, then this function should do so.

<ul>

 <li>void *userswap_map(int fd, size_t size);</li>

</ul>

This function should map the first size bytes of the file open in the file descriptor fd, using the swap scheme described above in the “Controlled memory regions” section.

If size is not a multiple of the page size, size should be rounded up to the next multiple of the page size.

The file shall be known as the <strong>backing file</strong>. fd can be assumed to be a valid file descriptor opened in read-write mode using the open syscall, but no assumptions should be made as to the current offset of the file descriptor. The file descriptor, once handed to userswap_map, can be assumed to be fully controlled by your library, i.e., no other code will perform operations using the file descriptor.

If the file is shorter than size bytes, then this function should also cause the file to be zero-filled to size bytes.

Like userswap_alloc, this function may be called multiple times without any intervening userswap_free, i.e., there may be multiple memory allocations active at any given time.

If the SIGSEGV handler has not yet been installed when this function is called, then this function should do so.

<ul>

 <li>void userswap_free(void *mem);</li>

</ul>

This function should free the block of memory starting from mem.

mem can be assumed to be a pointer previously returned by userswap_alloc or userswap_map, and that has not been previously freed.

If the memory region was allocated by userswap_map, then any changes made to the memory region must be written to the file accordingly. The file descriptor should <strong>not</strong> be closed.

<ul>

 <li>void userswap_set_size(size_t size);</li>

</ul>

This function sets the LORM to size.

If size is not a multiple of the page size, size should be rounded up to the next multiple of the page size.

If the total size of resident memory in all controlled regions is above the new LORM, then the <strong>minimum</strong> number of pages should be evicted according to the page eviction algorithm until the total size is under or equal to the LORM.

<h2>Assumptions</h2>

You may assume that:

<ul>

 <li>The size of a page is 4096. You <em>may</em>, but are not required to, ignore the high 16 bits of virtual addresses when designing your data structures.</li>

 <li>All arguments passed to the userswap functions are valid.</li>

 <li>All system calls made with valid arguments succeed. Despite this, you should still check the success of every system call, and emit some warning, as they may fail during development due to programmer error, e.g., providing wrong arguments. Omitting error-checking may lead to difficult-to-debug issues.</li>

 <li>There will be no concurrent accesses to controlled memory regions. (Relaxed in the bonus exercise.)</li>

 <li>There will be no attempts to execute memory in a controlled memory region.</li>

 <li>No other code will alter the memory protection of controlled memory regions using mprotect, or perform operations on those regions using madvise, etc.</li>

</ul>

<h2>Limitations</h2>

<ul>

 <li>Your implementation should have no more than about 128 bytes of overhead (i.e., used for metadata tracking the state of each page) per page of memory in a single allocation. It is okay to exceed this limit for small allocations below 512 pages. This requirement will not be strictly checked, but egregious cases may be penalised.</li>

 <li>You <strong>must not</strong> use the mmap syscall to map a file into memory to perform file I/O. That is, when you use mmap, the fd argument must always be -1, even for userswap_map.</li>

 <li>Your implementation should work with reasonable performance. We will not be grading based on performance (except for the bonus), but all provided test workloads should still run within reasonable time (i.e., less than 10 seconds).</li>

</ul>

<h1>2.   Implementation guide</h1>

Please read the overall specifications before reading this section. Each successive exercise builds upon previous exercises; when you complete all exercises, you will have fulfilled the specifications above.

In the lab archive, you will find these files:

<ul>

 <li>c: A skeleton for you to start from. You should implement everything in this file.</li>

 <li>h: Prewritten header; defines function prototypes.</li>

 <li>Makefile: Prewritten Makefile.</li>

 <li>workload_*.c: Provided workloads to test your library, as described above.</li>

</ul>

You should modify <strong>only</strong> userswap.c. If you write additional workloads for your own testing, you may wish to add them to the Makefile for your convenience.

<h2>Compiling, running and testing</h2>

To compile the workloads with your library, simply type make. The -Werror flag is set. You may remove it while developing, but you are strongly advised to fix all warnings before submitting, as warnings are indicative of undefined behaviour. When grading, we will do so <em>without</em> -Werror, but all other flags will be kept as-is.

In each exercise below, workloads are suggested to test the functionality in that exercise. All workloads are self-contained and require no input, so you can simply run the workload and inspect the output/result. Note that the suggested workloads are in no way exhaustive tests; passing them does not guarantee full credit.

<ul>

 <li>workload_readonly: This workload simply allocates an array and then reads each element of the array as an integer, sums all the elements, and then prints the sum.</li>

 <li>workload_wraddr: This workload simply allocates an array of pointers, and then writes the address of the array element to each element itself. It then goes back to read each element of the array, verifying each element as it reads. No output is printed if it succeeds.</li>

 <li>workload_rdfile: This workload creates a file with some data, and then maps the file and verifies that the contents of the mapping are identical to those of the file.</li>

 <li>workload_wrfile: This workload does the same as workload_rdfile, then it writes to the mapping, frees the mapping and then verifies that the file contents are updated.</li>

</ul>

A tip for debugging: GDB by default breaks on SIGSEGV even though there is a signal handler for it. To disable this, use the GDB command handle SIGSEGV nostop. You can also make GDB not print on each SIGSEGV by handle SIGSEGV noprint.




<h2>2.1.  Exercise 0 [Optional 1% demo]</h2>

To get started, in this exercise, you will simply get to a base upon which you can build the rest of the library.

<ol>

 <li>Implement userswap_alloc to simply allocate the requested amount of memory (rounded up as needed; see the specifications) using mmap. Per the specifications, the memory should be initially non-resident and therefore should be allocated as PROT_NONE, in order for any accesses to the memory to cause a page fault. userswap_alloc should also install the SIGSEGV handler, if it has not already been done.</li>

 <li>Implement userswap_free, which should free the entire allocation starting at the provided address using munmap.</li>

</ol>

o You will need to track the size of each allocation in userswap_alloc. A simple linked list of allocations, storing the start address (from mmap) and the size, will be sufficient, but you are free to design and use more performant structures.

<ol start="3">

 <li>Write a SIGSEGV For this exercise, the handler does not need to check whether the faulting memory address is within a controlled memory region; it can simply call the page fault handler. The page fault handler will need the address to perform mprotect, however; the faulting memory address can be found in <a href="https://man.archlinux.org/man/sigaction.2#The_siginfo_t_argument_to_a_SA_SIGINFO_handler">the </a><a href="https://man.archlinux.org/man/sigaction.2#The_siginfo_t_argument_to_a_SA_SIGINFO_handler">siginfo_t</a><a href="https://man.archlinux.org/man/sigaction.2#The_siginfo_t_argument_to_a_SA_SIGINFO_handler"> struct passed to the signal handler</a><a href="https://man.archlinux.org/man/sigaction.2#The_siginfo_t_argument_to_a_SA_SIGINFO_handler">.</a></li>

 <li>Write a function to handle page faults. This is where the bulk of the logic for the “Controlled memory regions” will reside. For this exercise, the page fault handler only needs to use mprotect to make the page containing the accessed memory PROT_READ, thereby making the page resident. o Note that the specifications say that memory newly allocated by userswap_alloc should be initialised to zero. Nothing needs to be done for this, as memory allocated by mmap is always initialised to zero.</li>

</ol>

<strong>Suggested test workloads</strong>: workload_readonly

<strong>Syscall hints</strong>: mmap, mprotect, munmap, sigaction




<h2>2.2.  Exercise 1</h2>

In this exercise, you will implement more of the basic functionality of the library, stopping short of page eviction.

<ol>

 <li>Extend the SIGSEGV handler to verify that the faulting memory address is actually in a controlled memory region, and if not, it should reset the action for the SIGSEGV signal and return. This completes the functionality for the SIGSEGV</li>

 <li>Extend the page fault handler so that, upon a write, the accessed page is made PROT_READ | PROT_WRITE. In the case where non-resident memory is written to, it is acceptable to take two page faults, where the first makes the page PROT_READ, and the second makes the page PROT_READ | PROT_WRITE. To do this, you will need to track the state of each page; if a page fault happens on a resident page, the only possibility (given the assumptions) is that a write was attempted on a PROT_READ</li>

</ol>

o Pages are only made PROT_WRITE on the second fault so that it is possible to tell which pages are dirty, which will be needed for later exercises. o It is also possible to directly figure out if a SIGSEGV was caused by a read or write, but this is not required.

<ol start="3">

 <li>Extend userswap_free so that it cleans up the data structures corresponding to the allocation as well.</li>

</ol>

Here, you will need to design data structures to track the state of pages. You are free to design them as you wish. Remember that there can be multiple controlled memory regions. Keep in mind the overall requirements (i.e., the requirements of subsequent exercises) when designing these data structures. Also take note of the limit on memory overhead as mentioned above, although it is unlikely that you will hit the limit unintentionally.

Note that the data structures you create for this lab will be slightly different from those used by an OS in that they do not map logical or virtual addresses to physical addresses; they just store the state of pages.

Here are some suggestions for data structures:

<ul>

 <li>A <a href="https://os.phil-opp.com/paging-introduction/#paging-on-x86-64">4-level page table</a><a href="https://os.phil-opp.com/paging-introduction/#paging-on-x86-64">,</a> mirroring the structure used by x86_64. (The assumption above that allows you to ignore the high 16 bits of virtual addresses is relevant for this structure.) This has constant-time lookup and update once every level of page table is initialised for a particular address.</li>

 <li>Some kind of list of allocations that contains a list of pages in each allocation. The lookup and update performance will depend on how the list of allocations is designed.</li>

</ul>

<strong>Suggested test workloads</strong>: workload_wraddr

<strong>Syscall hints</strong>: No new syscalls needed.

<h2>2.3.  Exercise 2</h2>

In this exercise, you will implement page eviction, but without swap.

<ol>

 <li>Extend the page fault handler so that, if the LORM will be exceeded when making a page resident, a page should be evicted according to the page eviction algorithm <em>before</em> the new page is made resident. In this exercise, it will be sufficient to make the page PROT_NONE, without performing madvise(MADV_DONTNEED); therefore, a swap file is not needed in this exercise.</li>

 <li>Implement the userswap_set_size</li>

</ol>

You will likely need an additional data structure for the page eviction algorithm. Some form of queue of resident pages, such as a doubly linked list or dynamic array will work well here. Remember to extend userswap_free so that it updates that data structure as well when an allocation is freed.

<strong>Suggested test workloads</strong>: workload_wraddr

<strong>Syscall hints</strong>: No new syscalls needed.




<h2>2.4.  Exercise 3</h2>

In this exercise, you will implement swap. Extend the page fault handler so that:

<ol>

 <li>when a page is evicted <strong>and</strong> it has been modified since it was last brought into memory (i.e., the page is <strong>dirty</strong>), its new contents are written to a swap file, and then the physical page is freed by calling madvise(MADV_DONTNEED) on the page, in addition to it being set to PROT_NONE. The order of madvise(MADV_DONTNEED) and mprotect(PROT_NONE) does not matter. The swap file should be named and placed accordingly as described in the overall specifications, but the structure of the swap file is up to you to design.</li>

 <li>when a non-resident page is brought into memory, and it has been previously evicted into a swap file (i.e., not a freshly allocated page), its contents are restored from the swap file.</li>

</ol>

When designing your swap file format, keep in mind that multiple separate allocations may exist simultaneously, but there may only be one swap file for the entire process.

In general, a swap file does not need to have a particular structure and can simply be a file containing page contents. When a page is evicted from memory, you will need to find a place in the swap file to evict it to, and track the location so that you can read the page back into memory later on. The most straightforward place to record this information is in your page table or similar structure from exercise 1. You will also need to keep track of locations in the swap file corresponding to pages that have been freed, so that those locations can be reused. A simple linked list, or other structure, of free swap file locations will suffice.

You should avoid using buffered I/O for reads and writes to your swap file, as that will defeat the purpose of the library, since memory would be allocated for the buffers and evicted pages stored in those buffers. You <em>can</em> use the stdio functions if you wish, as long as you disable buffering on the swap file, although we would suggest using the direct syscalls.

There is no need to close the swap file. At the same time, your implementation should be able to deal with an already existing swap file; it is sufficient to just truncate or delete the existing file if one is seen (since the file is uniquely named by PID), or do nothing if your implementation can ignore the existing data.  <strong>Suggested test workloads</strong>: workload_wraddr

<strong>Syscall hints</strong>: madvise, open, read/pread, write/pwrite




<h2>2.5.  Exercise 4</h2>

In this exercise, you will implement userswap_map, i.e., user space memory-mapped files, but for reading only.

<ol>

 <li>Implement userswap_map, which is mostly the same as userswap_alloc, except that it needs to record that this allocation is backed by the file given. It should also length-extend the file as needed, as described in the specifications.</li>

 <li>Extend the page fault handler so that, upon a read to a non-resident page allocated by userswap_map, the page is filled with the contents of the file at the corresponding location. Also, when a page allocated by userswap_map is evicted, for this exercise, it is sufficient to just discard the page (i.e., madvise(MADV_DONTNEED) and mprotect(PROT_NONE)) without storing the contents anywhere.</li>

</ol>

<strong>Suggested test workloads</strong>: workload_rdfile

<strong>Syscall hints</strong>: pread, fstat, ftruncate







<h2>2.6.  Exercise 5</h2>

In this exercise, you will implement write support for your user space memory-mapped files functionality.

<ol>

 <li>Extend the page fault handler so that, when a page that was allocated by userswap_map, and that has been modified since it was last brought into memory, is evicted, its contents are written back to the corresponding location in the backing file. If the page was not modified, it should simply be discarded, as in the previous exercise.</li>

 <li>Extend userswap_free so that any dirty pages are written back to the backing file.</li>

</ol>

<strong>Suggested test workloads</strong>: workload_wrfile

<strong>Syscall hints</strong>: pwrite




<h2>2.7.  Exercise 6</h2>

In this exercise, you will make this library be able to support <strong>concurrent reads and writes</strong> to controlled memory regions.

You may use any synchronisation mechanism that works on the Compute Cluster. The bonus score you receive will depend on the performance and quality of your synchronisation mechanism; for example, a simple mechanism like a global mutex on the page fault handler may work, but will receive little credit.

Implement this exercise in a copy of userswap.c named bonus_userswap.c. Ex1–5 will be graded using userswap.c, and Ex6 will be graded using bonus_userswap.c; of course, your bonus implementation should be able to meet Ex1–5’s requirements as well. Include in a text file bonus_userswap.txt a short description of the idea behind your synchronisation mechanism.


