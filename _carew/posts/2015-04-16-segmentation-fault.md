---
layout: post
title:  Segmentation Fault (Computer memory layout)
---

## That word : segmentation fault

A quick word on what's this blog post is about.

When I make mistakes in my code, it very often generates a "segmentation fault" message, often shorten as "segfault". And then, my collegues and managers come to me exclaming "ha!, we got a segfault for you (to fix)".
"All right, that's mine" is often my answer. But, do you really know what a "segmentation fault" is ?.

In addition to that, I could plan in the future to write technical in-depth PHP-memory-related articles, I need my readers to have such knowledge to understand some concepts further explained.

To answer such a question, we'll have to rewind time back in 1960's. I'm going to explain you how a computer works, more accurately how memory is accessed into a modern computer and you'll then understand where this strange message comes from.

What you'll read here is a summary of the basics of computer architecture design, I won't go too far if it's not necessary, and will use well known wordings so that anyone working with a computer everyday can understand such crucial concepts about how the machine works.

Many books exist about computer architecture, if you want to go further into such a subject, I suggest you get some of them and start reading. Also, don't forget to grab one OS Kernel source code and study it, such as the Linux Kernel, or any else.

> We won't rewrite the history here. This post is detailed somehow, but not a full computer architecture class ; some info are missing or (sometimes extremely) simplified.

## Some computer science history

Back in the time, computer were enormous machines, weighting tons, inside what you could find one CPU with something like 16Kb of memory (RAM). We won't go back in time further, that is before the CPU chips era.

So in this period, a computer cost something like $150,000 and could execute exactly one task at a time : it could run one process at a given time.
If we would draw a schema of the memory architecure then, we would draw it like this :

![early_systems](../../../img/segfault/early_systems.png)

As you can see, the memory is 16Kb wide, and into it you can find two things :
*	The Operating System memory area, which is 4Kb wide (f.e)
*	The actually running process memory area, which is then 12Kb wide (f.e)

The operating system role was, at this time, to simply manage the hardware using CPU interruptions. So the OS needs memory for itself, to copy the data from a device f.e and treat them (PIO mode). Just to display something on the screen needs some main memory, back in this time where video chipsets bundled zero to few kilobytes of memory.

And then, our solo program was running just next to the OS memory, achieving its tasks.

We are fine with that, right ?

## Sharing the machine

But one big problem of such a model is that the machine (costing $150,000) could only perform one task at a time, and this task was terribly long to run : entire days, to treat few Kb of data.

At this huge price, it was clearly not possible to buy several machines to perform several treatments at the same time, so people was trying to share the machine resources.
The time of *multitasking* was born. Keep in mind that it is way too early to talk about multi-CPU machines. How can we make one machine, with one CPU, actually achieve several different tasks ?

The answer was *scheduling* : when one process was busy waiting for an I/O to come through an interrupt, the CPU would then run another process during this time. We won't talk about scheduling at all (a too wide subject), but about memory.

If a machine can run several process tasks one after the other, then that means that memory would then look something like this (f.e) :

![multitasking_memory](../../../img/segfault/multitasking_memory.png)

Both task A and task B memory are stored into the RAM, because copying it back and forth to a disk is a way too heavy process : the data have to stay into RAM, permanently, as their respective tasks still run.

Cool : the scheduler gives some CPU time to A, then B, etc... each accessing its memory area. But wait, there is a problem here.

When one programmer would write the task B's code, it then had to know the memory address bounds. Task B for example, would start from memory 10Kb to memory 12Kb, so every address in the program needs to be hardcoded exactly into those bounds, and if the machine could then run 3 tasks, then the address space would be divided in even more areas, and the memory bounds of task B would then move : B's programmer had to rewrite his program, to use less memory this time, and to rewrite every memory pointer address.

Another problem is clear here : what if task B accessed memory in the task A area ? It is really easy to do so, because when you manipulate pointers, a little mistake in computation lead to a totally different address : task B can access task A's memory and corrupt it (overwrite it). Security as well : what if task A is a very sensitive program playing with very sensitive data ? There is no way to prevent task B from reading task A's memory area.
And last : what if task B does a mistake, and overwrite the OS memory ? From 0Kb to 4Kb, here, it is OS memory, if task B overwrites it, then the OS will crash for sure.

## Address Space

So, to be able to run several tasks residing in memory, the OS and the hardware must help. One way of helping, is by creating what's called an address space.
An address space is an abstraction of the memory that the OS will give to a process, and this concept is absolutely fundamental, as nowadays, every piece of computer you meet in your life, is designed like that. There exists no other model (in the civilian society, army may retain secrets). You use your personnal computer, you keep typing on your mobile phone's screen, you start your TV, you play with your video game console, and you put your credit card in a cash machine and you ridiculously talk to your glasses or your watch.

> Every system nowadays is organized with a code-stack-heap memory layout like this, in its deepnesses.

The address space contains everything a task (a process) will need to run :
*	Its code : its machine instructions the CPU will have to run
*	Its data : the data the machine instructions will play with

The address space is divided like this :

![process_address_space](../../../img/segfault/process_address_space.png)

*	The _stack_ is the memory area where the program keeps informations about called functions, their arguments and every local variable into the functions.
*	The _heap_ is the memory area where the program does whatever it wants to, the programmer is absolutely free to do anything he wants in such area.
*	The _code_ is the memory area where the CPU instructions of the compiled program will be stored. Those instructions are generated by a compiler, but can be hand written. Note that the code segment is usually divided itself in three parts (Text, Data and BSS), but we don't need to go that far in our analyze.

*	The _code_ is fixed size : it is born from a compiler, will weight (in our example picture) 1Kb, and that's it.
*	The _stack_ however is a resizable zone, as the program runs. When functions get called, the stack expands, when function calls are terminated : the stack expands backwards.
*	The _heap_ as well is a resizable zone, as the programmer asks memory from the heap (`malloc()`), this latter will expand forwards, and when the programmer frees back memory (`free()`), the heap narrows.

As the stack and the heap are expandable zones, they've been located at opposite locations into the whole address space : the stack will grow backwards, and the heap forwards. They are both fully free to grow, each one in the direction of the other. What the OS will have to check is simply that the zones don't overlap at some time, using limits mainly.

## Memory virtualization

If task A has got an address space like the one we saw, and so task B... How can we fit both of them into memory ? That seems weird. Task A address space starts from 0kb to 16Kb and so for task B. Huh...

The trick relies in **virtualization**.

In fact, here is back the picture of both A and B in memory :

![multitasking_memory](../../../img/segfault/multitasking_memory.png)

When task A will try to access memory into its own address space at index, let's say 11K, probably somewhere into its own stack, the OS will have to find a trick to actually not load memory index 1500, because in memory, index 11K leads to task B.

In fact, the whole address space every program thinks is memory, is just **virtual memory**. *Everything is fake*.
In task A memory, memory index for example 11K, is just a fake address. This is a virtual memory address.

**Every program running on the machine plays with fake, virtual address**. With the help of some hardware chips, the OS will trick the process when this latter will try to access any zone of memory.

The OS will virtualize memory, and will guarantee every task that it can't access memory is doesn't own : the virtualization of memory has allowed process isolations : task A can't access task B's memory anymore, nether can it access the OS own memory. And everything is totally transparent to the user-level tasks, thanks to tons of complex OS Kernel code.

Thus, the OS will have to come on scene for every process memory demand. It will then have to be very efficient - not to slow down too much the different programs running - and to achieve this it will get helped by the hardware : mainly by the CPU and some electronical device around it, like the MMU (Memory Management Unit).
MMU appeared then in early 70's, with IBM, as separated chips. They are now embed directly into our CPU chips, and are mandatory for any modern OS to run.
In fact, the OS won't have tons of things to do, it will heavilly rely on some hardware specific behaviors that will ease a lot every memory access.

Here is a little C program showing some memory addresses :

	#include <stdio.h>
	#include <stdlib.h>

	int main(int argc, char **argv)
	{
		int v = 3;
		printf("Code is at %p \n", (void *)main);
		printf("Stack is at %p \n", (void *)&v);
		printf("Heap is at %p \n", malloc(8));
	
		return 0;
	}

On my LP64 X86_64 machine, it shows :

	Code is at 0x40054c 
	Stack is at 0x7ffe60a1465c 
	Heap is at 0x1ecf010

We can see that the stack is at a very higher address than the heap, and the code is located before the stack, just like we described.

But, every of those 3 addresses are fakes : in the physical memory, at address 0x7ffe60a1465c, there is absolutely not an integer, which value is 3.
Remember, every user-land program manipulates virtual memory addresses, whereas kernel-land programs such as the OS kernel itself (or its hardware drivers code) may manipulate the physical RAM addresses.

## Translating addresses

**Address translation** is the wording behind such a magical technic. The hardware (MMU) will translate every virtual address from a user-land program into the right physical address. That's all.

Thus, the OS will have to remember, for every task running, the correspondence between every of its virtual address, to the physical address. And this is challenging. The OS will have to manage every user-level task memory for every memory-access demand, thus providing a full illusion to the process. Hence, the OS transforms all the physical memory awful reality into a useful, powerful and easy abstraction.

Let's detail that in a simple scenario :

When a process is launched, the OS will book a fixed area of physical memory, let's say 16Kb. It will then save the starting address of such a space, in a special variable called the *base*. It will set another special variable, called the *bounds* (or limit) to the width of the space : 16Kb.
The OS will then save those two values into each process table, called the PCB (Process Control Block).

Now, here is a process virtual address space :

![process_address_space](../../../img/segfault/process_address_space.png)

And here is its physical image :

![memory_virt](../../../img/segfault/memory_virt.png)

The OS chose to store it into physical memory at address range 4K to 20K, thus the base address is set to 4K and the bound/limit is set to 4+16 = 20K.
When this process is scheduled (given some CPU time), the OS reads back both the *bound* and *limit* values from the PCB, and copies them into specific CPU registers.
Then the process will run and will try to load for example its virtual address 2K (something into its heap). The CPU will then add to this address the base it has received from the OS. Thus, this process memory access will lead to the physical location 2K + 4K = 6K.

**physical address = virtual address + base**

If the resulting physical address (6K) is out of the bounds ( -4K|20K- ), that means that the process tried to access invalid memory that it doesnt own : the CPU will then generate an exception, and as the OS did set up an exception handler for that, the OS will be triggered back by the CPU and knows a memory exception has just happened onto the CPU. Its default behavior is then to issue a signal to the faulty processk : a SIGSEGV : a Segmentation Fault, which by default (this can be changed) will terminate the task : the process has crashed about invalid memory access.


### Memory Relocation

Even better, if task A is unscheduled, that means it is taken out from the CPU because the scheduler asked to run another task now (say task B), when running task B, the OS is free to relocate the entire physical address space of task A.
The OS is given the CPU often when a user task runs. When this latter issues a system call, the control of the CPU is given back to the OS, and before honnoring the system call, the OS is free to do whatever it wants about memory management, like relocating an entire process space into a different physical memory slot.

For this, it is relatively simple : the OS moves the 16K wide area into another 16K wide free space, and simply updates the base and bound variables of the task A. When this latter will be resumed and given back the CPU, the address translation process still works, but it doesn't lead to the same physical address as before.

Task A has noticed nothing, from its point of view, its own address space still starts from 0K up to 16K. The OS and the hardware MMU take full control over every memory access for task A, and they have thrown a complete illusion to it. The programmer behind task A manipulates its allowed virtual addresses, from 0 to 16, and the MMU behind will take care of positionning everything into physical memory.

The memory image, after the move, would then look like this :

![process_relocation](../../../img/segfault/process_relocation.png)

It has become very easy to program memory now : a programmer no longer has to wonder where its program will be located in RAM, if another task will run next to his own, and what memory addresses to manipulate : it is taken by hand by the OS, helped with very performant and clever hardware : the Memory Management Unit (MMU).

## Memory segmentation

Notice the appearance of the "segmentation" word : we are close to the explaination of "why 'segmentation fault'".

In the last chapter, we explained about memory translation and rellocation, but the model we used has drawbacks :

*	We assumed every virtual address space is fixed 16Kb wide. This is obviously not the case in reality.
*	The OS has to maintain a list of physical memory free slots (16Kb wide), to be able to find a place for any new process asking to start, or to relocating a started process. How to do this efficiently to not slow down all the system ?
*	As you can see, every process memory image is 16Kb wide, even if the process doesn't use its whole address space, which is very likely. This model clearly wastes a lot of memory, as if a process really consumes say 1KB of memory, its memory image in physical RAM holds 16Kb. Such a waste is called *internal fragmentation* : the memory is reserved, but never used.

To address some of those problems, we're gonna dive into more complex memory organization from the OS : segmentation.

Segmentation is easy to understand : we widen the concept of "base and bounds" to the three logicial segments of memory : the heap, the code and the stack, of each process - instead of just cosiderating the memory image as one unique entity.

With such a concept, the wasted memory between the stack and the heap, is no longer wasted. Like this :

![basic_segments](../../../img/segfault/basic_segments.png)

Now it is easy to notice that the free space in virtual memory for task A is no longer allocated into physical memory, the memory usage has become much more efficient.

The only difference now is that for any task, the OS doesn't have to remember one couple base/bounds, but three of them : one couple for each segment type. The MMU takes care of the translations, just like before, and now supports as well 3 *base* values and 3 *bounds*.

For example, here, the task A's heap has a base of 126K and a bound of 2K. When the task A asks to access its virtual address 3Kb, into its heap; the physical address is 3Kb - 2Kb (start of the heap) = 1Kb + 126K (offset) = 127K. 127K is before 128K : this is a valid memory address that can be honored.

### Segment sharing

Segmenting the physical memory not only solves the problem of free virtual memory not eating any more physical memory, but it also allows to share physical segments through different processe virtual address spaces.

If you run twice the same task, task A for example, the code segment is exactly the same : both tasks run the same CPU instructions. While both tasks will use their own stack and heap, as they treat their own set of data, it is silly to duplicate the code segment of both in memory. The OS can now share it, and save even more memory. Something like this :

![shared_seg](../../../img/segfault/shared_seg.png)

On the picture above, both A and B have their own code area into their respective virtual memory space, but under the hood, the OS shared this area into the same physical memory segments.
Both A and B are once more fooled : they absolutely don't see this, for both of them, they own their memory.

To achieve this, the OS has to implement one more feature : segment protection bits.

The OS will, for each physical segment it creates, register the bound/limit for the MMU translation unit to work correctly, but it will also register a permission flag.

As the code is not modifiable, the code segments are all created with the RX permission flag. The process can load those memory areas for eXecution, but any process trying to Write into such memory area, will be shout at by the OS.
The other two segments : heap and stack are RW, the processes can read and write from their own stack/heap , but they can't execute code from it (this prevents program security flaws, where a bad user may want to corrupt the heap or the stack to inject code to run, mainly to be given root access : this is not possible as the heap and stack segments often are not eXecutable. Note that this has not always been the case in history and requires some more hardware support to work efficiently, this is called the "NX bit" under Intel CPU).

Memory segments permissions are changeable at runtime : the task may call for [mprotect()](http://man7.org/linux/man-pages/man2/mprotect.2.html) from the OS.

Those memory segments are clearly visible under Linux, use */proc/{pid}/maps* or the */usr/bin/pmap* utility

Here is an example for PHP :

	$ pmap -x 31329
	0000000000400000   10300    2004       0 r-x--  php
	000000000100e000     832     460      76 rw---  php
	00000000010de000     148      72      72 rw---    [ anon ]
	000000000197a000    2784    2696    2696 rw---    [ anon ]
	00007ff772bc4000      12      12       0 r-x--  libuuid.so.0.0.0
	00007ff772bc7000    1020       0       0 -----  libuuid.so.0.0.0
	00007ff772cc6000       4       4       4 rw---  libuuid.so.0.0.0
	... ...

Here, we have a detail of all the memory mappings. Addresses are virtual, and every memory area permissions are displayed.
Like we can see, every shared object (.so) is mapped into the address space as several mappings (likely code and data), and the code areas are eXecutable, and behind the hood, they are shared into physical memory between every process having mapped such a shared object into its own address space.

This is one enormous advantage of Shared Objects under Linux (and Unixes) : memory savings.

It is possible also to create a shared area leading to a shared physical memory segment, by using the [mmap()](http://man7.org/linux/man-pages/man2/mmap.2.html) system call. A 's' letter will then appear next to the area, standing for "shared".

### Segmentation limits

We've seen that segmentation fixed the problem of unused virtual memory space. When a process doesn't use some memory space, this latter is not mapped into physical memory, thanks to segments, that correspond to used memory.

However, that is not entirely true.

What if a process asks for 16Kb of heap ? The OS will likely create a 16Kb wide segment in physical memory. But if the user then frees 2Kb of such memory ? Here, the OS has to shrink down the 16Kb segment, back to 14Kb. What if now the programmer asks for suddenly 30Kb of heap ? The old 14Kb segment now has to grow to 30Kb, but can it do so ? Other segments may now surround our 14Kb segment, preventing it from growing. Then the OS will have to look for a free space of 30Kb, and relocate the segment.

![fragmented_memory](../../../img/segfault/fragmented_memory.png)

The big problem with segments, is that they lead to very fragmented physical memory, because they keep growing and shrinking as the user-land tasks ask for memory and release it. The OS then has to maintain a list of free memory holes, and manage them.

Sometimes, the OS by summing up all the free segments, has some space available, but as it is not contiguous, it can't use that space and must refuse memory demands from processes, even if there is space in physical memory. That is a really bad scenario.

The OS can try to compact the memory, by merging all the free areas into one big chunk that could be used in the future to satisfy a memory demand.

![compacted_memory](../../../img/segfault/compacted_memory.png)

But such compaction algorithm is really heavy for CPU, and in the meantime, no user process may be given the CPU : the OS is fully working reorganizing its physical memory, thus the system becomes unusable.

Memory segmentation addresses lots of problems about memory management and multitasking, but they also show real weaknesses. Thus, there is a need to enhance the segmentation capabilities and fix those flaws : this is done by another concept : *memory paging*.

## Memory pagination

Memory pagination shares some concepts with memory segmentation and tries to address segmentation problems. We saw that the main problem of memory segmentation is that segments will grow and shrink very often, as the user process asks and releases memory. Sometimes, the OS faces a problem in that it can't find a big enough free space to satisfy a user process memory ask, because the physical memory has been so much fragmented with time : it is full of many segments of different sizes, sparsed everywhere in physical memory leading to a highly fragmented physical memory.

Pagination solves this problem with a simple concept : what if every physical allocation the Kernel will perform is fixed-sized ?
Pages are physical memory fixed-size segments, nothing else.
If the OS uses fixed size allocations, it is way easier to manage the space, and the result is that there is no more physical fragmentation.

Let's show an example, once more assuming a tiny 16Kb wide virtual address space, to ease the representation : 

![paged_virtual_memory](../../../img/segfault/paged_virtual_memory.png)

With pagination, we don't talk about heap, stack or code segments, but we divide the whole process virtual memory into fixed size zones : we call them pages. On the example above, we divided the address space as 4 pages of 4Kb each.

Then, we do the same thing with physical memory.

![paged_physical_memory](../../../img/segfault/paged_physical_memory.png)

And the OS simply keeps in what's called the "process page table", the association between one process virtual memory page and the underlying physical page (which we call a "page frame").

![paged_memory_layout](../../../img/segfault/paged_memory_layout.png)

With this way of managing memory, there is no more problem in free space management : the page frame is weither mapped (used), or not ; and it is easy for the kernel to find enough pages to satisfy a user process memory request, it simply keeps a free list of page frames, and browse it at every memory demand.

The page is the smallest memory unit the OS can manage. A page (at our level) is indivisible.

With pages, every process is attached a page table that stores address translations. The translations no longer use the "bound" and "limit" values like it used to do with segmentation, but rather use a "virtual page number (VPN)" and an "offset" into this page.

Let's show an example of address translation with pagination. The virtual address space is 16Kb large, thus we need 14 bits to represent an address (2^14 = 16kb). The page size is 4Kb, so we need 4kb (16/4) to select our page :

![paged_address_layout](../../../img/segfault/paged_address_layout.png)

Now, when the process wants to load for example address 9438 (out of 16384 possibilities), it gives 10.0100.1101.1110 in binary, leading to this :

![paged_address_example](../../../img/segfault/paged_address_example.png)

That is the 1246th byte in virtual page 2 ("0100.1101.1110"th byte in "10"th page). Now, the OS simply has to lookup this process page table, to know what page 2 maps. According to what we suggested, page 2 is the 8k byte in physical memory.
Thus, virtual address 9438 leads to physical address 9442 (8k + 1246 offset). We found our data !

Like we said, there is one page table per process, because each process will have its own address translations, just like with segments. But wait, where are those page tables actually stored ? Guess... : in physical memory yes, where else could they be ?

If page tables are themselves stored into memory, that means that for every memory access, the memory will have to be accessed to fetch the VPN. So with pages, one memory access equals in fact two memory accesses : one to fetch the page table entry, and one to access the "real" data. Knowing that memory accesses are slow, that doesn't seem like a nice solution does it ?

## Translation-lookaside Buffer : TLB

Using paging as the core mechanism to support virtual memory can lead to high performance overheads. By chopping the address space into small, fixed-sized units (pages), paging requires a large amount of mapping information. Because that mapping information is generally stored in physical memory, paging logically requires an extra memory lookup for each virtual address generated by the program.

And here comes once more the hardware, to accelerate things and help the OS. In pagination, like in segmentation, the hardware will fire and help the OS Kernel to translate addresses in an efficient acceptable way. TLB is part of the MMU, and is just a simple cache of some VPN translations. TLB will prevent the OS from firing a memory access to access the process page table to get the physical memory address from the virtual one.

The hardware MMU will fire at every virtual memory access, extract the VPN from this address, and lookup the TLB if it has a hit for this specific VPN. If it has a hit, well, it just accomplished its role. If it gets a miss, then it will lookup the process page table, and if the memory reference is valid, it will update the TLB so that any further memory access of this page will find a hit in the TLB.

Like any cache, you'd better have a hit than a miss, as the miss situation will trigger a page lookup and will result in a slow memory access.
You'd guess as well that the bigger the pages are, the better it is for a TLB hit situation, but the more wasted space you'll experience in each page. A tradeoff must be found here. Modern kernels use different page sizes, like the Linux Kernel may use what's called "huge pages", pages that are 2Mb large instead of the traditionnal 4Kb large.

Also, you'd better store informations in contiguous memory addresses. If you sparse your data in memory, you'll more likely suffer from TLB misses or TLB beeing full. This is what's called spacial locality efficiency of the TLB : the data immeditely next to yours may be retained in the same physical page, thus benefit from the current memory access from the TLB.

For context switches, the TLB stores also what's called the ASID in each of its entries. This stands for the Address Space Identifier, which is something like a PID. Each process scheduled has its own ASID, then the TLB can manage any process address without risk of invalid addresses coming from other processes.

Here again, if the user process tries to load an invalid address, the address will likely not be stored into the TLB, hence a miss and a lookup into the process page table entry. Here, the address is stored (as every possible translation is stored) but with an invalid bit set. Under X86, each translation uses 4bytes, thus many bits in there are available, and it is not rare to find a valid bit, together with other complex stuff such as a dirty bit, the protection bits, a reference bit etc... If the entry is marked invalid, the OS will trigger by default a SIGSEGV, which leads to a "segmentation fault", even if here, we don't talk about segments anymore.

What you need to know is that paging is pretty more complex in modern OS that what I explained. Modern OS typically use multi level page table entries, multi page sizes as well as a crucial concept we won't explain : page eviction, known as "swaping" (a process where the Kernel exchange pages from memory to disk, to efficiently use main memory and give the illusion to user processes that main memory is unlimited in space).

## Conclusion

You now know what resides under the "segmentation fault" message. OSes used to use segments to map virtual memory space to physical memory space.
When a user land process wants to access some memory, it issues a demand that the MMU will translate to a physical memory address. But if this address is wrong : out of the bounds of the physical segment, or if the segment protection rights are not good (asking to write to a read-only segment), then the OS by default sends a signal to the faulting process : a SIGSEGV , which has a default handler that kills the process and outputs a message : "Segmentation fault". Under other OSes (guess), it is often reported as a "General protection fault". For Linux, we are lucky to be able to access the source code, here is the place of the source code, for X86/64 platforms, that manages memory access errors : [http://lxr.free-electrons.com/source/arch/x86/mm/fault.c](http://lxr.free-electrons.com/source/arch/x86/mm/fault.c) , and for SIGSEGV, its precisely [here](http://lxr.free-electrons.com/source/arch/x86/mm/fault.c#L731)

If you are interested in the design of segments for X86/64 platforms, you can [look at their definition in the Linux Kernel](http://lxr.free-electrons.com/source/arch/x86/include/asm/segment.h#L8). You will also find interesting stuff about memory paging, which pushes the segmentation of memory way further than classical segments usage.

I liked writing this article, it drove me back in late nineties, when I was programming my first CPUs : [Motorola 68HC11](http://www.freescale.com/files/microcontrollers/doc/data_sheet/M68HC11E.pdf) using C, VHDL and direct Assembly. I did not program a virtual memory management OS myself but used physical addresses directly (such a CPU did not need such complex mechanisms). Then I turned to Web ; but my first knowledge come from electronics, the systems we make use of everyday... Exciting.
