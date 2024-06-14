# psil
## Protected Stack Interpreter Language

Psil is a pet project, aiming to repair everything wrong with lisp and FORTH
by taking the two and mashing them together to create an unholy abomination.


How psil differs from lisp:
* Visually, programs appear to be written almost entirely in reverse.
* Programs can be interpreted directly, one token at a time.
* Underlying implementation details are more directly exposed.
* The language is typeless.
  * There are a variety of tools for interacting with different types of
    data, but no checking is performed, and the standard data type will
    be the native pointer size. (64 bit on most modern systems).
* Direct memory management is trivial.
  * For embedded platforms, everything is just direct pointers.
  * For OS platforms, a malloc/dealloc word set will be provided as part
    of supporting the C ABI.

How psil differs from FORTH:
* The stack is (softly) protected from overrun.
  * Bypassing this protection is easy, but requires deliberation.
* Obtaining word pointers is trivial, since the word itself simply pushes
  the pointer to the top of the stack. Invocation is performed by a 
  separate token ')'.
* Local vs global contexts are available and distinct, allowing variables
  to vanish with scope exit without memory allocation issues.
  * This is implemented by a separate variable stack with the same delimiters.
    the REPL parser has to manually check the local vs global dictionaries
    but anything that has been predefined (including anything inside of any
    word) will represent local vs global context directly as pointers.

Desirable: Word-level mailbox concurrency. Ideally, this would be accomplished
by separate stacks for each invocation, with a given outer stack waiting on
returns from inner stacks element-wise. In this fashion it is theoretically
possible to have multiple words executing concurrently in a green-thread style,
with a FIFO queue of launched tasks that cycle through when their dependencies
arrive.

## Conceptual overview of the backend

Psil operates using two separate stacks.

Call stack - Used for returning from word calls, implementing tracebacks, etc.
			 Also contains bottom-of-local stack offsets.
Work stack - Holds the global working data, and performs overrun checks against the call stack.
			 during function calls. This is the stack you directly interact
			 with the most.


## Basic syntax:

### Words vs Numbers vs Specials

A number is anything matching the interpreter's view of something "convertible to
a single value". For example:
* 12345 - 12345 in decimal
* 0b10101 - 21 in binary
* 0xffff - 65535 in hexadecimal
These values directly push the presented number to the stack.

A word is anything that, when presented to the interpreter, returns a single
value. This is almost always a pointer.

#### Specials

Special tokens have syntactical differences from others. Primarily,
they do not require separators to function, so for example "asdf" is a
valid string definition, and {word word word} is a valid code block.

The following list is a set of special-purpose tokens that are not eligible for
use in words.

( updates the stack bounds checker's position. (pushing to the bounds stack)
) pops and calls the top of the stack as a function.
" begins and ends a string literal (UTF-8 as standard) which is placed on
  the heap and the pointer to it is pushed to the stack.
  Standard C style escapement applies
' behaves the same as " but only matches itself, as seen in python
, equivalent to whitespace in most scenarios
\# starts a comment that ends at the next newline.

{ starts defining a code block 
  The primary behavior here is as follows:
  * The start of the code block on the stack is labeled.
  * Push-like operations affect that stack instead of the main stack
    * This includes words that would be executed.
  * ( and ) push execution literals instead of actually executing.
  * [ and ] force actual execution to occur like ( and ) do outside.
    This allows compile-time programmatic definitions, like FORTH supports,
    and is far more flexible than eg preprocessor directives in C.
    It also allows the definition of static variables inside a word-local context.

} finishes defining a code block, returning the invocation pointer.
  Implementation-wise, the resulting substack is moved elsewhere
  (code heap, alloc'ed space, *somewhere*)
  naming does not occur inside the code block.

Curly braces can be nested, and are always evaluated at definition time.
  
[ and ] always run immediately, whereas ( and ) only run if they are
interpreted outside of curly brace definitions.


= this stores the top value in the stack to the address referenced by
  the next-down value in the stack. Since larger data types return pointers,
  this also allows strings, arrays, code, etc to use the same assignment.
  
  ( a 123 = )

### Initialization

Words must be 'defined' before use. This assigns them a single memory location, 
whose address cannot be changed, and is implementation specific in size.
It also allocates them to the

In C terms, they are of type void * const.

The words used to initalize new words are:

"new_word_name" def

The result of this is the same as if the new word had just been used:
the new pointer is at the top of the stack. This means creating and storing
a value into a word looks like:

("word" def 123 =)

In an ideal implementation, the words are defined in a pseudo-vector,
sorted by their UTF-8 name. This makes for very fast word lookup.

The dictionary stores the name and the address of the word's memory as
two native values in size. (eg for x64, 16 bytes total)
The first value is a pointer to the name's string, which is thrown on a
heap. 
The second value is a pointer to the allocated memory.

These values are packed in sorted order in a size convenient for the system.
(eg 256 values to a 4KB block on x64). The first 2 entry pairs in the packing
are reserved for a direct-packed value indicating:
* The first n bytes of the name of the last word in the block
* The position of the last word in the block, as an offset
* The location of the next block

Words are inserted into their blocks, insertion sort style. Any leftovers
pushed out of the block, instead of inserting into the next, create a new block
with just them in it.

Calling the word refers to the location, which might contain any of:
* An integer, float, or other numerical type
* An 8 character string
* A pointer to an array of arbitrary type.
* A pointer to some more complex structure.
* A pointer to executable code.

### Recursion

A code block can recurse on itself by re-calling its own word. To do this,
the code block should be stored into a word that has already been defined

For example, this is an infinite recursion loop.
( "recursing" def { (recursing) } = )

This works because the word is a pointer to the code block, and the pointer
is allocated before the code block is compiled, so the address that is placed is correct.

This would not work if you wanted to do immediate [] recursion, since the
pointer does not yet point to the code block that is compiling.

### Lambdas/Anonymous functions

Code blocks need not be named to be executed. Consider the following statement:

( args { lambda-expression } )

So long as recursion is not required, this works just fine.

### Conditionals

There are two styles of builtin conditional: 

Ternary operators
If/then on code blocks

In C, a ternary operator checks an input and returns either of 2 following values as true or false.

In psil, ...(iftrue iffalse condition ?) does something very similar.'
If the condition is true, the stack is left with only ...iftrue
If not, the stack is left with only ...iffalse

If this is for flow control, iffalse and iftrue can each be their own code blocks, like so:

( pre-args {code that runs if true (then-like)} {code that runs if false (else-like)} condition ? )

This executes the true/false block conditionally (but never both) with the
same provided leading arguments, while preserving stack protection.

### Printing


### Example - Fibbonaci Sequence

("fib" def { (. .. +) } =) (1 1 fib)
