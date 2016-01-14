---
layout: post
title:  PHP memory and Zend Memory Manager (PHP 5)
---

##Introduction

This blog post is gonna introduce you the dynamic memory management layer PHP relies on : Zend Memory Manager (ZendMM). We'll detail why we need such a layer, what it does, how to customize it, how to interact with it from PHP land.

##Recall on C memory management

C has several allocation storage classes :

*	automatic allocation ;
*	static allocation ;
*	dynamic allocation.

Auto allocation is about function received arguments, or any variable declared into a function body. The compiler has all informations it needs to figure out how many bytes to allocate, and it will automatically free the memory by itself, when the container becomes out of scope. You cannot get your hands on such automatic feature, especially you can't reallocate this memory zone (because you'd need more memory that the compiler computed for you, or less).

Static allocation is about global or static variables. Like with automatic allocation, the compiler allocates memory depending on variable type, but this time it will never free it until the program ends. Here again, you cannot get into that process to customize it.

Finally dynamic allocation is where the programmer (you) will declare himself how much bytes of memory he needs. You can reallocate the memory, meaning you can enlarge it, or shrink it, whenever you want. This is necessary as many things in a program lifetime are not known at compile time, and evolve within the program lifetime. So you can do whatever you want with dynamic allocation, but there is a duty you have to accept : free the memory zone whenever you don't need it anymore, because absolutely noone will do it for you. If you forget, you create what's called a *memory leak*, that means you allocated some memory for your program, but you never released it back to the OS so that it may use it for any other program which may need it. That's bad.

![OS memory](../../../img/php-memory/os-memory.png)

Here is an example :

	#include <stdio.h>
	#include <stdlib.h>
	 
	/* Static allocation, global variable
	   the compiler takes care of everything, but won't free
	   this bloc of memory */
	int myint;
	 
	char *myfunction(int i)
	{
		/* Static allocation, static variable
	   the compiler takes care of everything, but won't free
	   this bloc of memory */
		static char *my_static = "Foo";
	 
		if (myint == i) {
		    my_static = "Bar";
		}
		return my_static;
	}
	 
	int main(int argc, char *argv[])
	{
		/* Automatic allocation, the compiler takes care of allocation
		and will free this memory bloc when the function will end */
		int *p_int;
		int i;
	 
		myint = 18;
	 
		/* Dynamic allocation. The programmer asks himself to allocate
		   10*sizeof(int) bytes in memory, he will have to free it by himself */
		p_int = (int *)malloc(10 * sizeof(int));
	 
		for (i=0; i<10; i++) {
		    *(p_int + i) = i*myint;
		    myfunction(*p_int);
		}
	 
		/* Free of dynamic memory bloc. Forgetting this stage creates a real
		  memory leak */
		free(p_int);
	 
		return 0;
	}
	
> We stop here. Just note that depending on the allocation class, the memory area will differ.
> An automatic allocation is done on the stack, a dymanic allocation on the heap and a static is done in the BSS or Data segment of your ELF binary.

Dynamic allocation is really frequently used, as many data are not known at the time the program is run, thus no memory size can yet be figured out, and will only be at runtime. PHP uses lots of dynamic allocation, just by the dynamic nature of the language itself : it doesn't know when you compile it, what size your variables will be, for example.
Dynamic allocation is available through libc's `malloc()` and `free()` functions, themselves being wrappers over the OS Kernel user-land memory services, like `sbrk()` or `mmap()`.
As PHP is a long-living process, often a daemon (FastCGI, PHP-FPM or Apache's mod_php), any memory leak will hurt not only PHP but the whole system.
Because PHP is designed into hundreds of thousands of lines of C code, generating a leak in dynamic memory allocation is really really easy. There must be a solution to prevent leaks or help tracking them, and PHP's got a layer that is dedicated in
dynamic memory management and leak tracking : Zend Memory Manager (ZendMM).

##Dynamic memory allocation problems regarding PHP

###OS differences

Libc's is a wrapper over the Kernel services, and the Kernel is really different according to the OS. Windows and Linux for example, are really different. Unix flavours as well.

![ZendMM](../../../img/php-memory/zendMM.png)

### Heap fragmentation

To understand heap fragmentation, you should write your own memory manager in C. This is an exercize you usually have to deal with in your studies.
`malloc()` manages a heap that it cuts into blocs. When you free a bloc, you create a hole in the heap. As any bloc in the heap is managed into binary trees or linked lists, the more holes you
create, the more CPU cycles will be needed for the next `malloc()` call to succeed (best-fit algorithm). It also happens that a call to `free()` triggers a heap compacting algorithm, which is usually very CPU intensive as well.
Those are well known problems, and any "serious" software have dealed with them by creating a (usually very complex and big) layer over malloc/free duo, and the program asks for dynamic memory
using this specific layer.

> As an example, you may read the [Apache server's dynamic memory library](http://apr.apache.org/docs/apr/0.9/group__apr__pools.html). Big projects such as Firefox or MySQL have even more complex and exciting layers.

Zend Memory Manager (ZendMM) is PHP's dynamic allocation layer. It's been designed to offer good performances for the PHP case, by managing an internal heap over the process heap. You know that PHP is share-nothing-architecture designed, that means that at the end of one web request, PHP will discard any memory allocation done during the request so that it cleans the room for the next request to come.
This is done using the Zend Memory Manager.

For more information about malloc/free internals, you may start your readings by [Once uppon a free()](http://phrack.org/issues/57/9.html) or [the Glibc manual](http://www.gnu.org/software/libc/manual/html_node/Unconstrained-Allocation.html)

###Managing memory leaks

Ah... memory leaks... a whole story every programmer knows about. Fortunately, there exists tools to track them, and they work pretty nicely. valgrind, mtrace, ccmalloc, electric fence...
Let's see how valgrind does the job :

	#include <stdlib.h>
	#include <string.h>
	
	#define MYSTRING "Hello, world ; I'm gonna leak some memory"
	
	int main(int argc, char *argv[])
	{
		char *p_char = (char *)malloc(sizeof(MYSTRING));
		char string[] = MYSTRING;
		memcpy(p_char, string, sizeof(string));
		return 0;
	}

	$ valgrind --tool=memcheck --leak-check=full ./leak
	==9488== Memcheck, a memory error detector
	==9488== Copyright (C) 2002-2010, and GNU GPL'd, by Julian Seward et al.
	==9488== Using Valgrind-3.6.0.SVN-Debian and LibVEX; rerun with -h for copyright info
	==9488== Command: ./leak
	==9488== 
	==9488== 
	==9488== HEAP SUMMARY:
	==9488==     in use at exit: 42 bytes in 1 blocks
	==9488==   total heap usage: 1 allocs, 0 frees, 42 bytes allocated
	==9488== 
	==9488== 42 bytes in 1 blocks are definitely lost in loss record 1 of 1
	==9488==    at 0x4C2815C: malloc (vg_replace_malloc.c:236)
	==9488==    by 0x4005DB: main (leak.c:8)
	==9488== 
	==9488== LEAK SUMMARY:
	==9488==    definitely lost: 42 bytes in 1 blocks
	==9488==    indirectly lost: 0 bytes in 0 blocks
	==9488==      possibly lost: 0 bytes in 0 blocks
	==9488==    still reachable: 0 bytes in 0 blocks
	==9488==         suppressed: 0 bytes in 0 blocks
	==9488== 
	==9488== For counts of detected and suppressed errors, rerun with: -v
	==9488== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 4 from 4)

Valgrind is a very powerful tool. It tracks leaks, but also invalid accesses which may be dangerous about program security (null pointer dereference, write out of alloc'ed bounds, memory leaks, overlaping zones, etc...).

Whatever tool you use, the tool helps you track the problem but won't make it disappear magically : this is your work.
As the program gets bigger, it becomes more and more difficult to track the leaks. A solution is to rely on a layer that embeds some checks about leaks or security. Zend Memory Manager does that for PHP, and trully helps a lot designing extensions or patching PHP itself.

Here is a quick example of ZendMM usage :

	PHP_FUNCTION(make_leak)
	{
		void *leak = emalloc(200); /* emalloc is ZendMM's "malloc" */
		RETURN_NULL(); /* return, forgeting to free the previously allocated buffer */
	}

	$> php /tmp/leak_check.php
	 
	[Thu Apr  7 17:48:07 2011]  Script:  '/tmp/leak_check.php'
	/usr/local/src/php/ext/leak/leak.c(172) :  Freeing 0x01DBB2E0 (200 bytes), script=/tmp/leak_check.php
	=== Total 1 memory leaks detected ===

You can easilly see the stderr output : it tells you're leaking some memory, and it tells you in which place in your code (in the example : leak.c line 172).
What you have to do is to use ZendMM specific alloc functions in place of the default libc's ones; ZendMM will then track any allocation you ask for, and checks that you effectively free them until the end of the current web request.
It will also implement guards (known as *canaries*) to inform you if you write past the allocated blocks, which is a very nice feature to count on as well because forgetting a +1 or -1 in an allocation is really frequent.

> Reminder : ZendMM only complains about leaks and overlaps if PHP's been built in debug mode (--enable-debug), so this is not the case for any "traditionnal" PHP build. Also, ZendMM only takes care of request-bound allocations; most of them are of this kind, but some allocations may need to "persist" through different requests. Those latter are not managed using ZendMM but traditionnal libc calls (mainly).

##Introduction to Zend Memory Manager

###Goals

*	Prevent heap fragmentation by reimplementing a custom heap onto the process' heap. Segmentation, pools and alignment features are in ;
*	Scream at your face about memory leaks or overlaps in the current web request ;
*	Automatically free leaked memory at request shutdown ;
*	Monitor and limit memory usage into all PHP (*memory_limit*) ;
*	Allow to choose the low level allocation stack (depends on OS) ;
*	Allow beeing disabled, so that any memory check tool like valgrind is not hindered by ZendMM.

Zend Memory Manager appeared in PHP 4 and has been fully redesigned in PHP 5.2 and in PHP 7.0

###Configuration

Zend Memory Manager (ZendMM) is enabled by default, and can be disabled (it will still be here, but skirted). It will however change its behavior depending on your compilation options.
A debug PHP build will have a ZendMM telling you about leaks on stderr, if *report_memleaks=1* in php.ini.

![ZendMM-phpinfo](../../../img/php-memory/zendMM-phpinfo.png)

Then come four environment variables to set up ZendMM at runtime : **USE_ZEND_ALLOC**, **ZEND_MM_MEM_TYPE**, **ZEND_MM_SEG_SIZE** and **ZEND_MM_COMPACT**.
If **USE_ZEND_ALLOC** is set to 0, ZendMM is disabled and any call to its functions will be proxied to the OS's low level call, usually malloc/free. `phpinfo()` will then tell you Zend Memory Manager is disabled. We use this to short-circuit ZendMM and run PHP under valgrind, f.e.

**ZEND_MM_MEM_TYPE** defines the low level implementation ZendMM should rely on. Default is "malloc", but you can choose between "mmap_anon" - "mmap_zero" - or "win32".

**ZEND_MM_SEG_SIZE** defines the allocation step. Its like Kernel's PAGESIZE, the minimum allocation unit to use. Default is 256Kb which is a good value. More on that later.

	$ USE_ZEND_ALLOC=0 valgrind --tool=memcheck php /tmp/small.php 
	==6861== Memcheck, a memory error detector
	==6861== Copyright (C) 2002-2010, and GNU GPL'd, by Julian Seward et al.
	==6861== Using Valgrind-3.6.0.SVN-Debian and LibVEX; rerun with -h for copyright info
	==6861== Command: ./php /tmp/small.php
	==6861== 
	 
	==6861== HEAP SUMMARY:
	==6861==     in use at exit: 0 bytes in 0 blocks
	==6861==   total heap usage: 9,697 allocs, 9,697 frees, 2,686,147 bytes allocated
	==6861== 
	==6861== All heap blocks were freed -- no leaks are possible
	==6861== 
	==6861== For counts of detected and suppressed errors, rerun with: -v
	==6861== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 4 from 4)
	 
	$ valgrind --tool=memcheck ./php /tmp/small.php
	==6866== Memcheck, a memory error detector
	==6866== Copyright (C) 2002-2010, and GNU GPL'd, by Julian Seward et al.
	==6866== Using Valgrind-3.6.0.SVN-Debian and LibVEX; rerun with -h for copyright info
	==6866== Command: ./php /tmp/small.php
	==6866== 
	 
	==6866== HEAP SUMMARY:
	==6866==     in use at exit: 0 bytes in 0 blocks
	==6866==   total heap usage: 7,854 allocs, 7,854 frees, 2,547,726 bytes allocated
	==6866== 
	==6866== All heap blocks were freed -- no leaks are possible
	==6866== 
	==6866== For counts of detected and suppressed errors, rerun with: -v
	==6866== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 4 from 4)

> Reminder : PHP memory consumption depends on the run script, and also on PHP extensions which can allocate many memory.
ZendMM only measures and manages request-bound memory allocations ; every "permanent" alloc done using directly malloc() is not accounted here.

PHP memory consumption is weaker when ZendMM is enabled. Also, PHP is faster when ZendMM is enabled.
This is obvious as ZendMM has been designed to fit PHP's memory needs : it preallocates known-size blocs and manages internal lists about them more efficiently than malloc would do. It also prevents many `malloc()` calls, as the valgrind output shows (above).

	$> ZEND_MM_SEG_SIZE=8k php /tmp/my_script.php

Here, ZendMM will ask the underneath layer for 8Kb allocs.

**ZEND_MM_MEM_TYPE** lets you choose the underneath layer to use. Libc's `malloc()` is used by default.

**ZEND_MM_COMPACT** tells ZendMM the size from which it must compact the internal heap. This feature is not enabled under Unix OSes.

##How does ZendMM work ?

ZendMM is an allocator, so its operation is close to any memory allocator. Here are its structures :

*	*zend_mm_heap* : the heap ;
*	*zend_mm_mem_handlers* : the bottom allocators available handlers (malloc, mmap_anon, win32...) ;
*	*zend_mm_segment* : memory segments. Linked list ;
*	*zend_mm_block* & *zend_mm_free_block* : memory blocs (usefull blocs), pluggued into segments.

I wont detail too much ZendMM as it may become very complex if you are not comfortable with memory allocators, and such details are useless here.

###Noticeable structures

*zend_mm_mem_handlers* is the low-level allocator to be used by ZendMM, it is then full of function pointers. malloc-based allocator is defined into the **ZEND_MM_MEM_MALLOC_DSC** macro.

	typedef struct _zend_mm_mem_handlers {
		const char *name;
		zend_mm_storage* (*init)(void *params);
		void (*dtor)(zend_mm_storage *storage);
		void (*compact)(zend_mm_storage *storage);
		zend_mm_segment* (*_alloc)(zend_mm_storage *storage, size_t size);
		zend_mm_segment* (*_realloc)(zend_mm_storage *storage, zend_mm_segment *ptr, size_t size);
		void (*_free)(zend_mm_storage *storage, zend_mm_segment *ptr);
	} zend_mm_mem_handlers;
	
	struct _zend_mm_storage {
		const zend_mm_mem_handlers *handlers;
		void *data;
	};
	
	#define ZEND_MM_MEM_MALLOC_DSC {"malloc", zend_mm_mem_dummy_init, zend_mm_mem_dummy_dtor, zend_mm_mem_dummy_compact, zend_mm_mem_malloc_alloc, zend_mm_mem_malloc_realloc, zend_mm_mem_malloc_free}

*zend_mm_segment* is a memory segment (the base unit ZendMM will use when allocating from the OS). It's size may be changed using the **ZEND_MM_SEG_SIZE** env. The allocator will use this size to allocate a buffer, place a *zend_mm_segment* as head and return the leaving buffer which will be itself cut into blocs linked with each other.

	typedef struct _zend_mm_segment {
		size_t size;
		struct _zend_mm_segment *next_segment;
	} zend_mm_segment;

	typedef struct _zend_mm_free_block {
		zend_mm_block_info info;
	#if ZEND_DEBUG
		unsigned int magic;
	# ifdef ZTS
		THREAD_T thread_id;
	# endif
	#endif
		struct _zend_mm_free_block *prev_free_block;
		struct _zend_mm_free_block *next_free_block;
	 
		struct _zend_mm_free_block **parent;
		struct _zend_mm_free_block *child[2];
	} zend_mm_free_block;

*zend_mm_heap* is the shared heap. It's shared as a global variable into PHP, but you will usually never use it directly (except if you design extensions that plays with PHP memory in any way).

	struct _zend_mm_heap {
		int                 use_zend_alloc;
		void               *(*_malloc)(size_t);
		void                (*_free)(void*);
		void               *(*_realloc)(void*, size_t);
		size_t              free_bitmap;
		size_t              large_free_bitmap;
		size_t              block_size;
		size_t              compact_size;
		zend_mm_segment    *segments_list;
		zend_mm_storage    *storage;
		size_t              real_size;
		size_t              real_peak;
		size_t              limit;
		size_t              size;
		size_t              peak;
		size_t              reserve_size;
		void               *reserve;
		int                 overflow;
		int                 internal;
	#if ZEND_MM_CACHE
		unsigned int        cached;
		zend_mm_free_block *cache[ZEND_MM_NUM_BUCKETS];
	#endif
		zend_mm_free_block *free_buckets[ZEND_MM_NUM_BUCKETS*2];
		zend_mm_free_block *large_free_buckets[ZEND_MM_NUM_BUCKETS];
		zend_mm_free_block *rest_buckets[2];
	#if ZEND_MM_CACHE_STAT
		struct {
			int count;
			int max_count;
			int hit;
			int miss;
		} cache_stat[ZEND_MM_NUM_BUCKETS+1];
	#endif
	};

As you can see, compiling PHP with the debug flag enables many things into those low level structures.

###Published functions

ZendMM API is mainly used when designing PHP internals (extensions or core development), for every request-bound allocation. For this goal, it publishes some functions the developer must use instead of traditionnal libc functions.
Those published functions are very intuitive and easy to use, let's have a look:

	void *emalloc(size_t size);
	void *pemalloc(size_t size, char persistent)
	void *ecalloc(size_t size);
	void *pecmalloc(size_t size, char persistent)
	void *erealloc(void *ptr, size_t size);
	void *perealloc(void *ptr, size_t size, char persistent)
	void *estrdup(void *ptr)
	void *pestrdup(void *ptr, char persistent)	void *strdup(void *ptr)
	void efree(void *ptr)
	void pefree(void *ptr, char persistent)	void free(void *ptr)

As you can see, they share the same API as malloc/free/strdup etc... from libc. A quick word on "p" functions. "p" stands for "persitent", this means that the allocation will persist through requests.
In reality, those "persitent" functions directly proxy to the bottom layer (malloc/free), one should use them for every allocation that is not request bound, meaning that ZendMM won't warn you about possible leaks for them, simply because they can't really leak from request to request and will anyway be cleaned when PHP shuts down.

###From PHP land

PHP allows you, as a PHP developer, to interact with it. Functions `memory_get_usage()` and `memory_get_peak_usage()` are published, as well as the ini setting *memory_limit*.
As most of dynamic allocation request from PHP go through the ZendMM layer, it is very easy for it to count the number of bytes asked so far, and bail out in case of reaching a limit : the *memory_limit*.

	zend_mm_safe_error(heap, "Allowed memory size of %ld bytes exhausted (tried to allocate %ld bytes)", heap->limit, size);

To know PHP dynamic memory usage at a given moment, `memory_get_usage()` may be used. This function returns the size used into the allocated segments. This means that it is less than the real usage of PHP.
To know the real usage, aka the memory to fit the segments in it, pass 1 to the function : `memory_get_usage(1)`;

	ini_set('memory_limit', -1); // unlimited memory
	 
	function show_memory($real = false) {
		printf("%.2f Kb\n", memory_get_usage($real) / 1024);
	}
	 
	show_memory();
	show_memory(1);
	 
	$a = str_repeat('a', 1024*1024*10); // 10Mb
	 
	echo "\n";
	 
	show_memory();
	show_memory(1);

	$> php /tmp/mem.php
	621.62Kb
	768.00 Kb
	 
	10861.83 Kb
	11264.00 Kb

`memory_get_peak_usage()` returns the peak ZendMM recorded in its life.

> Important : Nothing forces the C developers to use ZendMM. Anyone developing an extension (for example) could absolutely not use ZendMM and rely directly on malloc/free, thus allocating dynamic memory that will not be seen by ZendMM and `memory_get_usage()`, *memory_limit* etc... Here, you may use your OS to monitor this. Obviously, C developers know that and heavily rely on ZendMM, but still.

> `memory_get_usage()` give an average information, often accurate, but not 100% accurate on a byte-basis. Use your OS for that.

###Tuning

####ZEND_MM_SEG_SIZE, sizing ZendMM heap allocations

To understand segments and why this is important, imagine a PHP with 256Kb segment size (default). If it has to consume say 320Kb, ZendMM will allocate onto its heap 2 segments, thus using from the OS 512Kb, filled at 320Kb. This is a cursor, and this is where ZendMM gives performances : it allocates more than what is really used, thus it allocates less often, it then prevents the process heap from fragmentation.

	// /tmp/mem.php is the code shown in the above example
	 
	$> ZEND_MM_SEG_SIZE=1048576 php /tmp/mem.php 
	625.67 Kb
	1024.00 Kb
	 
	10865.88 Kb
	12288.00 Kb

It is clear here. PHP consumes 625.67Kb, ZendMM allocated 1Mb segments, so one segment to fit the usage. The real usage is then 1Mb, and the usage is only 625.67Kb.
We then create a 10Mb string, so the memory consumption raises to 10865.88Kb and the real reaches 12288Kb : 12 segments of 1Mb each (1024Kb).

> ZEND_MM_SEG_SIZE must obviously be power-of-two aligned

	
	ini_set('memory_limit', -1); // unlimited memory
	 
	function get_mem_stats() {
		printf("Memory usage %.2f Kb\n", memory_get_usage() / 1024);
		if ($segSize = getenv('ZEND_MM_SEG_SIZE') {
		    printf("Heap segmentation : %d segments of %d bytes (%d Kb used)\n", memory_get_usage(1)/$segSize, $segSize, memory_get_usage(1)/1024);
		}
	}
	 
	get_mem_stats();
	 
	$a = str_repeat('a', 1024*1024*10); // 10 Mb
	 
	echo "\n";
	 
	get_mem_stats();

	$> ZEND_MM_SEG_SIZE=2048 php /tmp/mem.php
	Memory usage 630.97 Ko
	Heap segmentation : 325 segments of 2048 bytes (650 Kb used)
	 
	Memory usage 10871.18 Ko
	Heap segmentation : 5446 segments of 2048 bytes (10892 Kb used)

We can then say that the more tiny the segments are, the more the heap close to real memory usage (economical) but the more often it has to create segments.
This is why, by default, the segment size is 256Kb. With such a value, ZendMM will have to allocate few segments to fit the needs, which are usually around 5Mb. Sure, a framework based app (much more greedy in memory) may benefit from a tunning of the allocator. Look at that :

	
	get_mem_stats();
	
	/* This is the date component from ZF1. This class is known beeing huge
		https://github.com/zendframework/zf1/blob/master/library/Zend/Date.php */
	require 'Zend/Date.php';
	 
	echo "\n";
	get_mem_stats();

	$> ZEND_MM_SEG_SIZE=2048 php /tmp/mem.php 
	Memory usage 630.35 Ko
	Heap segmentation : 325 segments of 2048 bytes (650 Kb used)
	 
	Memory usage 4994.70 Ko
	Heap segmentation : 2687 segments of 2048 bytes (5374 Kb used)

Yeah, we raise from 630Kb to about 5Mb just by making PHP parse the `Zend/Date.php`, which contains a huge class. We even did not make any use of this class, just parsed it.
Remember that in PHP objects are really tiny and very well designed to be thrifty, but all the weight is passed back to the class. In PHP, a class is something consuming memory, a fortiori a big class. You may read more about classes, objects and memory, [in the dedicated article](http://jpauli.github.io/2015/03/24/zoom-on-php-objects.html).

To well tune segment size, you must know the average PHP memory consumption of your app, so that with well sized segments, the allocator won't create and free too many segments too often.
A bad thing for performance is having an application oscillate around a segment, forcing the ZendMM to call the underlying allocator.

![memory_get_usage](../../../img/php-memory/mm.png)
![memory_get_usage](../../../img/php-memory/mm2.png)

####ZEND_MM_MEM_TYPE : choosing the underlying allocator

As we've seen so far, the underlying allocator ZendMM will rely on is configurable. By default, it is set to 'malloc' ('win32' under Windows).

	#define ZEND_MM_MEM_WIN32_DSC {"win32", zend_mm_mem_win32_init, zend_mm_mem_win32_dtor, zend_mm_mem_win32_compact, zend_mm_mem_win32_alloc, zend_mm_mem_win32_realloc, zend_mm_mem_win32_free}
	 
	#define ZEND_MM_MEM_MALLOC_DSC {"malloc", zend_mm_mem_dummy_init, zend_mm_mem_dummy_dtor, zend_mm_mem_dummy_compact, zend_mm_mem_malloc_alloc, zend_mm_mem_malloc_realloc, zend_mm_mem_malloc_free}

You can choose also *mmap_anon* or *mmap_zero*. *mmap_anon* will create a new anonymous memory mapping in your process mapping table :

	zend_mm_segment *ret = (zend_mm_segment*)mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANON, -1, 0);

*mmap_zero* is the same, but using */dev/zero* descriptor (usually BSD based Unixes) :

	zend_mm_dev_zero_fd = open("/dev/zero", O_RDWR, S_IRUSR | S_IWUSR);
	zend_mm_segment *ret = (zend_mm_segment*)mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_PRIVATE, zend_mm_dev_zero_fd, 0);

Using *mmap_anon* or *mmap_zero* is better if you don't want to suffer from your libc's malloc overhead :

	ini_set('memory_limit', -1);
	 
	function heap() {
	return shell_exec(sprintf('grep "VmData:" /proc/%s/status', getmypid()));
	}
	 
	printf ("Original heap status: %s\n", heap());
	 
	$a = range(1, 1024*1024); /* A very big array */
	 
	printf("I'm now eating heap memory: %s\n", heap());
	unset($a); /* Free the memory */
	printf("Memory should now have been freed: %s\n", heap());

This code gets information about the process heap usage using the VmData field provided by the Linux Kernel. Here is the output with malloc :

	> php leak.php
	 
	Original heap status: VmData:         4504 kB
	I'm now eating heap memory: VmData:    152232 kB
	Memory should now have been freed: VmData:    143780 kB

mmm, seems like there is something strange. Seems like memory is heavily leaking because it does not reach back its original value when I destroy the very big array that did ask for mush memory from the heap. What's happening ?

Let's launch this script again, but now choosing *mmap_anon* as underlying allocator for ZendMM :

	>ZEND_MM_MEM_TYPE=mmap_anon php leak.php 
	 
	Original heap status: VmData:         4404 kB
	I'm now eating heap memory: VmData:	  152116 kB
	Memory should now have been freed: VmData:      4916 kB

Aha, seems much better. In this particular case, we've been hit by malloc implementation details. When we freed the memory, ZendMM did call `free()`, but `free()` itself did not free the memory back to the OS, but prefered keeping the blocks in a heat area to serve them back later. This is good if you don't use an overlay, like ZendMM. But using ZendMM, which itself implements a heat zone an reusage of pointers, it is silly to suffer from libc's malloc implementation details (which may vary a lot depending on how `malloc()` has been compiled on your system, you should read your system manual to know about this).

So using *mmap_anon*, if you know what this is, ZendMM will call `munmap()`, which is a Kernel service (system call) which will mark the physical pages as freed, thus unpaging them from your process memory image : your memory consumption will then drop.

> It is better to not use "malloc" as underlying allocator. If you use "malloc", you'll end having two heap managers (ZendMM and malloc) on top of each other, and you'll then suffer from the bottom (malloc) management operations, such as compaction algorithm firing, or blocks not freed when requested.

##A quick word on the Garbage Collector

Let me clarify. Zend Memory Manager has nothing to share with ZendGC. ZendGC, appeared in PHP 5.3, is about clearing circular references in PHP variables and that's absolutely all it does. It then acts far on top of ZendMM, for PHP variables containing themselves (circular references). PHP has always freed back the request-bound memory when it has not used it anymore (request finished), and this is ZendMM role

![ZendGC](../../../img/php-memory/zendgc.png)

##Deeper example

We're gonna trace every dynamic memory allocation from a PHP process, just to have an idea of how PHP uses the heap memory. We're gonna use Valgrind-Massif for that.

Here is the very simple script we'll benchmark :

	
	echo "hello world";

With such a script, there is no chance we use lots of memory from PHP land, as echoing a tiny string is something trivial for memory usage

	> valgrind --tool=massif --massif-out-file=massif.out --max-snapshots=1000 --stacks=yes php /tmp/void.php && ms_print massif.out > massif.txt

		MB
	4.189^                                                                  #     
		 |                                                                 @#@    
		 |                                                               ::@#@:   
		 |                                                              :::@#@@   
		 |                                                             @:::@#@@@  
		 |                                                            :@:::@#@@@  
		 |                                                           @:@:::@#@@@  
		 |                                                          @@:@:::@#@@@: 
		 |                                                        ::@@:@:::@#@@@@ 
		 |                                            @@@@@@:@@@@@@:@@:@:::@#@@@@ 
		 |                                          @:@@@@@@:@@@@@@:@@:@:::@#@@@@ 
		 |                                         @@:@@@@@@:@@@@@@:@@:@:::@#@@@@ 
		 |                                        @@@:@@@@@@:@@@@@@:@@:@:::@#@@@@:
		 |                                        @@@:@@@@@@:@@@@@@:@@:@:::@#@@@@:
		 |                                   @:::@@@@:@@@@@@:@@@@@@:@@:@:::@#@@@@:
		 |                                   @:::@@@@:@@@@@@:@@@@@@:@@:@:::@#@@@@:
		 |                                   @:::@@@@:@@@@@@:@@@@@@:@@:@:::@#@@@@:
		 |                                   @:::@@@@:@@@@@@:@@@@@@:@@:@:::@#@@@@:
		 |                                   @:::@@@@:@@@@@@:@@@@@@:@@:@:::@#@@@@:
		 |                                   @:::@@@@:@@@@@@:@@@@@@:@@:@:::@#@@@@:
	   0 +----------------------------------------------------------------------->Mi
		 0                                                                   27.69

	Number of snapshots: 589
	 Detailed snapshots: [14, 19, 24, 27, 40, 44, 52, 59, 71, 77, 81, 82, 95, 113, 117, 154, 170, 172, 188, 192, 218, 221, 264, 268, 270, 299, 307, 317, 323, 324, 325, 338, 343, 351, 361, 364, 375, 390, 396, 400, 403, 406, 414, 423, 438, 442, 443, 446, 458, 461, 462, 492, 498 (peak), 508, 518, 528, 538, 548, 558, 568, 578, 588]

The max memory usage is (about) 4Mb, and 588 snapshots have been taken. Be warned that this represents the memory usage of my PHP, on my platform etc...
If you have another OS or architecture, the numbers will vary. Also, if you activate more or less PHP extensions, those numbers will vary as well.

	--------------------------------------------------------------------------------
	  n        time(i)         total(B)   useful-heap(B) extra-heap(B)    stacks(B)
	--------------------------------------------------------------------------------
	265     14,029,321              640               37            19          584
	266     14,097,959              848               37            19          792
	267     14,152,209           10,064            3,841           751        5,472
	268     14,194,235        1,336,904        1,328,911         1,913        6,080
	99.40% (1,328,911B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
	->78.43% (1,048,576B) 0x83F90B: zend_interned_strings_init (zend_string.c:48)
	| ->78.43% (1,048,576B) 0x81E0AC: zend_startup (zend.c:744)
	|   ->78.43% (1,048,576B) 0x7BDA18: php_module_startup (main.c:2055)
	|     ->78.43% (1,048,576B) 0x8CFA9B: php_cli_startup (php_cli.c:417)
	|       ->78.43% (1,048,576B) 0x445AA6: main (php_cli.c:1358)
	|         
	->19.61% (262,144B) 0x7F75FB: _zend_mm_alloc_int (zend_alloc.c:1982)
	| ->19.61% (262,144B) 0x7F8750: zend_mm_startup_ex (zend_alloc.c:1126)
	|   ->19.61% (262,144B) 0x7F8888: zend_mm_startup (zend_alloc.c:1221)
	|     ->19.61% (262,144B) 0x7F9306: start_memory_manager (zend_alloc.c:2733)
	|       ->19.61% (262,144B) 0x81DD8A: zend_startup (zend.c:649)
	|         ->19.61% (262,144B) 0x7BDA18: php_module_startup (main.c:2055)
	|           ->19.61% (262,144B) 0x8CFA9B: php_cli_startup (php_cli.c:417)
	|             ->19.61% (262,144B) 0x445AA6: main (php_cli.c:1358)
	|               
	->01.36% (18,191B) in 20 places, all below massif's threshold (01.00%)

At timeslot 268, we can notice that 1.3Mb have been allocated, of which `zend_interned_string_init()` uses 1Mb, and `_zend_mm_alloc_int()` uses 256Kb.
`zend_interned_string_init()` is the interned string buffer used for string interning. By default, it is 1Mb size and can only be changed
at PHP compilation. If you are interested in the way PHP manages strings internally, [there is a dedicated article about that](http://jpauli.github.io/2015/09/18/php-string-management.html).

`_zend_mm_alloc_int()` allocated 256Kb, yes, this is our underlying allocator call, to allocate one segment of memory, the very first one (default is 256Kb), PHP is actually starting and we are very soon in that process at timeslot 268.
Let's keep going :

	--------------------------------------------------------------------------------
	  n        time(i)         total(B)   useful-heap(B) extra-heap(B)    stacks(B)
	--------------------------------------------------------------------------------
	300     15,909,135        1,406,104        1,377,534        12,882       15,688
	301     15,963,752        1,407,088        1,378,252        13,196       15,640
	302     16,018,604        1,407,112        1,378,248        13,176       15,688
	303     16,089,865        1,408,296        1,379,153        13,503       15,640
	304     16,144,629        1,407,288        1,386,986        14,806        5,496
	305     16,183,971        1,743,384        1,720,538        16,886        5,960
	306     16,248,952        1,771,944        1,746,686        19,298        5,960
	307     16,288,999        1,790,200        1,763,176        20,960        6,064
	98.49% (1,763,176B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
	->58.57% (1,048,576B) 0x83F90B: zend_interned_strings_init (zend_string.c:48)
	| ->58.57% (1,048,576B) 0x81E0AC: zend_startup (zend.c:744)
	|   ->58.57% (1,048,576B) 0x7BDA18: php_module_startup (main.c:2055)
	|     ->58.57% (1,048,576B) 0x8CFA9B: php_cli_startup (php_cli.c:417)
	|       ->58.57% (1,048,576B) 0x445AA6: main (php_cli.c:1358)
	|         
	->17.88% (320,000B) 0x83D6F5: gc_init (zend_gc.c:124)
	| ->17.88% (320,000B) 0x81DA5F: OnUpdateGCEnabled (zend.c:81)
	|   ->17.88% (320,000B) 0x833883: zend_register_ini_entries (zend_ini.c:208)
	|     ->17.88% (320,000B) 0x7BDFC7: php_module_startup (main.c:2191)
	|       ->17.88% (320,000B) 0x8CFA9B: php_cli_startup (php_cli.c:417)
	|         ->17.88% (320,000B) 0x445AA6: main (php_cli.c:1358)
	|           
	->14.64% (262,144B) 0x7F75FB: _zend_mm_alloc_int (zend_alloc.c:1982)
	| ->14.64% (262,144B) 0x7F8750: zend_mm_startup_ex (zend_alloc.c:1126)
	|   ->14.64% (262,144B) 0x7F8888: zend_mm_startup (zend_alloc.c:1221)
	|     ->14.64% (262,144B) 0x7F9306: start_memory_manager (zend_alloc.c:2733)
	|       ->14.64% (262,144B) 0x81DD8A: zend_startup (zend.c:649)
	|         ->14.64% (262,144B) 0x7BDA18: php_module_startup (main.c:2055)
	|           ->14.64% (262,144B) 0x8CFA9B: php_cli_startup (php_cli.c:417)
	|             ->14.64% (262,144B) 0x445AA6: main (php_cli.c:1358)
	|               
	->03.43% (61,493B) in 42 places, all below massif's threshold (01.00%)
	| 
	->02.72% (48,640B) 0x82B9B2: _zend_hash_quick_add_or_update (zend_alloc.h:95)
	| ->02.52% (45,136B) 0x824F74: zend_register_functions (zend_API.c:2139)
	| | ->02.52% (45,136B) 0x8257C6: zend_register_module_ex (zend_API.c:1946)
	| |   ->01.79% (31,992B) 0x7BD8B3: php_register_extensions (main.c:1924)
	| |   | ->01.79% (31,992B) 0x7BE020: php_module_startup (main.c:2213)
	| |   |   ->01.79% (31,992B) 0x8CFA9B: php_cli_startup (php_cli.c:417)
	| |   |     ->01.79% (31,992B) 0x445AA6: main (php_cli.c:1358)
	| |   |       
	| |   ->00.73% (13,144B) in 1+ places, all below ms_print's threshold (01.00%)
	| |   
	| ->00.20% (3,504B) in 1+ places, all below ms_print's threshold (01.00%)

Interesting. At timeslot 307, the snapshot starts showing the famous garbage collector impact. The circular garbage collector, to be able to run and do its job, needs not less than 320Kb of memory, which is not trivial.  `php_module_startup()` is the call to start every PHP extensions, which will start registering some classes, some functions etc... 48Kb so far

Etc... We could detail all the snapshot if you wish to have a full night reading this article ;-)

##End

Zend Memory Manager (ZendMM) is a layer sitting on top of every PHP request-related heap allocation. It has been designed to improve PHP performances, as PHP started becoming more and more complex and heap dependent. As you know now, you must compile a debug build of PHP to activate all the interesting parts of ZendMM. ZendMM roles are multiple, but the main ones are to control and free memory blocks between each web request PHP has to treat, effectively implementing a heap manager over the default system one (malloc).

In any case, PHP is build with ZendMM, so you benefit from it, from its very complex lines of codes, without having noticing it yet. Am I wrong ? Memory allocation in a programm can easilly turn to nightmare when you want to take in consideration all the parts : leaks, performance, thread safety, etc...

You can read many articles about allocators (low level ones) on the web, starting by [Benchmarks of the Lockless Memory Allocator](http://locklessinc.com/benchmarks_allocator.shtml) or [Scalable memory allocation using jemalloc](https://www.facebook.com/notes/facebook-engineering/scalable-memory-allocation-using-jemalloc/480222803919).
