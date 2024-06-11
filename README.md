#psil
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

##Basic syntax:

###Words vs Numbers vs Specials

A number is anything matching the interpreter's view of something "convertible to
a single value". For example:
* 12345 - 12345 in decimal
* 0b10101 - 21 in binary
* 0xffff - 65535 in hexadecimal
These values directly push the presented number to the stack.

A word is anything that, when presented to the interpreter, returns a single
value. This is almost always a pointer.

####Specials

Special tokens have syntactical differences from others. Primarily,
they do not require separators to function, so for example "asdf" is a
valid string definition.

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
} finishes defining a code block, returning the invocation pointer.
  naming does not occur inside the code block.



= this stores the second value in the stack to the address referenced by
  the top value in the stack. Since larger data types return pointers,
  this also allows strings, arrays, code, etc to use the same assignment.
  
  123 a =
