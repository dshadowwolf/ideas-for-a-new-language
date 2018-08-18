One major idea I've come across that seems to be both the way it should be and a bit of a wank is the idea of a programming language where you define the type and value transitions in a "provably correct" manner, then define the program as a combination of those type and value transitions. I call this "a bit of a wank" because there are sections of programs where the actions are not because of type or value transitions - such a thing is not even applicable in "business" applications. But the idea that you define each "simple" chunk of a program as a simple block that can be joined with others to "compose" a program - that is a great idea and leads to what has been termed "Higher Order Software".

Earlier today, as I floated on the edge of sleep just after waking up I hit upon the truth that mathematical notation of functions is something that is well understood - almost everyone has been exposed to the idea of ``f(x) = x * x`` at some point in their schooling. Expanding that to something useful for programming is not a simple task, but gives a means of defining the different operations, transitions between types and values and could even be used to define branching and comparison.

In the end a modern language needs a way to define an encapsulated set of functions and data, a way to pull in other units defining things like that, means of controlling the flow through the code and some way of actually declaring what is a function and what is data. My preference for these things is geared towards functional-programming like immutable data, chained calls like Java 7 and above have been introducing and that there should be no real distinction between functions and data. The final point - functions and data being treated entirely equally - might not be achievable in its entirety within the quasi-mathematical notation behind this design but that does not mean that the idea has to be passed over.

An example of a primitive version of the language, with just definition of function and data and the basic "function call" flow control methodology. This example calculates a fixed multiple of the square of a given integer value and prints it out - the "print" function used is for example and uses a syntax similar to Java's "String.format" or the similar function in C#.

```
func f(x: integer) : integer;
func sqare(x: integer) : integer;
const mult : integer = 100;

f(x: integer) = square(x) * mult;
square(x: integer) = x * x;

print_line("square of 2 times 100 is {}", f(2));
```

This example is extremely simple and for demonstrative purposes only - it should not be taken to be anywhere close to representative of the final product. It begins with a forward definition of the functions and constant data, then follows on with the actual definition of the code and finally has the line of "driver code" that marks the entry point of the example. As far as this example goes it is cleanly understandable by a human, but for a computer there would be a requirement for there to be more than a single token of look-ahead and the language itself is not, completely, context free.

However, the faults of the example aside, this shows that functions are meant to be simple, small and do exactly one thing. This isn't exactly true - conditionals can add to this - but conditionals will actually be defined as functions themselves and will be composed of a series of calls to other functions. That is... complex code is built from small, simple blocks that are designed for doing exactly one function or operation each. In truth, the "functions are data" concept is necessary to help conditionals work correctly, as the code to run for a conditional branch needs to be passed in as something like a lambda. An idea for how this could work would be like...

```
func f(x: character) : Array(character);

f(x: character) = {
  switch(x, {
  compare_item_fallthrough( x, 'a', { "blargh"; });
  compare_item_fallthrough( x, 'b', { "foobar"; });
  default_item( x, { "default!"; });
  );
};

print_line("{}", f('a'));
```

Again, this syntax is purely for demonstrative purposes and will likely change as this idea is explored further. For demonstrative purposes the curly braces mark "long chains of function calls" and is, effectively, used to demarcate no-parameter functions when used as a parameter. This, however, is also another example that suffers from being context sensitive instead of context free. It does, additionally, demonstrate that every piece is built of functions - and if each function is "provably correct" then, if the input is also true and correct - and the functions are used correctly - this means that the end result is also "provably correct". Additionally... if all the functions of each program are also added to the systems library of available functions it would become possible for users to write complete, complex and, most importantly, ***NEW*** programs just by linking the existing functions together with whatever the programs input data would be - and have it be, provided that no errors are made in using the functions the program would be "provably correct". (and done correctly the program would be robust against "garbage input")

A better syntax, borrowing a couple ideas from the Rust programming language, including a final declaration that nearly everything in the language is an expression, is possible. In Rust itself, the square function from the examples, would be:
```rust
fn square(x: u64) -> u64 {
  x * x
}
```
There is no "return" statement in Rust, the value of the last expression is the return value of the function. By either making it such that we have the "arrow" ``->`` operator replaced by a "store" ``=`` (or perhaps ``:=``) we can differentiate a function definition from a function declaration. In addition, though it tends to make code more verbose, the declaration of a variable should be separate from the definition of its value. These changes, applied to the first given example, lead to a cleaner design with a lack of context sensitive pieces, as in:
```
func f(x: integer) -> integer;
func sqare(x: integer) -> integer;
const mult : integer;

mult = 100;

func f(x: integer) = square(x) * mult;
func square(x: integer) = x * x;

print_line("square of 2 times 100 is {}", f(2));
```
It might also be better, in fact, to declare that all blocks of code must be wrapped in curly-braces. Not, strictly, for the compiler, but also for the programmer - by having all blocks of code cleanly delineated tracking which pieces of code go together is made much easier. This would, for example, result in people not making a possible mistakes that are somewhat common to see in new programmers in the C family of languages, where a block of code in curly braces is, in some cases, treated as equivalent to a single expression or statement, such that:
```c
if( x == y ) do_z();
```
is seen as completely equal to:
```c
if( x == y ) { do_z(); }
```
and this does cause issues at times, as some programmers not used to this "feature" of the C family of languages will try adding to the code for the if-block and finding that it runs
regardless, because they did not include the braces. By declaring that the braces are needed to wrap any actual code we also wind up with a means of declaring code as a form of data. To that end, and making the syntax even more regular, we come up with the following:
```
func f(x: integer) -> integer;
func sqare(x: integer) -> integer;
var mult -> const integer;

var mult = { 100 };

func f(x: integer) = { square(x) * mult };
func square(x: integer) = { x * x };

{ print_line("square of 2 times 100 is {}", f(2)) };
```
Note that the ``mult`` variable is assigned a value as if that value is a chunk of code. Technically this is actually true - the chunk of code is a simple expression that evaluates to the constant value 100. Other than that the code is regularized and any context sensitive parts are actually given strict context markers such that the language itself still can be classified as "context free". This regularization makes the language easier for both humans and computers to properly parse correctly and should make it more difficult for people to write incorrect code that will compile but generate incorrect output.

### Older Notes and Information
~~~The things lacking from these examples and the problems they have (not context free, requiring more than one token of look-ahead, etc...) are because this is still a quite new idea for me and I have no firm idea how to actually implement it as far as the syntax and various rules go.~~~

Future Notes:
   - Lambda Calculus ties into this idea, in that the functions should be, as much as possible, state free, almost black boxes as Alonso Church defined things
    - In Lambda Calculus the following defines all boolean operations ('t' is the actual boolean true and 'f' the actual boolean false): 
```
	TRUE = λx.λy.x
    FALSE = λx.λy.y 
	IF = λb.λt.λf.b t f
	AND = λab. IF a b FALSE
    OR = λab. IF a TRUE b
    NOT = λb.IF b FALSE TRUE 
```
  - The preceding Lambda Calculus is not needed in this language, as it defines primitive operations that the language itself will be defining internally. The example is to show what a raw, purely mathematical notation for computation - even for things as simple as base boolean operations - looks like.
  - Comparison functions should return a defined value for various sorts of options - a value for "didn't match" would be needed in the example
    - the switch function in the example above, if processed in a manner compliant with the standard C-like fallthrough, would result in a set of "didn't match" and functions, which might just be a set of return statements.
  - A set of default operators for math, comparison, boolean operations and bit manipulation - this should probably include something like the famous "ternary" operator and some means of saying "compose as a function" for allowing for functions to return functions. (ie: lambda functions)
  - Definite syntax for function and variable declaration and definition
  - Definite syntax for code blocks, function returns, etc...
  - Encapsulation, structured data - ie: objects, structs, enums, etc... all need some definition
  - Program entry point, namespaces, etc...
  - Standard library
  
Concepts missing from the examples:
  1) Encapsulation/Objects
  2) Libraries/Namespaces
  3) Actual low-level definition of conditionals
  4) Built-in/predefined types - perhaps even "primitives"
  5) Proper lambdas and block structuring

Mistakes:
  1) Not single-token look-ahead
     - single token look-ahead makes parsing much easier and, in some cases, faster
	 - allows for much, much simpler state tables and a smaller number of states overall
  2) Not context-free
     - a context free language has every construct meaning the exact same thing regardless of the context of it - ie: a function call is a function call where the examples given have function definitions only differentiated from function calls by having the types defined in the parameter list
	 - context free languages are much easier for programmers to understand
	 - they are also a lot easier to locate bugs in because of the "each construct means the same thing regardless of where it is found" nature.
	