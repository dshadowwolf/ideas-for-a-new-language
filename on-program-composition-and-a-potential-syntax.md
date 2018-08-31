### Original Thoughts
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
var mult -> integer;

var mult = { 100 };

func f(x: integer) = { square(x) * mult };
func square(x: integer) = { x * x };

{ print_line("square of 2 times 100 is {}", f(2)) };
```
Note that the ``mult`` variable is assigned a value as if that value is a chunk of code. Technically this is actually true - the chunk of code is a simple expression that evaluates to the constant value 100. Other than that the code is regularized and any context sensitive parts are actually given strict context markers such that the language itself still can be classified as "context free". This regularization makes the language easier for both humans and computers to properly parse correctly and should make it more difficult for people to write incorrect code that will compile but generate incorrect output.

### New Thoughts
In trying to work out a means of defining "structured data" - think the "struct" of C or the classes of object oriented languages - and seeking some help from a mathematician it has come to my awareness that I left quite a bit of context dependency in the language and was still carrying over an artificial separation between functions and data that should not exist in a language in which functions and data should be interchangable.

After much thought on this - while working on the issue of how to present the structured data to the compiler and also allow for it to take functions as members - what has come out is not a means of defining structured data, though I am working on ideas for this, but actually a system in which lambda functions are realized and the artificial separation between functions and data is removed.

While this new language definition removes some of the regularity of the old one - most notably making it so that curly braces are once more only used to denote blocks of code - it also, to me (at the least), makes the language more expressive. That this also removes some features that might have led this to becoming a [Bondage and Discipline Language](http://www.catb.org/jargon/html/B/bondage-and-discipline-language.html) is another relatively major benefit.

An example of this redesign, still lacking any thoughts on the way of defining the programs entry-point or structured data:
```
define square: mutable int = (x: integer) -> { x * x };
define f: mutable int = (x: integer) -> { square(x) * mult };
define mult: int = 100;

/* for illustrative purposes, definition of program entry point has not been decided on */
print_line("square of 2 times 100 is {}", f(2));
```

As can be seen, this uses a single keyword - ``define`` - to tell the compiler to reserve the following token as a "symbol" and has a unified syntax for defining data and code, with all functions now being lambda functions that are directly associated with a "defined" name. It also introduces the concept of the ``mutable`` keyword, which is needed as otherwise every name is defined to be associated with a constant value. The use of ``mutable`` as part of defining a function is done as it can be assumed that a compiler, having been told that anything lacking the keyword is constant, could take the result of the function being called the first time as the only result it will ever produce as part of an optimization and generate code that elides later calls, with possibly different parameters, because it can replace it with the (assumed) fixed value of the original call.

And it might, still, be possible to use the structured data definition that I had been working on prior to being notified of what I had done and realizing I had to alter the language syntax and grammar accordingly. This would require the parser to do some tracking, but would make a Lambda Functions definition a combination of the definition of a data structure and its association, via the arrow operator (``->``) with a block of code. Practically this could make the arrow operator actually translate as "applied to"...

By that definition, and defining a "function call" as a "postfix expression" where the operator is a "structured data initializer" we could, potentially, have structured data defined as:
```
define point: structure = ( x: double, y: double, z: double );
define location: point = ( 10.00, -25.42, 15.33 );
```
And, as the language makes no distinction between code and data, we can have the start of an object system:
```
define point: structure = ( x: double, y: double, z: double,
                            addOther: mutable point = (other: point) -> {
							( x + other.x, y + other.y, z + other.z )
							} );
define location: point = ( 10.00, -25.42, 15.33 );
define newLocation: point = location.addOther( (-5.25, 25, -16) );
```
All that would need to be added is some way of denoting protection to the encapsulated data and perhaps a means of denoting a function as a type...

With the full redesign that has occurred, the simple fact is that there is no "assignment" happening - a "defined and typed name" (``define x: double;``) is actually "associated" with a value - or, alternatively, a "value" is associated with the "defined and typed name". This might seem like a bit of nit-picking pedantry, but in terms of thought processes it is something that programmers should have in mind, so rather than using an "assignment operator" (``=`` in C derived languages, ``:=`` in the original operators proposal for this language and in Pascal descended languages) this difference should be made clear with an "association operator", just as parameter list definitions are "applied as extra context to the scope of a function body" by the "arrow operator" (``->``). I was at my wits end when Abastro on Discord suggested using ``<->``. This double-arrow immediately appealed to me, as it has the duality of the actual "association" between the name and the value, and it isn't used in any current, modern language that I know of.

Taking this into account and applying it to the previous examples of the proposed syntax, we end up with the following for the "raw" example:
```
define square: mutable int <-> (x: integer) -> { x * x };
define f: mutable int <-> (x: integer) -> { square(x) * mult };
define mult: int <-> 100;

/* for illustrative purposes, definition of program entry point has not been decided on */
print_line("square of 2 times 100 is {}", f(2));
```
Yes, the "association" can also associate a description of the data as well as data or a function, so we wind up with the following for the "structured data" and "structured data with function" examples:
```
define point: structure <-> ( x: double, y: double, z: double );
define location: point <-> ( 10.00, -25.42, 15.33 );
```
And also:
```
define point: structure <-> ( x: double, y: double, z: double,
                            addOther: mutable point <-> (other: point) -> {
							( x + other.x, y + other.y, z + other.z )
							} );
define location: point <-> ( 10.00, -25.42, 15.33 );
define newLocation: point <-> location.addOther( (-5.25, 25, -16) );
```
### Language Concepts and the humble Function Call
It seems that the current structure of the function call action - something that was one of the key, defining pieces that led me down the road to the current form of Turned C - is now in need of a replacement as it does not fit with the concept of the language as the language is defined. When a function is called, it is not "passed arguments" - they are bound to typed names that are part of the functions "execution context" or "scope". That means that the current form of the function call in Turned C presents the wrong picture of what is happening to the programmer.

Then there is the fact that we have, effectively, a need for two different forms of function call - one that needs to be evaluated immediately, without needing an evaluation of the name storing its response and one that can have its execution deferred until the value is actually referenced. From this we can see the need for two new operators, both related to the "associate" operator but what is not obvious is that we also need an operator for a "nominal association" since the current "positional" setup for defining values for structured data is highly positional and does not fit within the growing "there is nothing implicit, everything is explicitly shown" nature of the language.

Using the "fat double arrow" (``<=>``) for immediate execution, the "blunted fat double arrow" (``[=]``) for deferred execution and the "left pointing arrow" (``<-``) to apply a "nominal association" we wind up with the standard examples as:
```
define square: mutable int <-> (i: integer) -> { i * i };
define f: mutable int <-> (x: integer) -> { (i <- x) <=> square * mult };
define mult: int <-> 100;

/* for illustrative purposes, definition of program entry point has not been decided on */
(format <- "square of 2 times 100 is {}", args <- [(x <- 2) <=> f]) <=> print_line;
```
and:
```
define point: structure <-> ( x: double, y: double, z: double);
define location: point <-> ( x <- 10.00, y <- 25.42, z <- 15.33 );
```
I have not used the "Object" example here as there are some changes in that area, detailed in the next section.
   
### On Object Oriented Programming and "Turned C"
The current form of "Turned C" is a functional language, if not for the fact that it makes no differentiation between code and data then for the other features not commonly found outside of functional languages. Let us consider the actual declaration and definition of a function:
```
define square: mutable int <-> (x: int) -> { x * x }
```
This consists of 4 parts and, outside of the effective code of the function itself, 2 operators. The breakdown of this code is:
```
define square: mutable int <-> (x: int) -> { x * x }
      ^           ^         ^     ^     ^      ^
	  |           |         |     |     |      |
	  |           |         |     |     |      +------ function body closure
	  |           |         |     |     |
	  |           |         |     |     +---- "inject" operator
	  |           |         |     +---- "parameter list" - a "structured data definition" that
	  |           |         |           is, via the "inject" operator, added to the execution context
	  |           |         |           of the "function body closure"
	  |           |         |	  
	  |           |         +---- "associate operator"
	  |           |
	  |           +---- type and type modifier declaration
	  |
	  +---- name definition
```
As can be seen, it begins with the ``define`` keyword followed by the name being defined. (Using this alone is not legal in the language as it is currently defined, but as the language is not completely defined, this may change.) Following this is a colon - this is not an operator, per-se, but should be treated as one - and a set of keywords and, optionally, a type-name. This constitutes a "complete type definition" and the end of the data the compiler needs to reserve the name properly in the symbol table.
Following the opening part is the "associate operator", which creates an association between a given "typed name" and a value. That is followed by a "structured data initializer" and then the "inject" operator, which marks this as a lambda function definition. The "structured data initializer" gives a set of "typed names" - they will have values attached at the time of the function call - that are "injected" into the execution context of the last item in our example function. Said "final item" is a code-block or "closure" (the "function body closure", in this case) that contains the code executed at function call time.

I hope the preceding example shows exactly why "Turned C" is a Functional language, rather than Imperative, Procedural or Object Oriented at this point in its design. In a number of the other documents I have written about my thoughts on programming languages I have talked about making the language multi-paradigm. In truth the current direction of Turned C is towards a mix of procedural and functional paradigms, though, as the name of this section hints, work is being done to define something approaching a good object oriented - or even just plain object - system for the language.

On that front I noted, earlier, that Structure Data could include functions as they are just another form of data and not treated any different (linguistically if not on the machine level) than a string or number would be. Extending this out to a full object oriented system in the manner that most expect - with data hiding, overloaded functions, singleton/static members, etc... - would require changes to a number of core concepts of the language in such a way that it could introduce confusion at either the compiler implementation level or the user level. So what must be done is a careful evaluation of the various features that do not, already, fit with the language and whether they are needed for the objects defined in Turned C to be as effective and powerful as objects in other languages.

This leads to the conclusion that we do not add the features that require deep language changes, but instead add a few ideas to the way the closure scope/function execution concept are handled. One of these is the concept of a "this" - which is, actually, already implicitely present though not fully defined for its use in this case and the change is to actually add an explicit 'this' type pointer to the scope/context to help with clarity in some places where it might not, otherwise, exist. Another is the addition of a set of keywords to define an items "protection domain" - this would allow for data hiding - and a signifying name for the constructor and destructor of an object.

So, as a suggestion for the syntax:
```
define point: structure <-> ( x: private constructable double, y: private constructable double, z: private constructable double,
                            addOther: mutable public point <-> (other: point) -> {
							( x <- this.x + () <=> other.getX, y <- this.y + () <=> other.getY, z <- this.z + () <=> other.getZ )
							},
							getX: public int <-> () -> { this.x }, getY: public int <-> () -> { this.y }, getZ: public int <-> () -> { this.z });
define location: point <-> ( x <- 10.00, y <- -25.42, z <- 15.33 );
define newLocation: point <-> (x <- -5.25, y <- 25, z <- -16) [=] location.addOther;
```
Note that we do not allow, specifically, for a constructor in this but instead mark the members that are to be set as part of "construction" as being such, even if they are otherwise private. This saves on boilerplate code and programmer time while not breaking the concepts of the language. Also note that in the above example I have used the "lazy evaluating" function call syntax, which means that, without the addition of something that actually references the value of ``newLocation``, the function call specified will not take place, though the execution context is immediately bound.

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
	