###Basics of any programming language
1. Math operations
  * At its core a computer does exactly what the name implies - computes. So every language needs a basic set of math operations
    1. Addition
	2. Subtract
  * From just addition and subtraction all other operations can be built - this is one of the core concepts of a RISC computer. However, humans don't work this way, so a more complete list of operations is:
    1. Addition
	2. Subtraction
	3. Multiplication
	4. Division
  * From those 4 any required mathematical operation can be performed on integers. This does lead to some issues, however, when pure integer operations are not available or when operations are not being carried out on integers. This means that at least one more operation is required... The modulo operation. (Which is part of all modern encryption and a key part of the one known unbreakable encryption) More than that, however, most computers also provide a bitwise shift and a bitwise "rotate" operation - both of which get used, to some extent, in encryption and numerous other fields. (even though bitwise shifts are, basically, just multiplication and division by powers-of-two) Including the shift and rotate operations, this makes the final list of mathematical operations...
    1. Addition
	2. Subtraction
	3. Multiplication
	4. Division
	5. Modulo
	6. Shift Left
	7. Rotate Left
	8. Shift Right
	9. Rotate Right
2. Without the ability to compare numbers, all the computation in the world is useless. In its most generalized form there would be a single "conditional" keyword that allowed for all possible uses and could be specialized off in some manner. Invariably this will lead to some changes to the language away from the general "it should be like C" standard - however this is acceptable. Basic comparisons are...
  * Equality - do two given objects have a value (via some given 'value' member function, perhaps) that are equal ?
  * Inequality - without care for which is greater or lesser, do two objects have inequal values ?
  * Less-than - specialized case of inequality where the test is whether one object has a value that is less-than another
  * Greater-than - the opposite test of 'Less-than'
  * In all of the above listed tests, it should likely be relegated to an overridable function or set of functions of the generic 'Object' class what the result of any given comparison would be. This allows for the maximum flexibility and also allows for custom objects to fit into the existing structure of operators without having to directly provide for a customized implementation of such.
3. Binary combination
  * These are operations that a lot of programmers are familiar with, even if they do not think of them as more than simple syntactic sugar (I, personally, did not back when I was first learning to program)
    1. AND
	2. OR
	3. XOR
	4. NOT
  * These four operations are well defined boolean/binary operations and can actually be used, along with the shift operators, to replicate the 7 non-shift math operations. In practical use, most programmers will only use these to combine the results of various comparison operations that form the base of input to a condition operation.
4. Other operations
  * At this point there is only one other generic, always needed operation - Assignment.
  * A direct, "literally equal" comparison might be handy - this would apply in situations where the type and value are both equal or, potentially, where an object is being compared to itself. (perhaps as part of finding a given object in a linked list or similar)
  
###Proposed 'operators' mapped to operations named above
1. Addition:       +
2. Subtraction:    -
3. Multiplication: *
4. Division:       /
5. Modulo:         %
6. Shift Left:     <<
7. Rotate Left:    <<<
8. Shift Right:    >>
9. Rotate Right:   >>>
10. Equality:      ==
11. Inequality:    !=
12. Less-than:     <
13. Greater-than:  >
14. Assignment:    :=
15. AND:           &&
16. OR:            ||
17. XOR:           ~
18: NOT:           !
* 18 operators, total. One of them a partial combination of two others.
* These follow the C/C++ standard for operators in everything except for assignment, where the Pascal standard is used. In the case of assignment the Pascal standard is used as a means to help catch a common error seen in numerous C and C++ programs.
* I'd actually prefer it if classic math operators could be used for all of the given operations that have direct mappings, but this would make writing programs in this language difficult for people using classic keyboards.

###Control Structures
At its most basic, every program is a series of instructions. This is why the oldest class of languages are known as "procedural" - they are sets of instructions grouped into procedures and run in order, as needed. Even the newest functional languages operate this way, although they often contain features to delay the execution of any given set of instructions until absolutely needed and change the most basic construct from the instruction to the function. The choice of which instructions to execute and how many times is controlled by a languages set of "control structures", and as this fits well with most people thought processes we will not be changing that.

* Conditional Execution
  * The following are proposed ideas
    1. 'cond' operator - similar to Lisp.
      * takes a group/list of pairs of structures as input, uses one part of the pairs to decide if the other part of the pair should be executed. Basically... give it a comparison and a block, if the comparison is true, the block is executed. Note that all comparisons are done and all blocks where the comparison is true will run.
	  * If a method to early-end the testing is provided this could be used as part of a basis to provide the more well known if-then-else setup, similar to how Common Lisp has done it.
	2. 'if/then/else' - similar to most existing languages
	  * Works like the 'if statement' in C/C++/Java. Only does the tests until it finds the first one that returns a truth value, then executes the matching block of code.
  * Alternation
    * Basically... the 'switch - case' setup. 
	  * _Differentiation_: provided that comparisons are generated based on results of a function built into 'Object', it would be possible to have a 'switch - case' that is not restricted to integers and/or primitive types, but cover the entire breadth of the type system.
* Loops
  * Most languages provide several different forms of looping constructs with different semantics. For instance, in C a do-while loop always executes at least once, but a while-loop does not. What follows is a proposal for a generalized loop construct that could be used to construct all other forms of loops - and also a proposal for a classical "loop zoo".
    1. Generalized 'loop' keyword:
	  * Given a test that returns a boolean value and a block of code, execute the code until a boolean truth is returned.
	  * This provides a generalized basis from which others can construct any required type of looping construct.
	  * Producing most forms of the looping construct would either require some strange (to most people) language semantics or the macro-processor.
	2. Loop Zoo:
	  * Provide the following set of loop constructs:
	    1. Generalize 'for'
		2. 'foreach'
		3. 'for ... in' specialized version of the Generalized for-loop
		4. while
		5. do...while  (with C-like semantics)
* Functions
  * Any modern language is goung to be built around functions/subroutines to allow for maximum code re-use and to better match how modern processors are built.
  * In general a function is a name associated with a block of code and, possibly, a few variables that reference the current stack-frame. (ie: its parameters). Because of some features that are proposed (well, pretty much required, actually) it is easily possible to make all functions a first-class member of the language and provide for them to be manipulated as if they were no different from any variable.
  * In specific, a function is an entry in an objects member-table that also responds properly to the 'call' function of that object.
  
* Strange thoughts/ideations on program structure:
  * In thinking about possible methods of implementing loops and conditionals it struck me that allowing each separate "block" of code to be treated as if it were a standalone block would not be a bad idea, just problematic. By doing this it would allow the system to potentially schedule any separate blocks execution to a different core, thread, process or co-routine as well as providing one more level of genericity to the language. This would also allow passing functions entire blocks of code as parameters and provide some basis for creating a generic "conditional" and "loop" operator that could then be easily specialized out.
  * This idea would also provide, in some manner, for an "eval" function. Though it may not be the best argument in its favor, the fact that it would be treating a block of code as if it was just another piece of data is meaningful. And besides providing some small basis for an 'eval' function, it would also provide a firm basis for lambda functions and things similar to Pythons list-comprehensions.
  * In treating blocks as just chunks of data, it would make runtime binding of functions to names - possibly through use of the reflection system - a lot easier. It would also generalize some things and show that a function is just a variable that has had a block of code associated with the name. This would also allow things like the return value of a function being stored in the same place that a variables data might have been...

###Closing notes
This document is as complete as I can make it at this point and should serve as a starting point for defining the rest of the language. While I have grown partial to the idea of treating each block of code as nothing more than another piece of data, this does not mean it is a good idea or the best way forward.