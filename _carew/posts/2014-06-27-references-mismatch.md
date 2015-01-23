---
layout: post
title:  Reference mismatch in PHP function calls
---

##A recall on references

Once again, a coworker just pinged me about a huge memory usage in a Symfony2 based project.
What is bad about Symfony2 ecosystem, is that people tend to use everyone else's code, because it
seems to fit the **usage** need. This is not bad as-is, but wait, what about the performance of the piece of code you're gonna
heavily use ? Nowadays we want fast things, treating much more data than 10 years ago. I think it's time for programmers
who ignore performance to start learning about it.

Tons of PHP programmers are very nice technical guys
at creating functionnal code, and what they call "nice code", you know, with tons of objects and interfaces everywhere...
Fine, right ! But when it comes to write critical parts of code, where performance
trully matter, here suddenly, noone stays on the scene.
This is simply because unfortunately, many people just ignore how PHP works, let me refresh your mind ;-)

So, usually the main problem comes about memory usage. When I hear "my code is eating a gigabyte of
memory" , I just wonder if it has been designed to solve a problem as huge as its memory impact, or what ?
Seriously...

Memory usage, in PHP code, is not really hard to understand. PHP uses a reference counting mechanism
to track variable usages, just like any other language, or even the Kernel itself to manage lists of ... many things.
Reference counting is a really really common basic computer programming trick to save memory.

PHP is very well designed (I'm serious). It tries to do its best to save memory while running your code.
But should you know how reference counting works, you should know there are some situations you should avoid.

I'm gonna talk about reference mimatch in PHP function calls here.

##What to do or not to do ?

What you should do is not use references (&) , until you really master what you do.

More seriously, you should absolutely avoid **reference mismatch** when calling functions.
This is absolutely awfull for PHP, as any mismatch will make it duplicate the variable's memory.
If the variable is big (a very long string, a very complex array), then you're gonna start feeling it.
Worse, you're gonna complain against PHP, which has nothing to do with that. The problem is you, and the
code you are using.

##What is a reference mismatch ?

A reference mimatch is when you call a function whose argument is expected to be passed by reference, and
you pass it a non-reference, or the opposite case.

Here are few examples :

	function foo($arg) { }
	
	$a = "some var";
	$b = &$a; /* turn $a and $b into references */

	foo($a); /* Reference mismatch */
	foo($b); /* idem */

!

	function foo(&$arg) { } /* this function accepts an arg by reference */

	$a = "some var";
	$b = $a; /* increment $a refcount so that the content is bound to two different variables */

	foo($a); /* reference mismatch */
	foo($b); /* idem */

So those both examples are things you should avoid. PHP will duplicate the memory of the argument before
passing it to the function (this is true for every argument).

##What about the objects ?

Objects are a special case. Let me be really clear : PHP never, ever, ever, duplicates an object in memory
until you explicitely tells it to do so. And you only have one way to tell it to do so : the **clone** keyword.

This is really easy to demonstrate :

	function wow($arg) { var_dump('in function wow : ', memory_get_usage()); }

	$big = range(1, 1024*1024); /* This consumes lots of memory */
	$big2 = &$big; /* $big and $big2 are both references to the same memory slot */

	var_dump('original memory', memory_get_usage());
	wow($big); /* $big is a reference, but the function accepts a non-reference : mismatch : duplicate memory */
	var_dump('final memory', memory_get_usage());

Result :

	string(15) "original memory"
	int(151223288)
	string(18) "in function wow : "
	int(251886800)
	string(12) "final memory"
	int(151224488)

Here, there is a classical reference mismatch on a non-object, so PHP will duplicate the passed argument,
which is a big array, so memory usage will raise significantely because PHP will duplicate (shallow copy) a very huge array
(and this burns many CPU cycles as well). Sure, if you don't use the variable elsewhere, when the function call
is finished, PHP destroys the function stack and cleans the dup memory. This, I repeat, if you don't use the argument
elsewhere. This is just a [refcount strategy](http://en.wikipedia.org/wiki/Reference_counting)

What about now using an object as passed arg ?

	function wow($arg) { var_dump('in function wow : ', memory_get_usage()); }

	$big = range(1, 1024*1024); /* This consumes lots of memory */
	$obj = new StdClass; /* Create a basic object */
	$obj->big = $big; /* Turn this object into a BIG object by affecting one of its property to a huge var */
	$obj2 = &$obj; /* Turn $obj into a reference, by linking it by reference to another variable */

	var_dump('original memory', memory_get_usage());
	wow($obj);
	var_dump('final memory', memory_get_usage());

Result:

	string(15) "original memory"
	int(151223928)
	string(18) "in function wow : "
	int(151224040)
	string(15) "final memory"
	int(151223992)

This is a confirmation : PHP doesn't duplicate objects, because objects are internally reference counted themselves.
Here, PHP just adds one more reference to the object, something that can't be done for other types.
So yes : usually, using objects in PHP tend to decrease global memory usage, because if you were using references at some
places, for objects, PHP won't duplicate memory container.

This can easilly be spoted into PHP source code. Have a look at [zval_copy_ctor()](http://lxr.php.net/xref/PHP_5_5/Zend/zend_variables.c#106)
,the function called when PHP duplicate a variable. We can see that in the special case of an object,
PHP just increments a counter, whereas for any other types, it really duplicates memory of the variable, which usually is
not a bad thing as you don't use very big memory variables everytime, but cases happen where you'll carry a big array (with
lots of slots) or a huge string (a result of a file_get_contents() for example).

If you were curious about the duplication of arguments when a function is called, you should have a look at the
[ZEND_SEND_VAR](http://lxr.php.net/xref/PHP_5_5/Zend/zend_vm_def.h#3182) and [ZEND_SEND_REF](http://lxr.php.net/xref/PHP_5_5/Zend/zend_vm_def.h#3145) opcodes.
The [On PHP function calls](http://jpauli.github.io/2015/01/22/on-php-function-calls.html) article may also be worth reading if you get interested in such concepts.

##Other use cases

Any mismatch in function calls is bad. This is true also for internal functions, and some of them accept
parameters by references, like `array_shift()` for example.
When you use such functions, make sure to respect the references as well.

But there are other tricks, which I consider not tricks, but just normal and logical behaviors.
The case of `func_get_args()` is interesting :

	function foo()
	{
		var_dump('Before func_get_args()', memory_get_usage());
		$args = func_get_args();
		var_dump('After func_get_args()', memory_get_usage());
	}
	
	/*
	An example output with some big input variables could be :
	
	string(22) "Before func_get_args()"
	int(151222120)
	string(21) "After func_get_args()"
	int(251885904)
	*/

What you should know is that `func_get_args()` will duplicate all the passed variables to the function, ignoring references or not.
It has to do so, because PHP has no way to know if you're gonna modify the variables later-on.
You all agree that modifying a variable in $args here should not modify the passed arg right ?
Example:

	function foo()
	{
		$args = func_get_args();
		$args[0] = 'foo';
	}
	
	$str = 'bar';
	foo($str);
	
	// here, $str still owns the string 'bar', and not 'foo'

So PHP has to duplicate every variable passed on the function stack, when you call `func_get_args()`.

##End

Well, as I said, usually you don't carry over huge variables in PHP scripts. This is a pretty uncommon use case, however,
as time pass and we ask PHP to build more and more complex systems, managing more and more data; knowing what happens
behind the scene becomes more and more valuable.
Scripting languages show advantages and drawbacks, and one should really master them all before choosing the right language.
For example, I will not make PHP the first choice when talking about designing a language grammar parser.
