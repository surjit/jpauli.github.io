---
layout: post
title:  Exceptional PHP
---

## PHP Exceptions in short

Back in 2004, PHP5 came out with a new model object. This latter allows PHP users to make use of a well known OO paradigm : Exceptions.
The Exceptions model has then not been really reworked. PHP 5.3 introduced stacked Exceptions, which is a very nice improvement in stack traces analysis, and PHP 5.5 added the "*finally*" feature.
As you know how Exceptions work with PHP, because you are a heavy user of the PHP language, let me show you how this big feature works in the PHP source code, into the Zend Engine : how it's been implemented.

## Exception : this so particular class

You know that the Exception class (and all of its possible children) is very special into the engine, because it is the only one that may react with the *try-throw-catch-finally* PHP language constructs.
Also, you know the Exception class automagically builds a stacktrace when an object is built from it.
Let's recall its structure :

	> php --rc Exception
	Class [ <internal:Core> class Exception ] {

	  - Constants [0] { }

	  - Static properties [0] { }

	  - Static methods [0] { }

	  - Properties [7] {
		Property [ <default> protected $message ]
		Property [ <default> private $string ]
		Property [ <default> protected $code ]
		Property [ <default> protected $file ]
		Property [ <default> protected $line ]
		Property [ <default> private $trace ]
		Property [ <default> private $previous ]
	  }

	  - Methods [10] {
		Method [ <internal:Core> final private method __clone ] {
		}

		Method [ <internal:Core, ctor> public method __construct ] {

		  - Parameters [3] {
		    Parameter #0 [ <optional> $message ]
		    Parameter #1 [ <optional> $code ]
		    Parameter #2 [ <optional> $previous ]
		  }
		}

		Method [ <internal:Core> final public method getMessage ] {
		}

		Method [ <internal:Core> final public method getCode ] {
		}

		Method [ <internal:Core> final public method getFile ] {
		}

		Method [ <internal:Core> final public method getLine ] {
		}

		Method [ <internal:Core> final public method getTrace ] {
		}

		Method [ <internal:Core> final public method getPrevious ] {
		}

		Method [ <internal:Core> final public method getTraceAsString ] {
		}

		Method [ <internal:Core> public method __toString ] {
		}
	  }
	}

As you can see, lots of methods are final : you may not change the default behavior, dictated by the engine source code.
You cannot clone Exceptions, and if you extend the class, some attributes are private : you can't change them.

This is because the engine expects some well defined behavior when it comes to play with an Exception object, or one of its children.

You also know that Exception is the base class of every other exceptions, SPL ones, foobar ones, or yours.
This is checked when one wants to throw something, for PHP extensions one must call `zend_throw_exception_object()`, and for PHP userland code the compiler will anyway lead to this same function :

	ZEND_API void zend_throw_exception_object(zval *exception TSRMLS_DC)
	{
		zend_class_entry *exception_ce;

		if (exception == NULL || Z_TYPE_P(exception) != IS_OBJECT) {
			zend_error(E_ERROR, "Need to supply an object when throwing an exception");
		}

		exception_ce = Z_OBJCE_P(exception);

		if (!exception_ce || !instanceof_function(exception_ce, default_exception_ce TSRMLS_CC)) {
			zend_error(E_ERROR, "Exceptions must be valid objects derived from the Exception base class");
		}
		zend_throw_exception_internal(exception TSRMLS_CC);
	}

So, you can't throw something which is not an object, and you can't throw an object which doesn't have the Exception class as an ancestor. Things are somewhat "locked" here.

## A little tour of Exception's internals

Internally, Exceptions are nothing more than objects. They however define custom Zend Engine object handlers, which provide them this so particular behavior.
When the engine starts, among many other boots, it boots the exceptions by calling `zend_register_default_exception()` :

	void zend_register_default_exception(TSRMLS_D)
	{
		zend_class_entry ce;

		INIT_CLASS_ENTRY(ce, "Exception", default_exception_functions);
		default_exception_ce = zend_register_internal_class(&ce TSRMLS_CC);
		default_exception_ce->create_object = zend_default_exception_new;
		memcpy(&default_exception_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
		default_exception_handlers.clone_obj = NULL;

		/* ... */
	}

As we can see, this function registers the Exception class, and puts the clone handler to NULL. That means (somehow) that the Exception class is not clonable. Also, the `__clone()` method which is added to the class (not shown here) is passed final, so that you can't redefine it, Exceptions are not clonable, that's a fact.

We can notice as well that we overwrite the *create_object* handler. This handler is triggered when an object will be created from the class, and before the userland constructor : `__construct()`.
That means that even if you extend the class, redefine the constructor, and forget to call the parent one, the Exception object will be just right to the eyes of the engine.
Here is the *create_object* handler :

	static zend_object_value zend_default_exception_new_ex(zend_class_entry *class_type, int skip_top_traces TSRMLS_DC)
	{
		zval obj;
		zend_object *object;
		zval *trace;

		Z_OBJVAL(obj) = zend_objects_new(&object, class_type TSRMLS_CC);
		Z_OBJ_HT(obj) = &default_exception_handlers;

		object_properties_init(object, class_type);

		ALLOC_ZVAL(trace);
		Z_UNSET_ISREF_P(trace);
		Z_SET_REFCOUNT_P(trace, 0);
		zend_fetch_debug_backtrace(trace, skip_top_traces, 0, 0 TSRMLS_CC);

		zend_update_property_string(default_exception_ce, &obj, "file", sizeof("file")-1, zend_get_executed_filename(TSRMLS_C) TSRMLS_CC);
		zend_update_property_long(default_exception_ce, &obj, "line", sizeof("line")-1, zend_get_executed_lineno(TSRMLS_C) TSRMLS_CC);
		zend_update_property(default_exception_ce, &obj, "trace", sizeof("trace")-1, trace TSRMLS_CC);

		return Z_OBJVAL(obj);
	}

What is interesting to note, is that we fetch the backtrace (`zend_fetch_debug_backtrace()`) immediately when the object is created, and *not* when the object is thrown.
Have a look :

	function bar(Exception $a)
	{
		throw $a;
	}
	function baz($a)
	{
		throw new Exception('foo');
	}
	function foo($a)
	{
		return new Exception('foo');
	}
	$b = foo('a');

	bar($b); /* first */
	baz($b); /* second */

`bar()` and `baz()` cant be run following as `bar()` will throw an exception and stop the script. But if you run this code with only `bar()` call, you get this

	Fatal error: Uncaught exception 'Exception' with message 'foo' in /tmp/php.php:13
	Stack trace:
	#0 /tmp/php.php(15): foo('a')
	#1 {main}
	  thrown in /tmp/php.php on line 13

But with a call to `baz()` instead of `bar()`, you'll get this :

	Fatal error: Uncaught exception 'Exception' with message 'foo' in /tmp/php.php:9
	Stack trace:
	#0 /tmp/php.php(17): baz(Object(Exception))
	#1 {main}
	  thrown in /tmp/php.php on line 9

As you can see, the line numbers in the stack trace are not the same, they always match the Exception construction and not the Exception throw.


## Exceptions into the Zend Virtual Machine Executor

If you don't know how the Zend Virtual Machine Executor works, you should read [the related article](jpauli.github.io/2015/02/05/zend-vm-executor.html).
Now that you have a glance on how the executor works, well, Exceptions are not hard to understand.

### Throwing an Exception

To throw an Exception, one may use the `throw` PHP keyword, compiled as a `ZEND_THROW` OPCode.
`ZEND_THROW` simply gets the exception, and adds it into the *engine executor exception slot*. This slot is alone : weither there is an exception into it, or not. There can't be several exceptions into the slot, the basic line is binary : weither or not there is an exception pending (not 2, 3 or 42).

The slot is accessible using the `EG(exception)` macro. If it is NULL : no exception have been thrown so far.
Here is `ZEND_THROW`, simplified :

	ZEND_VM_HANDLER(108, ZEND_THROW, CONST|TMP|VAR|CV, ANY)
	{
		/* ... */
		zend_exception_save(TSRMLS_C);
		ALLOC_ZVAL(exception);
		INIT_PZVAL_COPY(exception, value);
		if (!IS_OP1_TMP_FREE()) {
			zval_copy_ctor(exception);
		}

		zend_throw_exception_object(exception TSRMLS_CC);
		zend_exception_restore(TSRMLS_C);
		/* ... */
	}

So basically things are easy, the executor code leads to `zend_throw_exception_object()` , which will quickly lead to `zend_throw_exception_internal()` :

	void zend_throw_exception_internal(zval *exception TSRMLS_DC)
	{
		if (exception != NULL) {
			zval *previous = EG(exception);
			zend_exception_set_previous(exception, EG(exception) TSRMLS_CC);
			EG(exception) = exception; /* Fill in the executor slot */
			if (previous) {
				return;
			}
		}
		if (!EG(current_execute_data)) {
			if(EG(exception)) {
				zend_exception_error(EG(exception), E_ERROR TSRMLS_CC);
			}
			zend_error(E_ERROR, "Exception thrown without a stack frame");
		}

		/* ... */
		
		/* Change the VM path next to execute */
		EG(opline_before_exception) = EG(current_execute_data)->opline;
		EG(current_execute_data)->opline = EG(exception_op);
	}

That's it : `EG(exception)` is beeing filled with the thrown exception. And the very important things are the two last lines.

	/* Change the VM path next to execute */
	EG(opline_before_exception) = EG(current_execute_data)->opline;
	EG(current_execute_data)->opline = EG(exception_op);

We just threw an Exception, so now, the engine is not expected to continue on the next instruction like if nothing had happened, but to treat the exception : run the catch block (if any), run the finally block (if any), or continue running if none of those blocks are present.

This is done by changing the opline of the executor, and making it point to `EG(exception_op)`, which value is always the same : `ZEND_HANDLE_EXCEPTION`

### Handling an Exception 

So now, whatever was compiled next (whatever the PHP lines of code following the `throw` keyword), the engine will run `ZEND_HANDLE_EXCEPTION`.

This handler is a little bit big, because of some memory management routines. Simplified, it gives this :

	ZEND_VM_HANDLER(149, ZEND_HANDLE_EXCEPTION, ANY, ANY)
	{
		zend_uint op_num = EG(opline_before_exception)-EG(active_op_array)->opcodes;
		int i;
		zend_uint catch_op_num = 0, finally_op_num = 0;
		void **stack_frame;

		/* ... */

		for (i=0; i<EG(active_op_array)->last_try_catch; i++) {
			if (EG(active_op_array)->try_catch_array[i].try_op > op_num) {
				/* further blocks will not be relevant... */
				break;
			}
			if (op_num < EG(active_op_array)->try_catch_array[i].catch_op) {
				catch_op_num = EX(op_array)->try_catch_array[i].catch_op;
			}
			if (op_num < EG(active_op_array)->try_catch_array[i].finally_op) {
				finally_op_num = EX(op_array)->try_catch_array[i].finally_op;
			}
		}

		/* ... */
	}

We can see that this Zend Virtual Machine Executor handler's job is to find into the OPArray the very next `catch` block and `finally` block. Don't forget that such blocks may be nested : you can nest *try-catch-finally* structures, and it would be better in such case to run the *catch* block corresponding to the right *try* wouldn't it ?

So we search for the next catch block as *catch_op_num* (if any), and the next finally block as *finally_op_num* (if any).
The procedure is fast, because the compiler already filled-in the index of the *last_try_catch*, and every try-catch block is compiled into the *try_catch_array* of the current OPArray : the executor has here a little job to do, all the hard work has been computed by the compiler earlier.

### Catch

Continuing the source code of `ZEND_HANDLE_EXCEPTION` :

	/* ... */
	
	if (finally_op_num && (!catch_op_num || catch_op_num >= finally_op_num)) {
		zend_exception_save(TSRMLS_C);
		EX(fast_ret) = NULL;
		ZEND_VM_SET_OPCODE(&EX(op_array)->opcodes[finally_op_num]);
		ZEND_VM_CONTINUE();
	} else if (catch_op_num) {
		ZEND_VM_SET_OPCODE(&EX(op_array)->opcodes[catch_op_num]);
		ZEND_VM_CONTINUE();
	} else {
		if (UNEXPECTED((EX(op_array)->fn_flags & ZEND_ACC_GENERATOR) != 0)) {
			ZEND_VM_DISPATCH_TO_HANDLER(ZEND_GENERATOR_RETURN);
		} else {
			ZEND_VM_DISPATCH_TO_HELPER(zend_leave_helper);
		}
	}

Well, this is now very clear.
If there is a finally block and no catch block, or if the catch block appears after the finally block, make the executor jump to the finally block, and run it

	ZEND_VM_SET_OPCODE(&EX(op_array)->opcodes[finally_op_num]);
	ZEND_VM_CONTINUE();

If there is no finally block but a catch block, make the executor jump to this catch block, and run it

	ZEND_VM_SET_OPCODE(&EX(op_array)->opcodes[catch_op_num]);
	ZEND_VM_CONTINUE();

Else, no catch nor finally ? Then make the executor runs the `zend_leave_helper`, which will basically acts like a `return` statement : it will stop the current execution and return from it, quickly leading to the "Uncaught Exception" error.

The catch block is compiled as a `ZEND_CATCH` opcode. This latter will barely fetch the Exception class you declared into your catch block, and browse its inheritence tree to see if it will run the block or not.

	ZEND_VM_HANDLER(107, ZEND_CATCH, CONST, CV)
	{
		/* ... */
		catch_ce = zend_fetch_class_by_name(Z_STRVAL_P(opline->op1.zv), Z_STRLEN_P(opline->op1.zv), opline->op1.literal + 1, ZEND_FETCH_CLASS_NO_AUTOLOAD TSRMLS_CC);

		ce = Z_OBJCE_P(EG(exception));

		if (ce != catch_ce) {
			if (!instanceof_function(ce, catch_ce TSRMLS_CC)) {
				if (opline->result.num) {
					zend_throw_exception_internal(NULL TSRMLS_CC);
					HANDLE_EXCEPTION();
				}
				ZEND_VM_SET_OPCODE(&EX(op_array)->opcodes[opline->extended_value]);
				ZEND_VM_CONTINUE();
			}
		}
		exception = EG(exception);
		/* ... */
		if (UNEXPECTED(EG(exception) != exception)) {
			Z_ADDREF_P(EG(exception));
			HANDLE_EXCEPTION();
		} else {
			EG(exception) = NULL;
			ZEND_VM_NEXT_OPCODE();
		}
	}

So here, we try to load the class which is declared into the catch block, without triggering the autoload (this was a change from PHP 5.0 which triggered the autoload).

If it is the right class, or one of its ancestor : this catch block is ours, we must run its code : we then use `ZEND_VM_NEXT_OPCODE()` to tell the executor to fetch the next instruction and run it.

If it is not our Exception, then we must not run this current catch block : it is not ours. We then "rethrow" the Exception internally : `zend_throw_exception_internal()`. That will make the VM Executor re-run the `ZEND_HANDLE_EXCEPTION` OPCode, but this time, starting from our catch block : it will then re-do its operation, considerating the (eventually) next catch block , etc.. etc... until it finds the right catch block (if any) to run code from.

Important thing : if it is our catch block and we run its code, we reset the exception slot : `EG(exception) = NULL;`

### finally

*Finally* is a little bit harder to understand and has not been trivial to add to PHP 5.5.

The trick is that after having run the *finally* block, we must weither continue execution if a catch treated the exception, or stop the execution if the Exception was actually uncaught.

To achieve this, we introduced two new OPCodes in PHP5.5, just for the *finally* feature : `ZEND_FAST_CALL` and `ZEND_FAST_RETURN`, and we also play with `ZEND_JMP`.

So here is the code the Zend Compiler generates for finally block :

	void zend_do_finally(znode *finally_token TSRMLS_DC)
	{
		zend_op *opline = get_next_op(CG(active_op_array) TSRMLS_CC);

		finally_token->u.op.opline_num = get_next_op_number(CG(active_op_array));
		/* call the the "finally" block */
		opline->opcode = ZEND_FAST_CALL;
		SET_UNUSED(opline->op1);
		opline->op1.opline_num = finally_token->u.op.opline_num + 1;
		SET_UNUSED(opline->op2);
		/* jump to code after the "finally" block,
		 * the actual jump address is going to be set in zend_do_end_finally()
		 */
		opline = get_next_op(CG(active_op_array) TSRMLS_CC);
		opline->opcode = ZEND_JMP;
		SET_UNUSED(opline->op1);
		SET_UNUSED(opline->op2);

		CG(context).in_finally++;
	}
	
	void zend_do_end_finally(znode *try_token, znode* catch_token, znode *finally_token TSRMLS_DC)
	{
		if (catch_token->op_type == IS_UNUSED && finally_token->op_type == IS_UNUSED) {
			zend_error(E_COMPILE_ERROR, "Cannot use try without catch or finally");
		}
		if (finally_token->op_type != IS_UNUSED) {
			zend_op *opline;

			CG(active_op_array)->try_catch_array[try_token->u.op.opline_num].finally_op = finally_token->u.op.opline_num + 1;
			CG(active_op_array)->try_catch_array[try_token->u.op.opline_num].finally_end = get_next_op_number(CG(active_op_array));
			CG(active_op_array)->has_finally_block = 1;

			opline = get_next_op(CG(active_op_array) TSRMLS_CC);
			opline->opcode = ZEND_FAST_RET;
			SET_UNUSED(opline->op1);
			SET_UNUSED(opline->op2);

			CG(active_op_array)->opcodes[finally_token->u.op.opline_num].op1.opline_num = get_next_op_number(CG(active_op_array));

			CG(context).in_finally--;
		}
	}

What the compiler does is simple :
*	When we meet a finally block, we generate a `ZEND_FAST_CALL` plus a `ZEND_JMP`
*	We keep compiling the instructions into the finally block, normally, nothing to say
*	At the end of the finally block, we generate a `ZEND_FAST_RET` OPCode

We have just wrapped the finally block between two markers that will help the executor to find its path.

Here is a simplified compiler output :

	try {
		throw new Exception('foo');
	} catch (Exception $e) {
		echo "catch block";
	} finally {
		echo 'finally block';
	}

	echo "end";

	line     #* I O op                           fetch          ext  return  operands
	-----------------------------------------------------------------------------------
		     4    > THROW                                         0          $1
	   5     5*     JMP                                                      ->8
		     6  >   CATCH                                         8          'Exception', !0
	   6     7      ECHO                                                     'catch+block'
	   7     8      FAST_CALL                                                ->10
		     9    > JMP                                                      ->12
	   8    10*     ECHO                                                     'finally+block'
	   9    11*     FAST_RET                                                 
	  11    12  >   ECHO                                                     'end'
	  12    13    > RETURN                                                   1


When the executor comes to run the finally block, it then first runs the `ZEND_FAST_CALL` OPCode, it looks like this :

	ZEND_VM_HANDLER(162, ZEND_FAST_CALL, ANY, ANY)
	{
		USE_OPLINE

		if ((opline->extended_value & ZEND_FAST_CALL_FROM_CATCH) &&
			UNEXPECTED(EG(prev_exception) != NULL)) {
			/* in case of unhandled exception jump to catch block instead of finally */
			ZEND_VM_SET_OPCODE(&EX(op_array)->opcodes[opline->op2.opline_num]);
			ZEND_VM_CONTINUE();
		}
		EX(fast_ret) = opline;
		ZEND_VM_SET_OPCODE(opline->op1.jmp_addr);
		ZEND_VM_CONTINUE();
	}

What we do in the normal flow, is memorize the current opcode beeing dispatched, that is ourselves : `ZEND_FAST_CALL`, we save this OPCode index (into the OPArray) using `EX(fast_ret)`.
Then, we make the executor go to the OPCode stored into a jump address computed by the compiler : we make the executor run the finally block, just normally.

The trick is that this finally block is finished by a `ZEND_FAST_RET` OPCode, the compiler generated it as we saw. Here it is :

	ZEND_VM_HANDLER(163, ZEND_FAST_RET, ANY, ANY)
	{
		if (EX(fast_ret)) {
			ZEND_VM_SET_OPCODE(EX(fast_ret) + 1);
			if ((EX(fast_ret)->extended_value & ZEND_FAST_CALL_FROM_FINALLY)) {
				EX(fast_ret) = &EX(op_array)->opcodes[EX(fast_ret)->op2.opline_num];
			}
			ZEND_VM_CONTINUE();
		} else {
			/* special case for unhandled exceptions */
			USE_OPLINE

			if (opline->extended_value == ZEND_FAST_RET_TO_FINALLY) {
				ZEND_VM_SET_OPCODE(&EX(op_array)->opcodes[opline->op2.opline_num]);
				ZEND_VM_CONTINUE();
			} else if (opline->extended_value == ZEND_FAST_RET_TO_CATCH) {
				zend_exception_restore(TSRMLS_C);
				ZEND_VM_SET_OPCODE(&EX(op_array)->opcodes[opline->op2.opline_num]);
				ZEND_VM_CONTINUE();
			} else if (UNEXPECTED((EX(op_array)->fn_flags & ZEND_ACC_GENERATOR) != 0)) {
				zend_exception_restore(TSRMLS_C);
				ZEND_VM_DISPATCH_TO_HANDLER(ZEND_GENERATOR_RETURN);
			} else {
				zend_exception_restore(TSRMLS_C);
				ZEND_VM_DISPATCH_TO_HELPER(zend_leave_helper);
			}
		}
	}

So here now, if the exception was caught, `EX(fast_ret)` is still here. We fall into the `if` part and tell the executor to run the next OPCode using `ZEND_VM_SET_OPCODE(EX(fast_ret) + 1);` , and this next OPCode is a `ZEND_JMP` that will make it jump after the finally block.

If the exception was uncaught, that is we ran the finally block but no catch before, `ZEND_HANDLE_EXCEPTION` will have erased `EX(fast_ret)` setting it to NULL, we then fall into the `else` part and will barely return.

### Uncaught Exceptions

Uncaught Exceptions are easy to understand. Like we have seen in the past chapters, uncaught exceptions are managed by the executor calling `zend_leave_helper` , which will make it return from the current execute context.

At the end of the just-run context, we simply sniff the execution slot. If it is filled in with an exception, well, this one has not been handled (caught).
We then weither call the user exception handler, or generate our fatal error with a call to `zend_exception_error()` :

	/* ... */
	if (EG(exception)) {
		if (EG(user_exception_handler)) {
			zval *orig_user_exception_handler;
			zval **params[1], *retval2, *old_exception;
			old_exception = EG(exception);
			EG(exception) = NULL;
			params[0] = &old_exception;
			orig_user_exception_handler = EG(user_exception_handler);
			if (call_user_function_ex(CG(function_table), NULL, orig_user_exception_handler, &retval2, 1, params, 1, NULL TSRMLS_CC) == SUCCESS) {
				if (retval2 != NULL) {
					zval_ptr_dtor(&retval2);
				}
				if (EG(exception)) {
					zval_ptr_dtor(&EG(exception));
					EG(exception) = NULL;
				}
				zval_ptr_dtor(&old_exception);
			} else {
				EG(exception) = old_exception;
				zend_exception_error(EG(exception), E_ERROR TSRMLS_CC);
			}
		} else {
			zend_exception_error(EG(exception), E_ERROR TSRMLS_CC);
		}
	}

## PHP Exceptions tips and tricks

Ok, here we go. Like always, PHP is not perfect, and can't be either. There are some scenarios that are known to be a little bit strange.

### Stack traces

We've seen that the stack trace generation -which itself is a heavy and complex subject into the Zend Engine- is generated when the Exception object gets constructed. The stack trace is added to the `trace` private attribute. This is to be remembered, because now, when you will try to serialize an Exception (if you want to), then any of its attribute will get serialized (that's PHP default behavior you know about), but what about this `trace` attribute ?
The trick is that this latter may not be serializable, if the stack trace of the Exception contains something which is not serializable, like a SimpleXMLElement object, or a PDO one.

Have a look :

	$xml = new SimpleXMLElement('<foo></foo>');
	 
	function hey($arg)
	{
	  $s = serialize(new Exception());
	}
	 
	hey($xml);

This code lead to this error :

	Fatal error: Uncaught exception 'Exception' with message 'Serialization of 'SimpleXMLElement' is not allowed' in foo.php

Now you know why. If you want to serialize Exception for whatever reason, take care that the stack trace into the Exception object may be huge and not serializable.

### Exception thrown without a stack frame

This used to be a strange behavior in PHP < 5.3. Since PHP 5.3 however, this error should not show again with vanilla PHP, but some badly designed custom extensions could still lead to such an error message to appear.

### Using Reflection to crash PHP with Exceptions.

This code crashes the engine :

	$e  = new ReflectionClass('Exception');
	$ex = new Exception('foo');

	$file = $e->getProperty('trace');
	$file->setAccessible(true);
	$file->setValue($ex, new stdclass);

	throw $ex;

And this is perfectly normal if you followed the whole article.
The Exception class (object), is a very particular object that the Engine will manipulate internally for its own work.
The engine does not expect the Exception object to have its `trace` property beeing something else that a well formed array.
If you use the reflection API to change one of its private member's type, you are very likely to generate bad behavior, such as memory access failures into the engine, and crash it like the script above does.

We have been reported such a bug in the past, and we decided not to fix it, mainly for always the same reasons here :
*	This is a very tricky use case, which may not be used everyday
*	This code snippet trying to change the Exception internals brings nothing useful but is clearly hackish
*	Fixing such a case would lead to a massive code change all over the engine, which tells us it is not worth the use case, too uncommon.

PHP is a tradeoff you know, like every computer program.

## End

This post tried to explain you how Exceptions have been handled into the engine. We saw some Zend Compiler generated code, and analyzed Zend Executor handlers code to understand how things work into the VM.
As you know, the Zend VM is a big dispatcher loop which control flow may be altered at any time, using some macros such as `ZEND_VM_SET_OPCODE()`, `ZEND_VM_RETURN()` etc...
That's how Exceptions have been designed; with a little bit more tricky *finally* feature, requiring some extra OPCodes and some JMP playings.

Once more, the goal was to have a nice feature, usefull to people, but also performant : we've seen that once more, many hard and slow work is done into the compiler : *try-catch-finally* OPCode browsing loops, jump address computations, etc...
We can then say that Exceptions are not a slow feature into PHP, but a well optimized one. You may use them whenever you want, take care however of some little bit strange but uncommon behaviors about them.
