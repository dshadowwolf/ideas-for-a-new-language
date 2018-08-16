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
  compare_item_fallthrough('a', { "blargh"; });
  compare_item_fallthrough('b', { "foobar"; });
  );
};

print_line("{}", f('a'));
```

Again, this syntax is purely for demonstrative purposes and will likely change as this idea is explored further. For demonstrative purposes the curly braces mark "long chains of function calls" and is, effectively, used to demarcate no-parameter functions when used as a parameter. This, however, is also another example that suffers from being context sensitive instead of context free. It does, additionally, demonstrate that every piece is built of functions - and if each function is "provably correct" then, if the input is also true and correct - and the functions are used correctly - this means that the end result is also "provably correct". Additionally... if all the functions of each program are also added to the systems library of available functions it would become possible for users to write complete, complex and, most importantly, ***NEW*** programs just by linking the existing functions together with whatever the programs input data would be - and have it be, provided that no errors are made in using the functions the program would be "provably correct". (and done correctly the program would be robust against "garbage input")

The things lacking from these examples and the problems they have (not context free, requiring more than one token of look-ahead, etc...) are because this is still a quite new idea for me and I have no firm idea how to actually implement it as far as the syntax and various rules go.

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
	