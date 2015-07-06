---
layout: post
title:  On PHP function calls
---

## Introducting the facts

This blog post is a technical explanation of a PHP optimization found with [Blackfire profiler](https://blackfire.io/) into a PHP script.
The related post is located here : [http://blog.blackfire.io/owncloud.html](http://blog.blackfire.io/owncloud.html)

Basically, it concludes with :

	if (strlen($name) > 49) {
	...
	}

Beeing about 20% slower than

	if (isset($name[49])) {
	...
	}

Which is perfectly normal.

Hang on. You were going to stop reading here, to go browse your codebase and replace ``strlen()`` calls by ``isset()`` calls. So that's why I stopped you.
If you read carefully [the original blog post](http://blog.blackfire.io/owncloud.html), this performance result boost of about 20% is obtained because ``strlen()`` was used in a loop
of about 60 to 80K iterations (60,000 to 80,000).

## Why such a result ?

It's not the way ``strlen()`` computes the length that's in cause. Because the length of every PHP string is always known when ``strlen()`` is called. A big part of such lengths are even computed at compile time, when possible.
PHP encapsulates the length of the string into the C structure carrying a PHP string, when it creates such string into memory. So ``strlen()`` just reads that little info, and returns it to you as-is.
In fact, ``strlen()`` is probably the fastest PHP function that exists. It just does nothing in term of computation. Here is its source code:

	ZEND_FUNCTION(strlen)
	{
		char *s1;
		int s1_len;

		if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &s1, &s1_len) == FAILURE) {
			return;
		}

		RETVAL_LONG(s1_len);
	}

Knowing that ``isset()`` is not a function, the ~20% perf penalty of ``strlen()`` over ``isset()`` is mainly brought by the overhead of a function call in the Zend Engine.

There is another thing to say as well : comparing the result of ``strlen()`` with something, adds an extra OPCode, whereas using just an ``isset()``, represents one unique OPCode.

Here is the _if(strlen())_ construct disassembled :

	line     #* I O op                           fetch          ext  return  operands
	-----------------------------------------------------------------------------------
	   3     0  >   SEND_VAR                                                 !0
		     1      DO_FCALL                                      1  $0      'strlen'
		     2      IS_SMALLER                                       ~1      42, $0
		     3    > JMPZ                                                     ~1, ->5
	   5     4  > > JMP                                                      ->5
	   6     5  > > RETURN                                                   1

And here is the semanticaly equivalent _if(isset())_ structure disassembled :

	line     #* I O op                           fetch          ext  return  operands
	-----------------------------------------------------------------------------------
	   3     0  >   ISSET_ISEMPTY_DIM_OBJ                       33554432  ~0      !0, 42
		     1    > JMPZ                                                  ~0, ->3
	   5     2  > > JMP                                                       ->3
	   6     3  > > RETURN                                                     1

As you can see, there is no function call involved into the ``isset()`` code (**DO_FCALL**), as well as there is no **IS_SMALLER** OPCode (just ignore the RETURN statements).
``isset()`` will directly return a boolean for evaluation, whereas ``strlen()`` will return a temporary variable, passed to **IS_SMALLER** OPCode, and only this OPCode result will be evaluated by the ``if()``.
That's two OPCodes for the ``strlen()`` code structure and only one for the ``isset()`` one, which lets us smell that the ``isset()`` structure will also be faster because of this fact. (computing two operations is usually slower that computing just one).

Let's now analyze how function calls work in PHP, and how they are different from ``isset()``.

## PHP function calls in deep

Let me warn you : function calls are complex in PHP. If you want to continue reading this part, you'd better fasten your seat belt ;-)
In PHP's design and source code, the most complex execution part to analyze is all that's related to function calls.
I will try to sumarize things here, so that you get enough info to understand, without all the full details related to function calls. You still can fetch them by analyzing the source code.

Here, we will talk about *runtime* of a function call. You should know that the *compile time* PHP function related operations are also heavy to run for the machine (I mean, really heavy), but as you use an OPCode cache, you don't suffer from anything related to compile time (that is transformation of the PHP source code into Zend VM instructions).
So here, we assume the script is compiled, and we'll analyze what happens at _runtime_ only.

Let's just dump the OPCode of an internal function call (``strlen()``, here) :

	strlen($a);

	line     #* I O op                           fetch          ext  return  operands
	-----------------------------------------------------------------------------------
	3     0  >   SEND_VAR                                                 !0
	      1      DO_FCALL                                      1          'strlen'

To understand function calls, one should know those points :

*	Function calls and method calls are exactly the same
*	User functions calls and internal functions calls are differently handled
*	Function calls requires the creation/destruction of a VM stack frame (a memory space where to store variables passed to the function).

That's why I talked about an "internal" function call in my last statement, because I show an example of a call to an internal PHP function, that is a PHP function that's designed in C, here : PHP's ``strlen()``. If we were dumping the OPCode of a "user" PHP function - that is a function that a programmer wrote using the PHP language - OPCode could have been different, but they could have been exactly the same as well.

_This is because PHP doesn't generate the same OPCode weither at compile time it knows the function, or not._
Obviously, internal PHP functions are known at compile time (because they are discovered before the compiler even starts), but that is not necessary true for user functions, which can be called without having been declared before, but after.
Also, when talking about the execution, internal PHP functions are more efficient than user PHP functions, as well as internal benefit from more validation mechanisms than user functions.

### The Zend VM argument stack frame

Let's continue with the internal functions examples, and the ``strlen()`` one just above.
On the OPCode shown above, we can see that a function call is not managed using just one OPCode. In fact, there is a first thing you should remember when talking about functions : they own a stack.
Like in C, or in every language, when you want to call a function, you first have to build what's called a stack frame, and push onto this stack frame the arguments for the function.
Then, you call the function, and this one will most likely pop those arguments from the stack, to use them.
Once the function call is done, you have to destroy the stack frame you allocated to it.

This is the main rule of doing things, but PHP optimizes the stack frame creation and deletion and delays those operations.

When the PHP compiler compiles a script, it knows how many function calls will happen, and how many args those will need. For example, in such a code :

	function foo() { }
	
	foo('bar', false);

The compiler knows you are going to call foo() with two arguments (whatever number was declared in the function body). So the compiler tells the engine that the stack frame for this file will be of exactly two arguments. This number is always known by the compiler, even if you use variable argument passing.

The compiler does it by filling this number into the OPArray, (which is the main structure used by the executor), and when the VM will create the stack frame for this OPArray, it will know how many slots to allocate for the variables.

So PHP's stack frame creation for function argument passing is well optimized, there is nothing to compute at execution time, but at compile time.
Even better, the stack frame is allocated in some "permanent" memory pools. That means that when the stack frame is destroyed : when the function call ends, the engine doesn't free the argument list memory, it keeps it warm for further reuse. That prevents some memory allocation and free, back and forth, which is something really bad for a process overall performances.
A structure exists for the VM stack frame, as well as an API

	struct _zend_vm_stack {
		void **top;
		void **end;
		zend_vm_stack prev;
	};

	static zend_always_inline void *zend_vm_stack_alloc(size_t size TSRMLS_DC)
	{
		void *ret;

		size = (size + (sizeof(void*) - 1)) / sizeof(void*);

		if (ZEND_MM_ALIGNMENT > sizeof(void*)) {
			/* ... ... */
		} else {
			ZEND_VM_STACK_GROW_IF_NEEDED((int)size);
		}
		ret = (void*)EG(argument_stack)->top;
		EG(argument_stack)->top += size;
		return ret;
	}

	#define ZEND_VM_STACK_GROW_IF_NEEDED(count)							\
		do {															\
			if (UNEXPECTED((count) >									\
				EG(argument_stack)->end - EG(argument_stack)->top)) {	\
				zend_vm_stack_extend((count) TSRMLS_CC);				\
			}															\
	} while (0)

So finally, stack frame management is not that heavy in PHP, what however can be, is argument passing.


**SEND_VAR** is an opcode that is responsible of pushing args onto the stack frame. The compiler inevitably generates such an OPCode before a function call. And there will be as many of them as there are variables to pass to the function. See :

	$a = '/';
	setcookie('foo', 'bar', 128, $a);

	line     #* I O op                           fetch          ext  return  operands
	-----------------------------------------------------------------------------------
	   3     0  >   ASSIGN                                                   !0, '%2F'
	   4     1      SEND_VAL                                                 'foo'
		     2      SEND_VAL                                                 'bar'
		     3      SEND_VAL                                                 128
		     4      SEND_VAR                                                 !0
		     5      DO_FCALL                                      4          'setcookie'

This shows another OPCode : **SEND_VAL**. In fact, there exists 4 opcodes to send something on the function stack :

*	**SEND_VAL** : sends a compile-time constant value (a string, an int, etc...)
*	**SEND_VAR** : sends a PHP variable ($a)
*	**SEND_REF** : sends a PHP variable beeing a reference, to a function accepting its arg by reference
*	**SEND_VAR_NO_REF**: Optimized handler used in case of nested function calls

We'll just foresee **SEND_VAR**. What does **SEND_VAR** do ?

	ZEND_VM_HELPER(zend_send_by_var_helper, VAR|CV, ANY)
	{
		USE_OPLINE
		zval *varptr;
		zend_free_op free_op1;
		varptr = GET_OP1_ZVAL_PTR(BP_VAR_R);

		if (varptr == &EG(uninitialized_zval)) {
		    ALLOC_ZVAL(varptr);
		    INIT_ZVAL(*varptr);
		    Z_SET_REFCOUNT_P(varptr, 0);
		} else if (PZVAL_IS_REF(varptr)) {
		    zval *original_var = varptr;

		    ALLOC_ZVAL(varptr);
		    ZVAL_COPY_VALUE(varptr, original_var);
		    Z_UNSET_ISREF_P(varptr);
		    Z_SET_REFCOUNT_P(varptr, 0);
		    zval_copy_ctor(varptr);
		}
		Z_ADDREF_P(varptr);
		zend_vm_stack_push(varptr TSRMLS_CC);
		FREE_OP1();  /* for string offsets */

		CHECK_EXCEPTION();
		ZEND_VM_NEXT_OPCODE();
	}


It checks if your variable is a reference or not. If it is, it separates it, creating a reference mismatch, and that's bad. [I explain
that in this article](http://jpauli.github.io/2014/06/27/references-mismatch.html). That increases the price to pay for function calls in PHP, argument copies because of reference mismatch are really bad for performances.
Then it adds a refcount to the variable, and pushes it onto the VM stack :

	Z_ADDREF_P(varptr);
	zend_vm_stack_push(varptr TSRMLS_CC);

Yes, everytime you call a function, you increment the refcount of every stack argument variable by one : because the function stack itself will reference the variable, not yet the function code, just the stack at the moment. Also, as you can see and would have guessed, we push pointers onto the stack, not the argument itself, that's why we add a reference to it. Remember the stack frame slots have already been reserved and just wait to be filled in by pointers to variables, by the different SEND OPCodes. If no argument mismatch happening, pushing an arg onto the stack frame is pretty light, you end-up just adding a reference to it, and copying (in C) a pointer variable.

### Performing the function call

After pushing the args onto the stack, we run a **DO_FCALL** OPCode, and here, you'll see how many tons of code and checks are performed, allowing us to assume PHP function calls are a "slow" statement :

	ZEND_VM_HANDLER(60, ZEND_DO_FCALL, CONST, ANY)
	{
		USE_OPLINE
		zend_free_op free_op1;
		zval *fname = GET_OP1_ZVAL_PTR(BP_VAR_R);
		call_slot *call = EX(call_slots) + opline->op2.num;

		if (CACHED_PTR(opline->op1.literal->cache_slot)) {
		    EX(function_state).function = CACHED_PTR(opline->op1.literal->cache_slot);
		} else if (UNEXPECTED(zend_hash_quick_find(EG(function_table), Z_STRVAL_P(fname), Z_STRLEN_P(fname)+1, Z_HASH_P(fname), (void **) &EX(function_state).function)==FAILURE)) {
		    SAVE_OPLINE();
		    zend_error_noreturn(E_ERROR, "Call to undefined function %s()", fname->value.str.val);
		} else {
		    CACHE_PTR(opline->op1.literal->cache_slot, EX(function_state).function);
		}
		call->fbc = EX(function_state).function;
		call->object = NULL;
		call->called_scope = NULL;
		call->is_ctor_call = 0;
		EX(call) = call;

		FREE_OP1();

		ZEND_VM_DISPATCH_TO_HELPER(zend_do_fcall_common_helper);
	}

As you can see, here, we perform some little checks. And many caches are used. For example, the function handler pointer is looked up at the
very first call, then it is cached into the main VM frame, so that any further call will use the cached pointer.
Many cache tricks are used into the Zend VM, for it to be as efficient as possible.

After that, we call __zend_do_fcall_common_helper()__.
I can't copy the code of this function here, as it is so huge.
I will show you what operations are done into it. Basically, many checks are done, and they all are done now : at runtime, because once more of the heavilly dynamic nature of PHP.
PHP can define new functions at runtime, and PHP can autoload files, defining classes and functions at runtime, so PHP must perform a lot of its checks at runtime, and this is bad for performance, but that's the design of a dynamic language, those are othogonal concepts.

	if (UNEXPECTED((fbc->common.fn_flags & (ZEND_ACC_ABSTRACT|ZEND_ACC_DEPRECATED)) != 0)) {
			if (UNEXPECTED((fbc->common.fn_flags & ZEND_ACC_ABSTRACT) != 0)) {
				zend_error_noreturn(E_ERROR, "Cannot call abstract method %s::%s()", fbc->common.scope->name, fbc->common.function_name);
				CHECK_EXCEPTION();
				ZEND_VM_NEXT_OPCODE(); /* Never reached */
			}
			if (UNEXPECTED((fbc->common.fn_flags & ZEND_ACC_DEPRECATED) != 0)) {
				zend_error(E_DEPRECATED, "Function %s%s%s() is deprecated",
					fbc->common.scope ? fbc->common.scope->name : "",
					fbc->common.scope ? "::" : "",
					fbc->common.function_name);
			}
		}
		if (fbc->common.scope &&
			!(fbc->common.fn_flags & ZEND_ACC_STATIC) &&
			!EX(object)) {

			if (fbc->common.fn_flags & ZEND_ACC_ALLOW_STATIC) {
				/* FIXME: output identifiers properly */
				zend_error(E_STRICT, "Non-static method %s::%s() should not be called statically", fbc->common.scope->name, fbc->common.function_name);
			} else {
				/* FIXME: output identifiers properly */
				/* An internal function assumes $this is present and won't check that. So PHP would crash by allowing the call. */
				zend_error_noreturn(E_ERROR, "Non-static method %s::%s() cannot be called statically", fbc->common.scope->name, fbc->common.function_name);
			}
		}

See all those checks ? Let's continue, because it's far from beeing finished :

	if (fbc->type == ZEND_USER_FUNCTION || fbc->common.scope) {
		should_change_scope = 1;
		EX(current_this) = EG(This);
		EX(current_scope) = EG(scope);
		EX(current_called_scope) = EG(called_scope);
		EG(This) = EX(object);
		EG(scope) = (fbc->type == ZEND_USER_FUNCTION || !EX(object)) ? fbc->common.scope : NULL;
		EG(called_scope) = EX(call)->called_scope;
	}

You know that each function body has its own variable scope. Well it's not magical : the engine switches the scope tables before calling the function code, so that if this one asks for a variable, it will be looked for into the right table.
And as functions and methods are all the same, you can read some instructions about binding the ``$this`` pointer for a method.
Let's keep going.

	if (fbc->type == ZEND_INTERNAL_FUNCTION) {

I told you, internal functions (those designed in C) take a different execution path from user functions. Usually, internal function execution path is more optimized and shorter than user function one, because for internal functions, in C, we can tell the engine internal info about our function, something user designed functions cant.

	fbc->internal_function.handler(opline->extended_value, ret->var.ptr, (fbc->common.fn_flags & ZEND_ACC_RETURN_REFERENCE) ? &ret->var.ptr : NULL, EX(object), RETURN_VALUE_USED(opline) TSRMLS_CC);

This line above is the line that calls the internal function handler. For our ``strlen()`` example, this above line calls to run the ``strlen()`` source code :

	/* PHP's strlen() source code */
	ZEND_FUNCTION(strlen)
	{
		char *s1;
		int s1_len;

		if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &s1, &s1_len) == FAILURE) {
			return;
		}

		RETVAL_LONG(s1_len);
	}

And what does ``strlen()`` do ? It pops the argument stack using ``zend_parse_parameters()``.
And this zend_parse_parameters() function is "slow", because it has to pop the stack and transform the argument to a type which is expected by the function : for ``strlen()`` : a string.
So whatever the programmer passed onto the stack to ``strlen()``, this one could need to convert the argument to a string, which is not a light process in term of performance.
[Read the source of zend_parse_parameters()](http://lxr.php.net/xref/PHP_5_5/Zend/zend_API.c#729) to have an idea on how many operations we ask our CPU to do, when poping the arguments from a function stack frame.

Let's continue into the execution, we just executed the function body code, now, let's cleanup things, starting by restoring the scope :

	if (should_change_scope) {
			if (EG(This)) {
				if (UNEXPECTED(EG(exception) != NULL) && EX(call)->is_ctor_call) {
					if (EX(call)->is_ctor_result_used) {
						Z_DELREF_P(EG(This));
					}
					if (Z_REFCOUNT_P(EG(This)) == 1) {
						zend_object_store_ctor_failed(EG(This) TSRMLS_CC);
					}
				}
				zval_ptr_dtor(&EG(This));
			}
			EG(This) = EX(current_this);
			EG(scope) = EX(current_scope);
			EG(called_scope) = EX(current_called_scope);
		}

And clearing the stack :

	zend_vm_stack_clear_multiple(1 TSRMLS_CC);

Finally, if an exception has been thrown during this function execution, we must change the VM path to run the catch block (simplified) :

	if (UNEXPECTED(EG(exception) != NULL)) {
			zend_throw_exception_internal(NULL TSRMLS_CC);
			if (RETURN_VALUE_USED(opline) && EX_T(opline->result.var).var.ptr) {
				zval_ptr_dtor(&EX_T(opline->result.var).var.ptr);
			}
			HANDLE_EXCEPTION();
		}

### A conclusion on PHP function calls ?

Now, can you imagine the time your machine spends for calling a very-tiny-and-simple ``strlen()`` function ?
Now multiply this time, because ``strlen()``is called into a loop, with 25,000 iterations, slowly, micro/milli seconds turn to seconds...
Keep in mind that I just showed you the hot path of instructions beeing run at every PHP function call. Many other things happen as well next to those mandatory steps.
Also keep in mind that here, because ``strlen()`` "useful work" is just one line, the overhead of the engine preparing the function call is larger than the useful function code itself, but that is generaly not the case of a big average of function calls, where the own code of the function itself will consume hopefuly more performance than the noisy surrounding engine code.

You may complain about this. You have this right, but please, don't come to me complaining without proposing a valid technical solution to improve this, knowing that if you break compatibility, you'll have to show strong arguments about this ;-)

The PHP-function-call-related part of the code has been reworked into PHP7 (among many things), and in PHP7, function calls are faster than this. As you can imagine by reading this : there is lot of room for optimization yet into PHP source code, and we, as contributors, try to find ways to optimize things at every new PHP version.

Not only talking about PHP7, the PHP-function-call-related code has been optimized in every version of PHP, from 5.3 to 5.4, and especially from 5.4 to 5.5, where we changed the way the stack frame is computed and created, for example (without breaking compatibility).
It is always interesting to read the readme about internals change of each PHP version. [Here is the one for PHP5.5](https://github.com/php/php-src/blob/PHP-5.5/UPGRADING.INTERNALS#L24) talking about changes into the executor and into how function calls are performed, compared to PHP54.

As a conclusion, remember that this is not a blame to PHP. The PHP source code has been worked for twenty years now, by many different very talented brains, so believe me : it has been thought, worked and optimized many times as of nowadays.
The proof is that you use PHP, you have some PHP code in production, and it just works and perform very well, with a nice overall performance factor in the very large majority of use cases. Am I wrong ?

### What about isset() ?

``isset()`` is not a function. Parenthesis don't automatically mean "function call". ``isset()`` is compiled into a special Zend VM OPcode (**ISSET_ISEMPTY**), which will not trigger a function call and suffer from function call overhead we have detailed in the last chapter.

As ``isset()`` can take several parameter types, its Zend VM code is a little bit long, but just isolating the parameter-is-a-string-offset part, it leads to this :

	ZEND_VM_HELPER_EX(zend_isset_isempty_dim_prop_obj_handler, VAR|UNUSED|CV, CONST|TMP|VAR|CV, int prop_dim)
	{
		USE_OPLINE zend_free_op free_op1, free_op2; zval *container; zval **value = NULL; int result = 0; ulong hval; zval *offset;

		SAVE_OPLINE();
		container = GET_OP1_OBJ_ZVAL_PTR(BP_VAR_IS);
		offset = GET_OP2_ZVAL_PTR(BP_VAR_R);

		/* ... code pruned ... */
		} else if (Z_TYPE_P(container) == IS_STRING && !prop_dim) { /* string offsets */
		    zval tmp;
		    /* ... code pruned ... */
		    if (Z_TYPE_P(offset) == IS_LONG) { /* we passed an integer as offset */
		        if (opline->extended_value & ZEND_ISSET) {
		            if (offset->value.lval >= 0 && offset->value.lval < Z_STRLEN_P(container)) {
		                result = 1;
		            }
		        } else /* if (opline->extended_value & ZEND_ISEMPTY) */ {
		            if (offset->value.lval >= 0 && offset->value.lval < Z_STRLEN_P(container) && Z_STRVAL_P(container)[offset->value.lval] != '0') {
		                result = 1;
		            }
		        }
		    }
		    FREE_OP2();
		} else {
		    FREE_OP2();
		}

		Z_TYPE(EX_T(opline->result.var).tmp_var) = IS_BOOL;
		if (opline->extended_value & ZEND_ISSET) {
		    Z_LVAL(EX_T(opline->result.var).tmp_var) = result;
		} else {
		    Z_LVAL(EX_T(opline->result.var).tmp_var) = !result;
		}

		FREE_OP1_VAR_PTR();

		CHECK_EXCEPTION();
		ZEND_VM_NEXT_OPCODE();
	}

Appart from many decision points (many ifs structures), the real hot computing algorithm here can be summed up in one line :

	if (offset->value.lval >= 0 && offset->value.lval < Z_STRLEN_P(container))

If the offset is positive (you did not mean ``isset($a[-42])``), and it is stricly less than the length of the string,
result will be passed 1, and then the resulting operation will be the boolean TRUE.
Don't worry about the length computation : ``Z_STRLEN_P(container)`` will not compute anything. Remember that PHP already knows the length
of your string, ``Z_STRLEN_P(container)`` just read that value in memory : very few CPU cycles are needed for that.

Now I think you understand that there are much much more CPU instructions involved in the PHP function call of ``strlen()`` than in the
use of ``isset()`` in the case of a string offset. You see how ``isset()`` is light ? Don't be scared by many if statements, they are not the heaviest parts of the C code and can be optimized in some ways by the C compiler.
The ``isset()`` handler code doesn't lookup hashtables, doesn't perform any complex check, doesn't push any pointer to any stack frame, to pop them back later, eventually transforming the data...
The code is way lighter than the overall code of a function call, with many less memory accesses (this is the most important part to notice). And in this particular case of a string : it leads to a huge improvement if you were iterating this code over and over again into a loop.
Of course, if you just run one iteration of comparison between ``strlen()`` and ``isset()``, you will find the performance benefit really low, about 5ms difference, something like that. But multiplied by 50,000 iterations...

Notice as well, [by reading the whole isset() source code](http://lxr.php.net/xref/PHP_5_5/Zend/zend_vm_def.h#4445), that it is shared with the ``empty()`` code.
``empty()`` on a string offset will differ from the same ``isset()`` statement by just additionnaly reading if the first character of the
string is not the '0' character.
``empty()`` and ``isset()`` lead to the exact same code beeing run, with just one little tiny diff somewhere, so ``empty()`` will have the exact same performance impact that ``isset()`` (assuming you use both with the same parameter)

## What can OPCache do for us ?

Short answer : Nothing.

OPCache optimizes your code. I talked about this many times in worldwide conferences, you may fetch my slides at [http://fr.slideshare.net/jpauli/yoopee-cache-op-cache-internals](http://fr.slideshare.net/jpauli/yoopee-cache-op-cache-internals) for example.

We've been asked on github if we could add an optimization pass that would switch ``strlen()`` to ``isset()`` : That's not possible.

Remember that OPCache optimization passes act on the OPArray before storing it into shared memory. _This happens at compile time and not
at runtime_. But at compile time, how can we know that the variable you pass to ``strlen()`` is a string ? We can't. That's the PHP problem,
and that's a part of how HHVM/Hack solved it. If we could type our variables in PHP, that is, supporting strong typing, then we could optimize
much more things in compiler passes (as well as into the VM).
Because of the dynamic nature of PHP, nearly nothing is known at compile time. The only thing OPCache can optimize are static, compile-time-known things.
For example, this can be optimized by OPCache :

	if (strlen("foo") > 8) {
	 /* do domething */
	} else {
	 /* do something else */
	}

At compile time, we know here that "foo"'s string length is not above 8, and we can trash all the if() opcodes as well as the "true" part of
the if, and just run the "else" part.
But here :

	if (strlen($a) > 8) {
	 /* do domething */
	} else {
	 /* do something else */
	}

What is in $a ? Does even $a exist here ? Is it a string ?
At the time the optimizer shows in, it just can't answer those questions, that is the VM executor role. At compile time, we end up handling abstract structure with no real type yet, type and related memory usage will be known/allocated at runtime only.

OPCache optimizer already optimizes many things, but by the heavilly dynamic nature of the PHP language, we can't optimize everything, at least not as much as in Java's compilers, or even C's compilers.

That's why when I hear proposal to add strong typing to the language, I like it, because it will boost the performance. PHP 7 added some more typing, and took benefit from it for optimizing things because now, types may be known at compile time.

Also, proposal such as adding a read-only hint to class property declaration :

	class Foo {
		public read-only $a = "foo";
	}

Not talking about the functionnality itself, my mind is tied to performance : such proposals are really nice in term of performance optimization, because here, when we compile such class, we know the value of $a, and we know it can't
change, so we can store its value somewhere, use a cached pointer, and strongly optimize every access to such a variable into weither
PHP compiler, or OPCache optimization passes.

This case really looks like managing a constant, but I think you understood the base line here : the more informations you can give to the compiler about the type and the usage of your variables or functions, the more it will be able to optimize the OPCode for the VM and the structures used, to get closer to what the CPU will need. A good tradeoff between those two concepts is called "JIT" compilation, a subject I won't talk about in this article.

## Optimization tips and conclusion

The first tip I would like to share with you, is to not blindly change your code, everywhere you have a good feeling of it.
Profile. Profile, and see the results.
With profilers such as [Blackfire](https://blackfire.io/), you can immeditely see the hot path of your script, because it automaticaly trashes the irrelevant
information that usually catch your eyes when you read a profile.
You then know _where_ to start working, because your work costs money, and for it to be worth it, it has to be optimized as well.
That's a nice balance between the money you'll cost optimizing a script, and the money you'll save because your cloud will be smaller
and will cost you less.

The second tip is that PHP is fast, believe me.
For what you ask it to do, the way it does the job and the tools it represents to you : it is fast, efficient, reliable.
There is not that much room to optimize PHP scripts, at least not as if you were using lower level language like C.
The main trick is to optimize what is repeated : loops. If you use a profiler showing you the hot path of you script,
you'll happen to find that it is likely to be located into loops.
That's the same when we, as contributors, optimize PHP itself : we won't bother optimizing a part of code a few users will trigger, but better
optimize the hot path : variable accesses, engine function calls, etc... Because in here, the very little micro-second earned
will translate to final milli-seconds or even seconds, as such code is run tons of times (usually involving in loops).
Except ``foreach()``, in PHP, loops are the same and lead to the same OPCode. Turning a PHP's ``while`` loop into a ``for`` loop is
both useless and silly. Once more : profiling will tell you that.

So, about function calls, well... A language needs function calls ;-)
However, there are little tricks that can be used to prevent some function calls, because the information is available elsewhere.
Like :

	php_version() => use the PHP_VERSION constant
	php_uname() => use the PHP_OS constant
	php_sapi_name() => use the PHP_SAPI constant
	time() => read $_SERVER['REQUEST_TIME']
	session_id() => use the SID constant

The above examples are sometimes not 100% equivalent, and I let you read the documentation to find the differences.

Ah ho yes, let me tell it because we still see such things nowadays (uncommonly) : prevent silly things, like :

	function foo() {
		bar();
	}

Or even worse :

	function foo() {
		call_user_func_array('bar', func_get_args());
	}

Basically, don't stop working, don't stop using PHP, don't optimize something "because you heard that", "because someone told you that", and don't design your application by performance, but keep doing it by features.

However, profile your script, often, and verify every assumptions by yourself, do not blindly apply some performance patch assuming that...? Check by yourself.

We, at Blackfire engineer team, spend many time finding interesting metrics to show our PHP users. We use our deep knowledge of the PHP
engine to gather many items showing you what's happening into the deepness of your PHP scripts.
Even if the GUI doesn't show it yet, Blackfire probe extension measures many things, like when the garbage collector has been triggered, what it does, how many objects you created/destroyed in your functions, how many reference mismatch you performed when calling your functions, and even deeper metrics, such as session serialization time, ``foreach()`` bad behaviors (there are many things to say about foreach())... basicaly, everything that may help a developer find performance spots, and how to fix them.

Also, don't forget that you will at some point hit the limits of the language. Then it will probably be time to switch it.
PHP is not the right language to build an ORM, to make a video game, to create an HTTP server, to batch process some text based files or to create a parser of any kind.
It can do it, but it can't do it efficiently, under a high load. You'll hit a limit that is really close compared to
the one a better suitable language for such tasks will show, aka Java, Go or probably the most efficient language in the world nowadays, should you really well use it : C/C++ (Both Java and Go are also written in C/C++).
