---
layout: post
title:  PHP closures
---

## History

Back in 2009, when PHP 5.3 got released, a new feature (among many others) were introduced : anonymous functions (also called lambdas or closures).

The feature was very expected, as closures have proved their utility through several other languages, particularly javascript that web developers master.

But there has been a problem when developing such a feature into the heart of PHP : how could we add support for anonymous functions, without rewriting a big part of the engine ?
Because, you probably don't notice it (yet), but adding support for closures was not that simple. You could naively think as the final result would tell you to think : "Well, this is just
a normal function, but with no name". Heh, that's more than that.

Just the fact that this function can be assigned to a variable, must make us think there is a lot of work. Add a new type ? What type would then be the variable, when assigned a closure ?
That's impossible to add a new type, that's way too complex, and that would break for sure tons of things. PHP 5.3 was thought as being a minor, not a major, things could not break like that.

Also, the other problem was how to reuse things ? How to start from classical PHP functions source code base, and extend it in a way that we could add anonymous functions support, without changing
the general meaning of a "function" into the Zend engine ?

Let's see together how Closures have been added to PHP, as usual by turning to the truth : the PHP source code.

## The work to do for closures to get born

Some work need to be done in the parser (new syntax), in the compiler (new syntax to take care of,  new structures to use, and probably new OPCode to generate), in the executor and elsewhere in the engine.
Let's start.

### Choose a type, use objects

If the user would be able to assign a function to a variable, that means we need a type for this variable.

	$a = function() { }; /* what type for $a ? */

You know that PHP's anonymous functions are of type "Object", more accurately "object of class Closure".
What is amazing into the Zend engine, is that when the second version has been thought and built - Zend engine 2 (PHP 5.0, 2004) - its creators pushed one big topic very far : the object model.
The Zend engine 2 object model is so extensible, that critical features that have been added later to PHP have been built on top of this 2004-designed object model (2004 is the release date of PHP5, foundations date back from 2001-2002).
Anonymous functions, and PHP 5.5 Generators are two critical features of the PHP language that benefit from the Zend engine 2 object model extensibility, and that could be added thanks to it.

### Objects being callable

Now that we chose the object type, we must patch the engine to tell it that when a function call is made, it should allow it to be made on an object (that wasnt the case before).

	$a();
	/* Before PHP5.3, $a could only be of type array or string here */

So we need to patch the executor OPCode that is responsible of launching functions, and tell it that now we not only support strings or arrays, but also objects.

But, the problem will then be : what the hell could the engine do with an object given, as it is about to launch a PHP function ?
The answer can only be by using a nicely thought extensibility point of ZE2 : object handlers.

A new object handler has been added in PHP5.3, that every object of every class can take advantage of : the ``get_closure()`` handler :

	typedef int (*zend_object_get_closure_t)(zval *obj, zend_class_entry **ce_ptr, union _zend_function **fptr_ptr, zval **zobj_ptr TSRMLS_DC);

And with every new handler, usually comes a default implementation of it, that is used when nobody tells to use anything else. It is called ``zend_std_get_closure()``

	#define ZEND_INVOKE_FUNC_NAME "__invoke"
	
	int zend_std_get_closure(zval *obj, zend_class_entry **ce_ptr, zend_function **fptr_ptr, zval **zobj_ptr TSRMLS_DC)
	{
		zend_class_entry *ce;
		if (Z_TYPE_P(obj) != IS_OBJECT) {
			return FAILURE;
		}

		ce = Z_OBJCE_P(obj);

		if (zend_hash_find(&ce->function_table, ZEND_INVOKE_FUNC_NAME, sizeof(ZEND_INVOKE_FUNC_NAME), (void**)fptr_ptr) == FAILURE) {
			return FAILURE;
		}

		*ce_ptr = ce;
		if ((*fptr_ptr)->common.fn_flags & ZEND_ACC_STATIC) {
			if (zobj_ptr) {
				*zobj_ptr = NULL;
			}
		} else {
			if (zobj_ptr) {
				*zobj_ptr = obj;
			}
		}
		return SUCCESS;
	}

As you may have spotted it, what this handler does is that it looks up the class function table, to try to find a function named ``__invoke()``. If found, it fills in the **fptr_ptr** pointer the engine is expecting to be given back : that's the pointer to the function to run. What we can notice as well, is that the engine will fill-in ``ce_ptr`` and ``zobj_ptr`` , which both represent the calling scope and the current object to feed ``$this`` in. We'll talk about those later on.

The engine executor makes use of this handler and the function pointer it retrieves, in the OPCode that is dispatched when a dynamic function call is made : **ZEND_INIT_FCALL_BY_NAME**
The engine checks the argument type, against a string representing a function, against an array representing a class and a method to launch, and finally against an object defining a ``get_closure()`` handler :

	ZEND_VM_HANDLER(59, ZEND_INIT_FCALL_BY_NAME, ANY, CONST|TMP|VAR|CV)
	{

	/* ... here, code to take care of string and array arguments ... */

	} else if (OP2_TYPE != IS_CONST && OP2_TYPE != IS_TMP_VAR &&
		EXPECTED(Z_TYPE_P(function_name) == IS_OBJECT) && /* we are an object ? */
		Z_OBJ_HANDLER_P(function_name, get_closure) && /* we have a get_closure() handler ? */
		Z_OBJ_HANDLER_P(function_name, get_closure)(function_name, &call->called_scope, &call->fbc, &call->object TSRMLS_CC) == SUCCESS) { 
			/* the handler can take care of the call ? */
		
		/* ... */
	}

### The Closure class and the zend_closure object

So far so good, we know that the engine is able to call a function onto an object, and for that it looks if this object has a ``get_closure()`` handler, which default implementation tells it to look for (and make use of) an ``__invoke()`` function.

But with just those points, we haven't got closures yet.
What we have however, are the mechanisms needed for objects to be invokable. That is, now, any user may design a class, declare a special magic method "__invoke()", and then run the object from it as a function. Just like this :

	class Foo {
		public function __invoke() { }
	}
	
	$f = new Foo;
	$f();

This will work, because the default ``zend_std_get_closure()`` handler is added to every class not telling something else, so it is added to every userland class (class designed using the PHP language).
It can be different when designing PHP extensions, where in we can change this default handler to provide another one of ours.

For real PHP anonymous functions to work, there needs to be a base class every closure will use, and some more magic.
Let's have a tour and meet the ``zend_closure`` object.

	typedef struct _zend_closure {
		zend_object    std;
		zend_function  func;
		zval          *this_ptr;
		HashTable     *debug_info;
	} zend_closure;

This is a Closure into the Zend Engine. A standard object, then a function : ``zend_function``, a pointer to ``$this`` into the anonymous function body and some debug info.

That's all needed for the engine to work, but there is once again more work to do. We forgot two things here : what about PHP's reflection ? And what about PHP's anonymous function type ?

To fully support closures and merge their concept into the engine, we need a base ``Closure`` class.
That class is just here, because anonymous functions must be of type "something", and the type "object" has been chosen, then a class is needed.
If we don't talk about PHP's reflection, and other PHP features (like the rebinding of ``$this`` into closures), what we simply need is a nearly empty class, to support PHP's anonymous functions.
A class which could'nt be user manipulable, that's why we declared it as final, and we changed nearly all its handlers to forbid everything on it. Have a look :

	void zend_register_closure_ce(TSRMLS_D)
	{
		zend_class_entry ce;

		INIT_CLASS_ENTRY(ce, "Closure", closure_functions); /* the class is named "Closure" */
		zend_ce_closure = zend_register_internal_class(&ce TSRMLS_CC);
		zend_ce_closure->ce_flags |= ZEND_ACC_FINAL_CLASS; /* The class is final */
		zend_ce_closure->create_object = zend_closure_new; /* function to run when an object is created */
		zend_ce_closure->serialize = zend_class_serialize_deny; /* function to run on serialize() call */
		zend_ce_closure->unserialize = zend_class_unserialize_deny; /* function to run on unserialize() call */

		memcpy(&closure_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
		closure_handlers.get_constructor = zend_closure_get_constructor; /* function to run when fetching the constructor */
		closure_handlers.get_method = zend_closure_get_method; /* function to run when fetching a method */
		closure_handlers.write_property = zend_closure_write_property; /* function to run when willing to write to an attribute */
		closure_handlers.read_property = zend_closure_read_property; /* function to run when willing to read from an attribute */
		closure_handlers.get_property_ptr_ptr = zend_closure_get_property_ptr_ptr; /* function to run when fetching an attribute */
		closure_handlers.has_property = zend_closure_has_property; /* function to run when willing to test for an attribute existence */
		closure_handlers.unset_property = zend_closure_unset_property; /* function to run when willing to unset an attribute */
		closure_handlers.compare_objects = zend_closure_compare_objects; /* function to run when comparing two objects of this class */
		closure_handlers.clone_obj = zend_closure_clone; /* function to run when willing to clone an object of this class */
		closure_handlers.get_debug_info = zend_closure_get_debug_info; /* function to run when var_dump()ing an object of this class*/
		closure_handlers.get_closure = zend_closure_get_closure; /* function to run when willing to launch a method call from an object of this class */
		closure_handlers.get_gc = zend_closure_get_gc; /* function to run when the garbage collector fires in */
	}

You can see that when registering this class into the engine (automaticaly done when the engine starts), it provides ``zend_class_serialize_deny()`` and ``zend_class_unserialize_deny()``. So it will be impossible to both ``serialize()`` and ``unserialize()`` any objects of Closure.
Also, the class is declared final (it is added the **ZEND_ACC_FINAL_CLASS** flag), so the PHP user won't be able to extend from it.

And finally, many other handlers are redefined, for example, the ``get_constructor()`` handler -which is called by the engine when it needs to create an object from the class- is designed like this :

	static zend_function *zend_closure_get_constructor(zval *object TSRMLS_DC)
	{
		zend_error(E_RECOVERABLE_ERROR, "Instantiation of 'Closure' is not allowed");
		return NULL;
	}

Again, if you try to fetch, read, write or unset an attribute into an object of the Closure class ( that is : a PHP anonymous function), you will be warned :

	#define ZEND_CLOSURE_PROPERTY_ERROR() \
	zend_error(E_RECOVERABLE_ERROR, "Closure object cannot have properties")
	
	static zval *zend_closure_read_property(zval *object, zval *member, int type, const zend_literal *key TSRMLS_DC)
	{
		ZEND_CLOSURE_PROPERTY_ERROR();
		Z_ADDREF(EG(uninitialized_zval));
		return &EG(uninitialized_zval);
	}

	static void zend_closure_write_property(zval *object, zval *member, zval *value, const zend_literal *key TSRMLS_DC)
	{
		ZEND_CLOSURE_PROPERTY_ERROR();
	}

	static zval **zend_closure_get_property_ptr_ptr(zval *object, zval *member, int type, const zend_literal *key TSRMLS_DC)
	{
		ZEND_CLOSURE_PROPERTY_ERROR();
		return NULL;
	}

	static int zend_closure_has_property(zval *object, zval *member, int has_set_exists, const zend_literal *key TSRMLS_DC)
	{
		if (has_set_exists != 2) {
			ZEND_CLOSURE_PROPERTY_ERROR();
		}
		return 0;
	}

	static void zend_closure_unset_property(zval *object, zval *member, const zend_literal *key TSRMLS_DC)
	{
		ZEND_CLOSURE_PROPERTY_ERROR();
	}

Try it :

	$a = function() { };
	$a->foo = 'bar';  /* Fatal error : Closure object cannot have properties */

Ok, here we can see that once more, thanks to Zend engine 2 object handlers, we managed to lock every operations the user could try to perform on such a class (and objects of it).

### And the magic happens

Now, we're gonna see how the compiler compiles a PHP function into an object of class Closure when it sees it is an anonymous function, and how it is then launched.

When the compiler compiles a function, its main role is to create a ``zend_function`` based variable, and fill-in its OPArray. The OPArray represent all the instructions that are part of the function body.
So the compiler starts by firing a ``zend_do_begin_function_declaration()`` call , then it parses the body of the function, and finally it calls for ``zend_do_end_function_declaration()`` and ``zend_do_early_binding()``. That's true for classical PHP functions.

For anonymous functions though, the steps that change from regular functions is that the compiler will name the function "{closure}" (as it couldn't fetch a name from the syntax, because you know : "anonymous" function) , and it will NOT register that function name into the global function table (this step is done while early binding the function).

So the compiler will compile many anonymous functions, without adding their name to the global function table (obviously : they all got the same name). That seems like leaking and doing useless things. Not really, as the compiler for such functions, will generate a special OPCode for the executor to do some job later : **ZEND_DECLARE_LAMBDA_FUNCTION**

Here is the important part of the PHP parser code :

	function is_reference { zend_do_begin_lambda_function_declaration(&$$, &$1, $2.op_type, 0 TSRMLS_CC); }
		'(' parameter_list ')' lexical_vars
		'{' inner_statement_list '}' { zend_do_end_function_declaration(&$1 TSRMLS_CC); $$ = $3; }
	|	T_STATIC function is_reference { zend_do_begin_lambda_function_declaration(&$$, &$2, $3.op_type, 1 TSRMLS_CC); }
		'(' parameter_list ')' lexical_vars
		'{' inner_statement_list '}' { zend_do_end_function_declaration(&$2 TSRMLS_CC); $$ = $4; }

When meeting an anonymous function, the parser will call for ``zend_do_begin_lambda_function_declaration()`` from the compiler. Here it is :

	void zend_do_begin_lambda_function_declaration(znode *result, znode *function_token, int return_reference, int is_static TSRMLS_DC)
	{
		znode          function_name;
		zend_op_array *current_op_array = CG(active_op_array);
		int            current_op_number = get_next_op_number(CG(active_op_array));
		zend_op       *current_op;

		function_name.op_type = IS_CONST;
		ZVAL_STRINGL(&function_name.u.constant, "{closure}", sizeof("{closure}")-1, 1); /* the function will be named '{closure}' */

		/* classical compilation steps, shared with classical named PHP functions */
		zend_do_begin_function_declaration(function_token, &function_name, 0, return_reference, NULL TSRMLS_CC);

		result->op_type = IS_TMP_VAR;
		result->u.op.var = get_temporary_variable(current_op_array);

		current_op = &current_op_array->opcodes[current_op_number];
		current_op->opcode = ZEND_DECLARE_LAMBDA_FUNCTION; /* OPCode generation */
		zend_del_literal(current_op_array, current_op->op2.constant);
		SET_UNUSED(current_op->op2);
		SET_NODE(current_op->result, result);
		if (is_static) { /* static closures case */
			CG(active_op_array)->fn_flags |= ZEND_ACC_STATIC;
		}
		CG(active_op_array)->fn_flags |= ZEND_ACC_CLOSURE;
	}

We can see that the compiler effectively calls for generic function declaration function, ``zend_do_begin_function_declaration()``, but generates an OPCode **ZEND_DECLARE_LAMBDA_FUNCTION**.
This is very different from classical named functions, where no OPCode at all is generated (if the function is not declared conditionnaly), and where the function name is registered into the global function table for it to be looked for later, when called into the executor.

> So, anonymous functions require some runtime job, whereas non-anonymous classical PHP functions don't.

That means that anonymous functions will be less performant that named ones, because named onces are fully resolved at compile time, and OPCode cache solutions such as OPCache entirely take care of them (preventing the compiler from kicking-in). For anonymous functions, a big job is also done by OPCache, but there still need to go for some runtime job. Don't be scared however, we are talking about very little performance penalty.

What's then in this executor handler ?

	ZEND_VM_HANDLER(153, ZEND_DECLARE_LAMBDA_FUNCTION, CONST, UNUSED)
	{
		/* ... */

		if (UNEXPECTED((op_array->common.fn_flags & ZEND_ACC_STATIC) || 
				(EX(prev_execute_data) &&
				 EX(prev_execute_data)->function_state.function->common.fn_flags & ZEND_ACC_STATIC))) {
			zend_create_closure(&EX_T(opline->result.var).tmp_var, (zend_function *) op_array,  EG(called_scope), NULL TSRMLS_CC);
		} else {
			zend_create_closure(&EX_T(opline->result.var).tmp_var, (zend_function *) op_array,  EG(scope), EG(This) TSRMLS_CC);
		}

		CHECK_EXCEPTION();
		ZEND_VM_NEXT_OPCODE();
	}

If you feel lost into OPCodes and VM executor, please read [the dedicated article about those](http://jpauli.github.io/2015/02/05/zend-vm-executor.html)

As we can notice, the executor's OPCode handler for **ZEND_DECLARE_LAMBDA_FUNCTION** basically calls for ``zend_create_closure()``, and here the magic happens :

	ZEND_API void zend_create_closure(zval *res, zend_function *func, zend_class_entry *scope, zval *this_ptr TSRMLS_DC)
	{
		zend_closure *closure;

		object_init_ex(res, zend_ce_closure); /* create a closure object */
		closure = (zend_closure *)zend_object_store_get_object(res TSRMLS_CC);

		closure->func = *func; /* pass its function ; here the belt is buckled */
		closure->func.common.prototype = NULL;
		closure->func.common.fn_flags |= ZEND_ACC_CLOSURE; /* flag the zend_function as beeing of type closure */

		if ((scope == NULL) && (this_ptr != NULL)) {
			scope = zend_ce_closure;
		}

		/* ... */

		closure->this_ptr = NULL;
		closure->func.common.scope = scope;
		if (scope) {
			closure->func.common.fn_flags |= ZEND_ACC_PUBLIC;
			if (this_ptr && (closure->func.common.fn_flags & ZEND_ACC_STATIC) == 0) {
				closure->this_ptr = this_ptr;
				Z_ADDREF_P(this_ptr);
			} else {
				closure->func.common.fn_flags |= ZEND_ACC_STATIC;
			}
		}
	}

``zend_create_closure()`` is the trick : it creates an object of class Closure, and fill-in the zend_closure's function pointer, the function that itself received as a parameter : this is the function that the compiler compiled : the PHP user function OPArray.

And the belt is buckled !

Have a look at the Closure class's ``get_closure()`` handler. Unlike the ``zend_std_get_closure()`` handler, this one will not look for an "__invoke()" method in the class, but will fill the function pointer back with the one into the zend_closure, created by ``zend_create_closure()``

	int zend_closure_get_closure(zval *obj, zend_class_entry **ce_ptr, zend_function **fptr_ptr, zval **zobj_ptr TSRMLS_DC)/
	{
		zend_closure *closure;

		if (Z_TYPE_P(obj) != IS_OBJECT) {
			return FAILURE;
		}

		closure = (zend_closure *)zend_object_store_get_object(obj TSRMLS_CC);
		*fptr_ptr = &closure->func; /* here we are, the function to execute is the one that got compiled before */

		if (closure->this_ptr) {
			if (zobj_ptr) {
				*zobj_ptr = closure->this_ptr;
			}
			*ce_ptr = Z_OBJCE_P(closure->this_ptr);
		} else {
			if (zobj_ptr) {
				*zobj_ptr = NULL;
			}
			*ce_ptr = closure->func.common.scope;
		}
		return SUCCESS;
	}

And we can see that it binds the scopes as well.

### Scopes, static closures and rebinding of $this

You may remember that in PHP5.3 , it was not possible to access ``$this`` into a closure. In fact, you were accessing the ``$this`` of the Closure class itself, instead of the ``$this`` of the class the anonymous function is defined in.

If you compare the source code of Zend/zend_closures.c for PHP5.3 , and nowadays' PHP 5.6 , you will notice that every reference to ``$this`` (internally called "``this_ptr``") is missing in PHP5.3.
Closures have evolved through PHP versions. Accessing the "real" ``$this`` was added in 5.4, as well as the possibility to change the scope of the anonymous function. Let's see that together in detail.

 

When the compiler creates a standard, named, PHP function ; it looks if this function is declared into a class body. If it's the case, then the function is a method, and must be given the current class as its scope. This is done into the compiler. Here :

	void zend_do_begin_function_declaration(znode *function_token, znode *function_name, int is_method, int return_reference, znode *fn_flags_znode TSRMLS_DC)
	{
		/* ... */
		op_array.scope = is_method?CG(active_class_entry):NULL;
		/* ... */

``$this`` willl then be bound later at runtime (because $this obviously doesn't exist yet at compile time) by the engine which automaticaly creates this variable when a method call is done.

> Remember that in PHP's heart, a method is a function which scope is not NULL, that's all, no other difference (the same zend_function C structure is used, anyway).

What happens for closures however, is that the current scope (the class) and ``$this`` are both bound at runtime, by the **ZEND_DECLARE_LAMBDA_FUNCTION** OPCode. Look :

	/* void zend_create_closure(zval *res, zend_function *func, zend_class_entry *scope, zval *this_ptr TSRMLS_DC); */
	
	ZEND_VM_HANDLER(153, ZEND_DECLARE_LAMBDA_FUNCTION, CONST, UNUSED)
	{
		/* ... */

		if (UNEXPECTED((op_array->common.fn_flags & ZEND_ACC_STATIC) || 
				(EX(prev_execute_data) &&
				 EX(prev_execute_data)->function_state.function->common.fn_flags & ZEND_ACC_STATIC))) {
			/* if we are creating a closure into a static call -or we declared the closure explicitely static- pass the called_scope to the closure body */
			zend_create_closure(&EX_T(opline->result.var).tmp_var, (zend_function *) op_array,  EG(called_scope), NULL TSRMLS_CC);
		} else {
			/* else, pass the object scope to the closure body */
			zend_create_closure(&EX_T(opline->result.var).tmp_var, (zend_function *) op_array,  EG(scope), EG(This) TSRMLS_CC);
		}
		
		/* ... */
	}

This is tricky. When in runtime, when it comes time to create the closure, we browse the previous executor stack frame to see if the last call was static or not. If the last call was a non-static method call, then we extract the ``$this`` pointer from this method call, and pass it to the anonymous function.

	class Foo {
		public function hello() {
			return function () { };
		}
		public static function world() {
			return function () { };
		}
	}
	
	class Bar extends Foo { }
	
	$foo = new Foo;
	
	$closure = $foo->hello(); /* into hello() $this exists, the closure will then be created and bound to this $this and to the class scope */
	$closure = Foo::world(); /* world() is a STATIC function, so there will be NO $this passed to the Closure, but only its scope : Foo */
	$closure = Bar::world(); /* world() is a STATIC function, so there will be NO $this passed to the Closure, but only its scope : Bar */

That's how its done.

Also, if you spot the parser and the compiler code carefully, you'll notice that an anonymous function can be declared static itself :

	$a = static function () { };
	/* $a holds a static closure, that will never ever receive any $this into its body */

If the closure is declared static itself, it will never receive any ``$this`` pointer, even if we would have declared it into a non static method. This is what we call a "static closure".

	class Foo {
		public function hello() {
			return static function () { }; /* let's return a static closure */
		}
	}
	
	$foo = new Foo;
	
	$closure = $foo->hello(); /* the $closure has no $this access, because it was declared static itself */

Now let's talk about the scope, that is the current class where in the Closure resides (which is used to check for PPP access).
The scope is the class's which was used for the call to create the closure, like in the example above, with Foo and Bar classes.

> The scope is the current calling class, it is used to check for public/private/protected access rights (PPP).

Remember that both scope and ``$this`` pointer are important, as they will serve when the Closure will get called. They even can be changed at runtime, starting from PHP 5.4, using ``bind()`` or ``bindTo()``

	class Foo { private $a = 42; }
	$foo = new Foo;
	
	$closure = function() { var_dump($this->a); };
	
	$foo_closure = $closure->bindTo($foo);
	
	$foo_closure(); /* 42 */

The above code displays 42.

The ``$closure`` closure was created out of any context, so its ``$this`` and its scope are both NULL. This closure is actually unbound : it has no object scope and can't access any ``$this`` variable at the moment.

Using ``bindTo()``, we create another closure from this one, but we explicitely bind its ``$this`` variable to the ``$foo`` object; thus, calling this new created closure works : it can access a ``$this`` now.
But is it allowed to read the private property ? Yes it is, because the scope is also good. ``bindTo()`` accepts a second parameter : the scope, which will default to the class of the object passed as first parameter : to us : Foo class. Into Foo, I can access Foo's privates : no problem. Let's change that behavior :

	class Foo { private $a = 42; }
	$foo = new Foo;
	
	class Bar { }
	
	$closure = function() { var_dump($this->a); };
	
	$foo_closure = $closure->bindTo($foo, 'Bar'); /* Pass the scope of Bar */
	
	$foo_closure(); /* fatal error, accessing a private member */

Now the new closure still has a ``$this`` representing ``$foo``, but the scope it will act into is the scope of Bar. Hence, in Bar, you can't access Foo's $a, as both Foo's $a is private, and Bar doesn't even extend Foo.

This scoping functionnality is presented to the PHP user for the first time here. All those mechanisms used to be hidden and automatic before, when you use classical objects and methods , but with closures, things change as closures are by definition functions or methods that can be carried from class to class at runtime : you are then dealing manually with the scopes.

> PHP7 added Closure::call(object $to[, mixed ...$parameters])

### What about Reflection and direct __invoke() calls ?

All could have been finished if Reflection wouldn't exist...

But Reflection exists, and can do many things. Thus, we need to finalize the Closure class. Because even if the user can't manipulate this class, Reflection can.
Reflection will ask the class for some methods, particularly ``__invoke()`` , and we saw together that this ``Closure::__invoke()`` simply does not exist, it is not used by PHP's closures, it is short circuited by redefining a ``get_closure()`` handler that does the job.

So we need to implement some more handlers for Reflection to be happy. ``ReflectionFunction::getClosureScopeClass()``, ``ReflectionFunction::getClosure()``, ``ReflectionFunction::isClosure()``.

Try this :

	var_dump ((new ReflectionClass('Closure'))->getMethods());

You will see ``__construct()`` and ``bind()``/``bindTo()`` , but not ``__invoke()``, because the Closure class simply has no ``__invoke()`` method.

Now try this :

	var_dump ((new ReflectionObject(function(){}))->getMethods());

Here, an ``__invoke()`` method  will magically appear. It is faked, faked by the Closure class and a Reflection trick. It is built and specially crafted just to be shown in Reflection's output :

	#define ZEND_INVOKE_FUNC_NAME       "__invoke"
	ZEND_API zend_function *zend_get_closure_invoke_method(zval *obj TSRMLS_DC)
	{
		zend_closure *closure = (zend_closure *)zend_object_store_get_object(obj TSRMLS_CC);
		zend_function *invoke = (zend_function*)emalloc(sizeof(zend_function));
		const zend_uint keep_flags = ZEND_ACC_RETURN_REFERENCE | ZEND_ACC_VARIADIC;

		invoke->common = closure->func.common;
		invoke->type = ZEND_INTERNAL_FUNCTION;
		invoke->internal_function.fn_flags =
			ZEND_ACC_PUBLIC | ZEND_ACC_CALL_VIA_HANDLER | (closure->func.common.fn_flags & keep_flags);
		invoke->internal_function.handler = ZEND_MN(Closure___invoke);
		invoke->internal_function.module = 0;
		invoke->internal_function.scope = zend_ce_closure;
		invoke->internal_function.function_name = estrndup(ZEND_INVOKE_FUNC_NAME, sizeof(ZEND_INVOKE_FUNC_NAME)-1);
		return invoke;
	}

This ``zend_get_closure_invoke_method()`` that creates a fake ``__invoke()`` for the engine, is also used when one directly calls ``__invoke()`` on a closure.

	$a = function() { };
	$a->__invoke();

But you shouldn't do that. Doing this, you instruct the engine to fetch a method, whereas if you were using the ``$a()`` syntax, we saw that the engine directly fetches the function to run using the ``get_closure()`` handler.
The problem with the explicit ``__invoke()`` call is that you turn around, and ask the engine much more work to do. It has to fetch the method, it will then run the ``get_method()`` handler on the object, which is of class Closure. Here is its code :

	#define ZEND_INVOKE_FUNC_NAME       "__invoke"
	static zend_function *zend_closure_get_method(zval **object_ptr, char *method_name, int method_len, const zend_literal *key TSRMLS_DC)
	{
		char *lc_name;
		ALLOCA_FLAG(use_heap)

		lc_name = do_alloca(method_len + 1, use_heap);
		zend_str_tolower_copy(lc_name, method_name, method_len);
		if ((method_len == sizeof(ZEND_INVOKE_FUNC_NAME)-1) &&
			memcmp(lc_name, ZEND_INVOKE_FUNC_NAME, sizeof(ZEND_INVOKE_FUNC_NAME)-1) == 0
		) {
			free_alloca(lc_name, use_heap);
			return zend_get_closure_invoke_method(*object_ptr TSRMLS_CC);
		}
		free_alloca(lc_name, use_heap);
		return std_object_handlers.get_method(object_ptr, method_name, method_len, key TSRMLS_CC);
	}

It checks if you call a method named "__invoke", you can't call anything else, as if you do, you jump into the default ``get_method()`` handler, which will tell you that the Closure class has no "foobar" method.
So here, we called ``__invoke()``, so we return ``zend_get_closure_invoke_method()`` which code we already analyzed. It creates a "fake" ``__invoke()``, but with a real handler :

	invoke->internal_function.handler = ZEND_MN(Closure___invoke);

And here is its awful code :

	ZEND_METHOD(Closure, __invoke)
	{
		zend_function *func = EG(current_execute_data)->function_state.function;
		zval ***arguments;
		zval *closure_result_ptr = NULL;

		arguments = emalloc(sizeof(zval**) * ZEND_NUM_ARGS());
		if (zend_get_parameters_array_ex(ZEND_NUM_ARGS(), arguments) == FAILURE) {
			efree(arguments);
			zend_error(E_RECOVERABLE_ERROR, "Cannot get arguments for calling closure");
			RETVAL_FALSE;
		} else if (call_user_function_ex(CG(function_table), NULL, this_ptr, &closure_result_ptr, ZEND_NUM_ARGS(), arguments, 1, NULL TSRMLS_CC) == FAILURE) {
			RETVAL_FALSE;
		} else if (closure_result_ptr) {
			zval_ptr_dtor(&return_value);
			*return_value_ptr = closure_result_ptr;
		}
		efree(arguments);

		/* destruct the function also, then - we have allocated it in get_method */
		efree((char*)func->internal_function.function_name);
		efree(func);
	}

It does a ``call_user_function()``. That is, you launch ``__invoke()``, you pass through 2 handlers ending in running a ``call_user_function()``. Remember that a ``call_user_function()`` is really bad for performances, because it basicly pushes another stack frame and another function onto the engine stack, whereas we are already calling a function. It's like using ``call_user_func()`` in PHP where using a direct call is possible : just a pure waste of resources, for the exact same result.

> Don't call __invoke() directly on your Closure object, it is better for performance to call the $closure() directly.

## Conclusions

So, this is how Closures got added to PHP : A new class internal handler has been designed : ``get_closure()``. This latter is called when a function call is done in PHP, onto a variable of type object.
``get_closure()`` is then expected to fill in a ``zend_function`` pointer that will be run by the engine (otherwise this latter doesn't know what to do).
But to support anonymous functions, a class was needed : the Closure class was added back in 2009 (PHP5.3), and many of its internal handlers have been redefined so that the PHP user can't do anything with just the Closure class as-is.
Then, the compiler was patched to generate runtime opcode when it meets an anonymous function, that will turn this latter into an object of the Closure class, which when invoked will run the parsed OPArray of such a function.

Clever tricks to add more code onto a 10 year old codebase (Zend Engine 2), without breaking everything everywhere.
The same has been done when generators have been added to PHP5.5, but this time, it is way more complicated that just a Generator class and a new class handler. Perhaps the subject of a next article ?
