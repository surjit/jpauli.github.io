---
layout: post
title:  realpath_cache
---

##Introduction

Do you know those PHP functions, `realpath_cache_get()`, `realpath_cache_size()` ?
php.ini setting *realpath_cache* ?

Realpath cache is a really important concept to know about, especially when it comes to play with symbolic links, a situation
some meet when they deploy code.
This setting is about performance and IO reduction of your server. It has been introducted in PHP 5.1 , when frameworks started
to show in the PHP scene.

### A recall on the stat system call

Ok, so, you know how your system works don't you ? Let me refresh your mind.
When one want to play with a *path*, the Kernel and the filesystem must know exactly what you talk about.
So, whenever you'll use a path to access a file (in the Unix meaning), you or your library or at least your Kernel will
have to resolve it.
Resolving a path is getting information about it : basically is it a file ? is it a directory or is it link ?

The way to do this is by asking the system about the file type, and, in case of a symbolic link, the final file target.
Whenever you use relative paths, such as *"../hey/./you/../foobar"*, you have to resolve them to full paths, and then resolve
those full paths to file entities (Unix sense of "file", so a true file of any type or a directory or a link).

Usually, for relative paths, you're gonna call the [realpath() C function](http://repo.or.cz/w/glibc.git/blob/edea402804bce917cfd7cd1af76212e6364c23db:/stdlib/canonicalize.c#l43). As you can see, [it will lead to](http://repo.or.cz/w/glibc.git/blob/edea402804bce917cfd7cd1af76212e6364c23db:/stdlib/canonicalize.c#l161) a stat() system call.

Calling stat() is heavy, first because this is a system call, needing a Kernel trap and a context switch, and also because it most likely asks the disk about metadata.
The kernel source for stat() is at [http://lxr.free-electrons.com/source/fs/stat.c#L190](http://lxr.free-electrons.com/source/fs/stat.c#L190). Not surprinsingly, it leads to a FileSystem call (inode->getattr()).
Usually, the kernel uses [its buffer caches](http://www.faqs.org/docs/linux_admin/buffer-cache.html), so the impact is really 
tiny, but the buffer cache on a very busy server may not contain your information, thus an IO, which is something you'd prefer
preventing as much as possible.

##What PHP does ?

In PHP projects, we use many files. Nowadays, we use tons of classes, meaning tons of files (assuming one class per file).
So, autoload or not, we'll have to include those files, we'll have to read them, we'll have to ask the Kernel for stat informations about them.
That's why whenever you access a file in PHP, PHP tries to resolve the paths, resolve the links, get file informations; all this using the `stat()` system call, and then caches the result from this call into what is called the **realpath cache**.
Many other softwares use a stat cache, read their source code and you'll notice that ;-)

PHP will cache the result of the call, but only about the realpath. Any other information (owner, access rights, times ...) won't be
cached in this cache, but in the last file access cache.

As usual, we find the solution by having a look at the source code.
Whenever you access a file in PHP, [php_resolve_path()](http://lxr.php.net/xref/PHP_5_5/main/fopen_wrappers.c#473) is used.
This function quickly calls [tsrm_reapath()](http://lxr.php.net/xref/PHP_5_5/TSRM/tsrm_virtual_cwd.c#1925) which itself
calls [virtual_file_ex()](http://lxr.php.net/xref/PHP_5_5/TSRM/tsrm_virtual_cwd.c#1151) and finally, [tsrm_realpath_r()](http://lxr.php.net/xref/PHP_5_5/TSRM/tsrm_virtual_cwd.c#750).

That's where things get interested. Functions like [realpath_cache_find()](http://lxr.php.net/xref/PHP_5_5/TSRM/tsrm_virtual_cwd.c#830) are called, to lookup in a table if the stat informations have already been asked and cached for this
specific path.

A [realpath_cache_bucket](http://lxr.php.net/xref/PHP_5_5/TSRM/tsrm_virtual_cwd.h#211) structure is used, which encapsulates many things :

	typedef struct _realpath_cache_bucket {
	    unsigned long                  key;
	    char                          *path;
	    int                            path_len;
	    char                          *realpath;
	    int                            realpath_len;
	    int                            is_dir;
	    time_t                         expires;
	#ifdef PHP_WIN32
	    unsigned char                  is_rvalid;
	    unsigned char                  is_readable;
	    unsigned char                  is_wvalid;
	    unsigned char                  is_writable;
	#endif
	    struct _realpath_cache_bucket *next;
	} realpath_cache_bucket;


If the bucket is not found, [php_sys_lstat()](http://lxr.php.net/xref/PHP_5_5/TSRM/tsrm_virtual_cwd.h#139) will be called, this function is a proxy to `lstat()`. Then finally, the bucket is [saved into the realpath cache](http://lxr.php.net/xref/PHP_5_5/TSRM/tsrm_virtual_cwd.c#1139).

##PHP Settings and customization

So, in PHP, you have several things to know about realpath cache.
First, the INI settings :

*	[realpath_cache_size](http://www.php.net/manual/en/ini.core.php#ini.realpath-cache-size)
*	[realpath_cache_ttl](http://www.php.net/manual/en/ini.core.php#ini.realpath-cache-ttl)

The manual warns you, if you use files that are not modified often (production servers), you should increase the
TTL.
Also, the default size is ridiculously weak. 16K are gonna be filled in one web request, assuming a framework usage like Symfony2.
Monitor your `realpath_cache_get()` return, you'll see that you hit the default 16K limit very soon. You'd better increase this value to something like 512K or even a megabyte.
If your realpath cache is full, there is no space for other entries, and then PHP will start abusing the `stat()` call because of cache
misses, stressing your Kernel even more.
The size is hard to compute theoretically. As we can see from the source code [in here](http://lxr.php.net/xref/PHP_5_5/TSRM/tsrm_virtual_cwd.c#643), each entry consume sizeof(realpath_cache_bucket) + the total number of characters of the resolved path + 1.
To me (LP64), sizeof(realpath_cache_bucket) = 56 bytes.

There is another trick. PHP resolves **every paths it meets** and splits every path part, resolving it.
I explain : if you access the file "/home/julien/www/fooproject/app/web/entry.php", PHP is gonna split this path into as many single units
as can fit. PHP is gonna resolve "/home", creating an entry for it into the cache. Then "/home/julien", then "/home/julien/www", etc..
Why this ? Well, first this is used to check access at every level of directory. Secondly, because many PHP users tend to build their
pathnames using string concatenations, PHP may have a chance to have checked simple parts, it will then know if the user may access
it or not, by asking the realpath cache for details. A cache hit is very cheap.
The source code of [tsrm_realpath_r()](http://lxr.php.net/xref/PHP_5_5/TSRM/tsrm_virtual_cwd.c#750) details the procedure. this is a recursive function which gets called for every subpath entry, by default.

As you can see from the preceding paragraph, better have a cache !

This also shows that priming the cache by hitting few URLs from your website before opening it to public just after a new deploy is important here as well. This will not only prime your OPcode cache, but also the realpath cache, and your Kernel's page cache as well.

How to clear this cache ? The function is hidden in PHP. `realpath_cache_clear()` ? No, it doesn't exist, too bad :-)
Welcome `clearstatcache(true)`.
The true parameter is very important, it is named *$clear_realpath_cache*, so yes, obviously this is what we want to do.

##An example

So here is an example.

	<?php
	$f = @file_get_contents('/tmp/bar.php');

	echo "hello";

	var_dump(realpath_cache_get());

And here is the result :

	hello
	array(5) {
	  ["/home/julien.pauli/www/realpath_example.php"]=>
	  array(4) {
	    ["key"]=>
	    float(1.7251638834424E+19)
	    ["is_dir"]=>
	    bool(false)
	    ["realpath"]=>
	    string(43) "/home/julien.pauli/www/realpath_example.php"
	    ["expires"]=>
	    int(1404137986)
	  }
	  ["/home"]=>
	  array(4) {
	    ["key"]=>
	    int(4353355791257440477)
	    ["is_dir"]=>
	    bool(true)
	    ["realpath"]=>
	    string(5) "/home"
	    ["expires"]=>
	    int(1404137986)
	  }
	  ["/home/julien.pauli"]=>
	  array(4) {
	    ["key"]=>
	    int(159282770203332178)
	    ["is_dir"]=>
	    bool(true)
	    ["realpath"]=>
	    string(18) "/home/julien.pauli"
	    ["expires"]=>
	    int(1404137986)
	  }
	  ["/tmp"]=>
	  array(4) {
	    ["key"]=>
	    float(1.6709564980243E+19)
	    ["is_dir"]=>
	    bool(true)
	    ["realpath"]=>
	    string(4) "/tmp"
	    ["expires"]=>
	    int(1404137986)
	  }
	  ["/home/julien.pauli/www"]=>
	  array(4) {
	    ["key"]=>
	    int(5178407966190555102)
	    ["is_dir"]=>
	    bool(true)
	    ["realpath"]=>
	    string(22) "/home/julien.pauli/www"
	    ["expires"]=>
	    int(1404137986)

What we can see, is that the full path to my example PHP file has been resolved, parts by parts.
Then, as */tmp/bar.php* doesn't exist on my disk, this entry is obviously missing from the cache. However, we can see that PHP
resolved */tmp*, so it now knows that it can access to /tmp, and any further resolution behind */tmp* will be cheaper than the first one. 

In the array returned by `realpath_cache_get()`, you can see important information, such as the expires timestamp.
This has been computed related to the *realpath_cache_ttl* setting, and the time the file has been accessed.
The key field is a hash of the resolved path, a variant of [FNV hash](http://www.isthe.com/chongo/tech/comp/fnv/index.html) is used, this
is an internal information you shouldn't really need though (which may be integer or float, depending on your integer max size).

Now, if you'd call `clearstatcache(true)`, you'd reset this array and force PHP to `stat()` any new file access that was previously cached.

##The OPcode caches case
Ready for another trick ?

**The realpath cache is process bound, and not shared into shared memory**

This means that anytime a cache entry expires, changes, or you empty the cache manually, you have to do this **for every process in your pool**.
This is usually why people fail at deploying code using OPCode caches solutions.
What people usually do when deploying, is changing a symlink from say /www/deploy-a to /www/deploy-b. What they usually forget is that opcode cache solutions (at least OPCache and APC) rely on the internal realpath cache from PHP.
So those opcode cache solutions won't notice the link change, and worse, they're gonna start noticing it little by little, as the realpath cache of every entry slowly expires. You know the result.

What I find beeing the best solution for deployment to prevent this uncool mechanism to happen, is to prepare a totally new PHP worker pool, and load balance your FastCgi Handler onto it, giving up with the old one when all old workers have finished.

This solution has many advantages : deploy A runs on memory pool A, and deploy B runs on memory pool B. End of story. We use memory image isolation to be absolutely sure that nothing will be shared between two deploys. Realpath cache, OPCode cache, etc... Everything is new.
FastCGI pools load balancing is possible at least with Lighttpd and Nginx :-)
I experienced this solution on production, and it is rock solid !

#End

I've been asked to write some lines about realpath cache, probably because people had bad experience about it (I think at code deployment). Well, now you know how it works, why it's here and how and why to customize it. Did I forget anything ?
