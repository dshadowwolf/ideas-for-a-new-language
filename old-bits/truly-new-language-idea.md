# More Thoughts on Language Design

## Prologue
For the past fourty years or so it has been habit to design a language around a single paradigm of writing the code - from semi-structured procedural languages like Fortran, to structured, weakly typed languages like C through to strongly typed, object-oriented languages like Java and even encompassing functional languages like Haskell.

This does make the design of the language a lot easier and help with controlling the errors that can exist, to the extent that some of them require all variables to be immutable after the first assignment. However this also traps people into a single paradigm of computing and can cause major issues that lead to a proliferation of patterns designed to help deal with the complexities involved in implementing some of the existing algorithms in those languages.

A better method would be to follow an example like Lisp. While nominally used and listed as a "functional" language, Lisp actually encodes a complete concept of computation, which shows in that you can write Procedural code, Functional code and it even has an object-oriented system available that can be used.

A system like that - one built up from a simple concept and syntax - is certainly not going to have the controls on potential sources of errors, but it is also not going to force any particular paradigm, leaving the programmer free to choose which paradigm to use to encode the various parts of their code.

## Basic Concept
The idea, at its root, is to build the language in a series of layers of increasing complexity and "known correctness" until there exists a layer for all levels that someone might choose to work at while not enforcing any single paradigm but instead giving users tools to encode their problems in whatever paradigm they choose.

That, in itself, makes the language a very simple and very complex beast at the same time. At its core the language would be a set of core routines written in the lowest-level language - one that is easy to translate into the final, target representation and possibly even one that it is also possible to do such with using nothing more than tools either shipped with the language source or readily available on the target system.

In Lisp the layering is quite simple - you have the core operators and the "special forms" making up the lowest layer and the layer that the programmers generally see sitting on top of that, giving a three tier design. As Lisp is a data-driven language where all code is data and data is code this three-tier layering is quite simple and nice, but adding a new paradigm means working hard to make the paradigm fit the language without looking out of place and might also mean having to implement new special-forms in the base language. That situation is non-ideal for the purposes of the idea being discussed here.

In the language concept being discussed here there are a set of layers, that, broken out with each set of functionality or interfacing as its own layer, would easily reach out to 16 or more layers. A more compact form of that is what this proposal will cover.

## More In Depth
Let the core of the language be defined in a manner similar to Lisp - there are a few core operations that are always defined and present but the rest is left undefined and up to later layers. In this case, the lowest layer on the stack - layer 1 - would come out looking like an assembly language, but instead of memory locations and registers, it would act on defined symbols and the operations and data assigned to them.

The next layer (Layer 2) would define a core library of operations surrouding the languages core defined types, which should include, but not be limited to, the machine primitives, the pointer (both function and data on machines where they are different things), the boolean, a structured data type, the symbol and the string. The symbol is inherited from layer 1 and it is important to preserve it as a type all its own at this and later layers as it can be used for multiple purposes.

With Layer 3 comes the first (and only completely separate, at this time) "complexity reduction" layer, where the complexity introduced in layer 4 is reduced and any interaction between the different paradigms is handled and any paradigm restrictions are checked across all the paradigms as encoded in the metadata associated with the various pieces of code. This is to make it so that the procedural code can't use a side-effect when called by functional code that expects variables to be immutable after assignment and so on.

At the fourth layer the various programming paradigms are defined and code regarding them can be written using layers 1 and 2 through adding defined metadata about their requirements and expectations to the various symbols metadata and also because this layer is provided access to the parser for defining new structuring and keywords as well as to the AST so that any semantic, syntactic or logical checks that are needed can be done.

And the final "programmers don't really have to work at this level" layer is layer 5 - where features of a paradigm (such as iterators, functors and similar) are defined and implemented. This is also the layer where experimental/proposed extensions to the language can be implemented for testing or even for providing them to older versions as possible.

The layer that programmers will truly interact with and change is layer 6 - which encompasses the "System Standard Library" and "User" layers of the language. It is in this level that the entirety of the system library - in a form ready for users to beat on - exists and it is also here where programs will be placed when the compiler and linker are working to build the final object code or executable.

This gives the following layers:
1) Core
2) Base Library
3) Interaction/Complexity Reduction
4) Paradigm Definition
5) Paradigm Features
6) System Library and User Code

### Core Layer
In a manner similar to Lisp this layer is a set of primitives needed for the language to not only be Turing-complete, but capable of handling any computation. In fact, consider this layer as something akin to Perl 6's PIL or LLVM's Intermediate Code.

Operators:
1) ASSIGN
2) ADD
3) SUBTRACT
4) MULTIPLY
5) DIVIDE
6) MODULO
7) COMPARE LESS THAN
8) COMPARE GREATER THAN
9) COMPARE EQUAL TO
10) AND
11) OR
12) XOR
13) NOT
14) SHIFT LEFT
15) SHIFT RIGHT
16) ROLL LEFT
17) ROLL RIGHT
18) DEFINE (for creating symbols)
19) UNDEFINE (delete a symbol - for scope-limited symbols)
20) LOAD
21) STORE
22) GET (symbol metadata)
23) SET (symbol metadata)
24) CALL (symbol metadata encodes call type)
25) MPUSH (stack interaction - machine stack)
26) MPOP (stack interaction - machine stack)
27) SETREG (machine register interaction)
28) GETREG (machine register interaction)
29) BR (branch, unconditional)
30) BRTRUE (branch, based on last compare being true)
31) BRFALSE (branch, based on last compare being false)
32) RETURN
33) PUSH (stack interaction - language/ABI stack)
34) POP (stack interaction - language/ABI stack)

At this layer the code starts at the first line and is meant to be interpreted through to the last line, with execution ending when the end of the file is encountered. Symbols belonging to known, defined system functions are pre-defined by the environment before this layers code is executed so this should start with definitions of any FFI symbols, followed by any compilation-unit wide symbols - top level functions/objects, paradigm defined features, constants and globals.

Following all those declarations should be the code itself, with the point of each function being defined with a `SET <symbol>, $#START#$, $#HERE#$` (or whatever the final syntax winds up as) and any early-out from the program easily done as a `BR $#END#$`

Variables and things local to a symbol (things such as member-functions or fields in an OO paradigm or structured data) can be assigned to the synmbols and variables local to a function can be declared with a DEFINE at the start of the function and removed with an UNDEFINE at the end

In some ways this is a step above most intermediate-code representations and a step below almost any assembly-code and this is by design. Only those implementing the language on a new platform or extending things on a layer below `Paradigm Definition` really need to worry about things on this level.

Basically, this is an intermediate-code representation of the language in a form that should be relatively easy to run a an assembler across to produce code for the target machine - or, in some ways, it could even be turned into a bytecode for a VM.

### Base Library
Here we have a set of DEFINE'd functions that implement the raw helpers needed for any true functionality at the higher levels. This includes base memory handling/garbage collection, pointer/array manipulation and much more.

Note that it is here that any helpers that are fully cross-paradigm should be defined.

**__//THE REST OF THIS SECTION INTENTIONALLY BLANK - CONTRIBUTE IDEAS FOR THIS CORE LIBRARY!//__**

### Interaction/Complexity Reduction
Does the Object Oriented ABI use symbol metadata for tracking member variables and functions but the Functional Paradigm has everything DEFINE'd individually? This is the layer where those differences are resolved.

**__//THIS REST OF THIS INTENTIONALLY BLANK - CONTRIBUTE IDEAS FOR THE COMPLEXITY REDUCTION FEATURES//__**

### Paradigm Defintion
This section is, basically, a domain specific language for specifying how the tokens returned from the parser are managed and how the AST that the parser generates is transformed into the paradigms version of the base-code mixed with the core-library.

In here 
**__//THIS REST OF THIS INTENTIONALLY BLANK - CONTRIBUTE IDEAS FOR THE COMPLEXITY REDUCTION FEATURES//__**

### Paradigm Features
**__//THIS SECTION INTENTIONALLY BLANK RIGHT NOW//__**

### System Library and User Code
**__//THIS SECTION INTENTIONALLY BLANK RIGHT NOW//__**

