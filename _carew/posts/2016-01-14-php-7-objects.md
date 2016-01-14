---
layout: post
title:  PHP 7 objects
---

## Introduction

Here is my first 2016 post, and as well my first PHP 7 targeted post.
I guess 2016 will be full of PHP 7 posts, and empty of PHP 5 posts. Clearly, today, PHP 7 is stable, production ready, released, a lot of extensions have been ported to it, and 2016 is PHP 7 year.

I myself work more under PHP 7 than PHP 5 today, but you know I'm not a PHP developer (any more) but now system developer mastering the internal C-level API of PHP.
So once more, in this post, we'll mainly talk about internal development, but as soon as interesting things show up for userland, we won't hesitate to get back to userland world for a moment to explain interesting concepts.

Let's go ?

## What has changed in objects against PHP 5

You first should read [Zoom on PHP objects and classes](http://jpauli.github.io/2015/03/24/zoom-on-php-objects.html) to have a full focus on PHP objects.

What have changed in PHP 7 about objects ? 

*	 In userland : in a word : nearly nothing. In other words : objects is not the subject that has the most been changed in PHP 7. In conclusion : objects in PHP 7 are the same as PHP 5 objects. Nothing has changed. Nothing deep, nothing you can feel in your day to day development. PHP 7 objects behave the exact same way PHP 5 objects behave in userland. Why ? Because our object model is mature, heavilly used, and there is no need to break it deeply in new PHP versions. Of course it got improved with new additions.
*	 In low level land : there are some changes, they are not **that** big, but they **will** require you to patch your extension code.
Basically, like everywhere else work has been done, PHP 7 new internal objects are much more meaningfull, clean and logical that they were back in PHP 5.
The first big change is related to the master change of PHP 7 : new zval and garbage collection management. We won't last on this point though, but focus on objects. However, new zval and garbage collection mechanisms bring by their own nature some changes into how objects are managed internally.

## Object layout and memory management

First of all, say byebye to *zend_object_value*, this structure does not exist any more in PHP 7.

Now, let's define an object, *zend_object* :

	/* in PHP 5 */
	typedef struct _zend_object {
		zend_class_entry *ce;
		HashTable *properties;
		zval **properties_table;
		HashTable *guards;
	} zend_object;

	/* in PHP 7 */
	struct _zend_object {
		zend_refcounted_h gc;
		uint32_t          handle;
		zend_class_entry *ce;
		const zend_object_handlers *handlers;
		HashTable        *properties;
		zval              properties_table[1]; /* C struct hack */
	};

As we can see, it is a little bit different from PHP 5. it embeds a *zend_refcounted_h* header, which is part of the new zval and garbage collection mechanisms of PHP 7.
Also, what's different from PHP 5 is that now the object embeds its *handle*, whereas on PHP 5, this responsibility was delegated to the *zend_object_store*. We'll see that now in PHP 7, the object store has much less responsibilities.
Also, you may have caught the C struct hack used to inline the zval vector *properties_table*, we'll have to take care of it when we'll create custom object.

## Custom objects management

What is an important change is how we now manage **custom objects**, those are objects of our taste, embedding a *zend_object*.
This is the true power behind Zend Engine object model : the fact that extensions can declare and manage their own objects, that extend Zend default objects implementation without the need of changing the engine source.

Back in PHP 5, we simply create a C struct inheritence embedding the basis definition of a *zend_object*, like this :

	/* PHP 5 */
	typedef struct _my_own_object {
		zend_object        zobj;
		my_custom_type    *my_buffer;
	} my_own_object;

Using C struct inheritence, a simple cast is enough to make it :

	/* PHP 5 */
	my_own_object *my_obj;
	zend_object   *zobj;

	my_obj = (my_own_object *)zend_objects_store_get_object(this_zval);
	zobj   = (zend_object *)my_obj;

You may notice that in PHP 5, when you receive a zval, like $this in OO methods, you can't from this zval access the object it points to directly.
You need to ask the object store. You extract the handle from the zval (in PHP 5), and with this handle you ask the store to give you back the object it found. This object - as it may be a custom one - is returned as a *void\** you must cast it as *zend_object\**, if you didnt customize anything, or as *my_own_object\** if you did.

So in PHP 5, to retrieve an object from a method, you need a lookup. This is not nice in term of performances.

In PHP 7, this is different.
In PHP 7, the object - should it be custom or classic *zend_object* - is nested into the zval directly. **The object store doesn't have any fetch operation available anymore** , that means you can't read from the object store anymore, but only put into it, or delete from it.

**The whole allocated object is embeded into the zval**, thus you don't need an extra lookup when receiving a zval as param and wanting to fetch back the object memory area it points to.

This is how you get an object in PHP 7 :

	/* PHP 7 */
	zend_object *zobj;
	zobj = Z_OBJ_P(this_zval);

Simpler and cleaner than in PHP 5 right ?

Now things change if you have some custom allocation. From the lines above, the only way to fetch a custom object, is to play with memory, aka slice your pointer in any direction of the quantity needed, to make it point where you want. Pure C programming and pure performance : you are very likely to stay in the same physical memory page, and thus prevent your kernel from firing to page-in.

In PHP 7, when you want to declare a custom object, you do it like this :

	/* PHP 7 */
	typedef struct _my_own_object {
		my_custom_type *my_buffer;
		zend_object     zobj;
	} my_own_object;

Notice the **inversion** in the members of the structure (compared to PHP 5). This is because now, when you read the *zend_object* from the zval, to be given back your *my_own_object*, you have to slice the memory backwards by substracting the offset of the *zend_object* into the structure.
This is done using `OffsetOf()` of *stddef.h* (which can be easilly emulated if needed).
This is C advanced structure usage, but if you master the language you use (and at our level you must), this should be something you have already done in your career , so nothing special in here.

So now to fetch your custom object in PHP 7, you'd do it like this :

	/* PHP 7 */
	zend_object   *zobj;
	my_own_object *my_obj;
	
	zobj   = Z_OBJ_P(this_zval);
	my_obj = (my_own_object *)((char *)zobj - XoffsetOf(struct my_own_object, zobj));

Using `offsetof()` here has one implication : **the zend_object must be the last member of your_custom_struct object**. Obviously if you declare types after it, you'll run into trouble accessing them, because of the way *zend_object* is allocated now in PHP 7.

Remember that now in PHP 7, *zend_object* has a struct hack. That means that the allocated memory will differ from `sizeof(zend_object)`.
To allocate a *zend_object*, one would use :

	/* PHP 5 */
	zend_object *zobj;
	zobj = ecalloc(1, sizeof(zend_object));
	
	/* PHP 7 */
	zend_object *zobj;
	zobj = ecalloc(1, sizeof(zend_object) + zend_object_properties_size(ce));

You need to allocate enough space for the members which exact size is given by your class (*ce*), because your class knows everything about declared attributes.

## Object creation

Now let's see a real example.
Let's have a custom object like this one :

	/* PHP 7 */
	typedef struct _my_own_object {
		void        *my_custom_buffer;
		zend_object zobj; /* MUST be the last element */
	} my_own_object;

Here is what its `create_object()` handler could look like :

	/* PHP 7 */
	static zend_object *my_create_object(zend_class_entry *ce)
	{
		my_own_object *my_obj;

		my_obj                   = ecalloc(1, sizeof(my_obj) + zend_object_properties_size(ce));
		my_obj->my_custom_buffer = emalloc(512); /* for example, our custom buffer will fit 512 bytes */

		zend_object_std_init(&my_obj->zobj, ce); /* take care of the zend_object also ! */
		object_properties_init(&my_obj->zobj, ce);

		my_obj->zobj.handlers = &my_class_handlers; /* I have custom handlers, we'll see them later */

		return &my_obj->zobj;
	}

So the difference with PHP 5 here, is that you must not miss the quantity of memory you allocate : you must not forget about the struct hack inlining the *zend_object* properties.
Also, there is no object store involvement anymore here. Back in PHP 5, the create handler had to register the object into the store, and when registering it, pass as well some function pointers for object future destruction and free. This is not needed anymore in PHP 7, resulting in a much more clear `create_object()` function compared to PHP 5.

Simple enough, to use this custom `create_object()` handler, you would declare it into your extension, and this is here as well that you'll declare every handler :

	/* PHP 7 */
	
	zend_class_entry     *my_ce;
	zend_object_handlers my_ce_handlers;
	
	PHP_MINIT_FUNCTION(my_extension)
	{
		zend_class_entry ce;

		INIT_CLASS_ENTRY(ce, "MyCustomClass", NULL);
		my_ce = zend_register_internal_class(&ce);

		my_ce->create_object = my_create_object; /* our create handler */

		memcpy(&my_ce_handlers, zend_get_std_object_handlers(), sizeof(my_ce_handlers));
		
		
		my_ce_handlers.free_obj = my_free_object; /* This is the free handler */
		my_ce_handlers.dtor_obj = my_destroy_object; /* This is the dtor handler */
		/* my_ce_handlers.clone_obj is also available though we won't use it here */
		my_ce_handlers.offset   = XtOffsetOf(my_own_object, zobj); /* Here, we declare the offset to the engine */

		return SUCCESS;
	}

You see here, that we declare the `free_obj()` and the `dtor_obj()` handlers in MINIT. Before, in PHP 5, we had to declare both of them to `zend_objects_store_put()`, when registering the object into the store, **operation which is not needed anymore in PHP 7**. In PHP 7, `zend_object_std_init()` will write the object into the store, you don't do that by hand anymore, so don't miss that call.

So we register our `dtor_obj()` and `free_obj()` handlers here, as well as an offset member, which represents the offset we used to compute our custom object placement in memory.
**This information is needed by the engine**. Why ? Well, simply because in PHP 7, **this is the engine that frees the object, not you any more**.
We'll see that later, but when it comes to free the object, back in PHP 5 you had to do it yourself (often with a simple *free()*), now in PHP 7, the engine does it ; but as the engine will only receive *zend_object* types (obviously), it will have to know the offset in your custom struct, to free the whole pointer. This is [done here](http://lxr.php.net/xref/PHP_7_0/Zend/zend_objects_API.c#189).

## Object destruction

Let's now focus on the destructor.

I remind you that the destructor is called when the object is destroyed from PHP userland, exactly the same way user land's `__destruct()` is called.
So, your destructor may not be called at all in case of fatal errors, and this has not changed in PHP 7.
Also, if you read carefully the articles I point you to or [some talks I gave in the past](http://fr.slideshare.net/jpauli/understanding-php-objects), you should remember that the destructor should not leave the object in an unstable state, because once destroyed, the object can still be accessed in some special cases. This is why we have a destructor handler **and** a free handler, separated. The free handler is called when the engine is extra sure the object is not used elsewhere. The destructor is called when the object's refcount has reached zero, but as now some userland code can be run (`__destruct()`), the current object could be reused as a reference elsewhere, thus it should stay in a stable state. That means that if you free memory in destructor, be extra careful about it. Usually, a destructor should release resources, but not free them (this is the free handler job).

So to sum up things about destructors :
	* The destructor is called exactly zero **or** one time (most likely), but never more than once. In case of fatal errors, it **won't** be called at all.
	* The destructor shouldn't free resources, as the object **could** be reused by the engine in some special rare cases.
	* If you don't call `zend_objects_destroy_object()` from your custom destructor, userland's `__destruct()` won't be triggered.

.

	/* PHP 7 */
	static void my_destroy_object(zend_object *object)
	{
		my_own_object *my_obj;

		my_obj = (my_own_object *)((char *)object - XoffsetOf(my_own_object, zobj));

		/* Now we could do something with my_obj->my_custom_buffer, like sending it
		   on a socket, or flush it to a file, or whatever, but not free it here */

		zend_objects_destroy_object(object); /* call __destruct() from userland */
	}

## Object storage free

Finally, the free storage function is triggered when the engine is extra sure the object is not used elsewhere. The engine **is** about to destroy it, but just before doing so, it calls your `free_obj()` handler. You allocated some resources in your custom `create_object()` handler ? It is now time to free them :

	/* PHP 7 */
	static void my_free_object(zend_object *object)
	{
		my_own_object *my_obj;

		my_obj = (my_own_object *)((char *)object - XoffsetOf(my_own_object, zobj));

		efree(my_obj->my_custom_buffer); /* Free your custom resources */

		zend_object_std_dtor(object); /* call Zend's free handler, which will free object properties */
	}

And that's all. What changes here, is that you don't free the object yourself anymore, like it was the case in PHP 5. In PHP 5, the handler would end with something like *free(object)*. You never do that in PHP 7.
In your `create_object()` handler, you allocated some space for your custom object structure, but as you passed the offset of your custom data to the engine in your MINIT, it is now able itself to free it.
It does that exactly [here](http://lxr.php.net/xref/PHP_7_0/Zend/zend_objects_API.c#189).

Of course, in many cases, the `free_obj()` handler is called immediately after the `dtor_obj()` handler. Only if the userland destructor passes $this to someone else, then it won't be the case (or in the case of a custom extension object which is badly designed or hackish).
Read [zend_object_store_del()](http://lxr.php.net/xref/PHP_7_0/Zend/zend_objects_API.c#147) for the full sequence of code run when the engine wants to release an object.

## Conclusion

We've seen how objects have changed in PHP 7 compared to PHP 5, from an internal point of view. In userland, nothing has changed : all works similar to PHP 5, simply the PHP 7 object model is more optimized, so faster, and has some little more features compared to PHP 5, but nothing big here.

Internally, there are some changes. They are not huge, they won't require you hours of work, but they will generate some work. Both object models are incompatible internally, that means that you must rewrite your extension source code about objects, a little bit.
I tried here to explain the differences keeping the focus on the objects. If you turn to PHP 7 internally, you will notice that it is generally cleaner and more well written than PHP 5, because PHP 5 has suffered from a 10 year old history. PHP 7 is a major and broke many things internally to redesign more correctly some code that kept getting patched over years.

Hope I helped, but remember : all is written in the source code, which is just plain old C ;-)
