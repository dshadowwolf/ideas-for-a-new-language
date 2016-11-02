# Random Thoughts

## On Types
Many problems arise in my thoughts because I really like languages with type inference or a system such as duck-typing, where as long as the type implements the interface, it can be used in a given place. The problems arise because a proper object-oriented language should have stronger typing than that as part of combatting some of the more common sources of errors. To that end, unless the type result is explicitely clear, a type name must be given when declaring a variable. Another possible exception would be to declare the result to have a type specified that is an incomplete type - in this case, an interface that whatever object is to be stored in the variable must implement.

This is not as big a hurdle for the programmer as it might sound - with the memory and computing power available 10 years ago, a clear idea of the type of value being stored in a variable would have been easy for a computer to infer. With todays technology, it would be even faster and easier. All that means is that the compiler will not know what amount of memory to allocate for a variable until its first use. As for the other part of it - using an interface as a type specification - Java does something very similar already, so this is not a new idea.

In the end it seems that most of the types specified in [Types and Collections](ideas-for-a-new-language/types-and-collections.md) would be better specified as both an interface specification and a type - or even that all types are just interface specifications.

## Thoughts on Syntax for Inherent Multi-processing
In its most basic form multi-processing is just a way of splitting up a job into a number of smaller pieces that the computer can then switch between rapidly, doing them all at once and completing all the tasks that much faster, with only a penalty for having to switch between them. On a modern computer you will find, usually, at least two wholly independent processing cores and some of them allow for interleaving faster instructions with slower ones - what Intel calls Hyperthreading - to double the number of cores present.

In all current programming languages the task of deciding how the various tasks get split up is left to the programmer, as is the task of co-ordinating between those tasks and making sure that they don't improperly interfere with each other. The number of different issues and specific names for bugs that arise from the languages forcing that difficult task on the programmer show just how hard it can really be. To this end, I've been thinking about how a language could be designed so that the compiler actually building the executable could decide the division of tasks by itself, putting the difficult task and the tracking of all the potential issues on a machine that is, almost literally, built for doing things like that.

No truly viable solution has presented itself to me, though I have had several ideas that could serve - if the issues that implementing them could be ameliorated or removed from the equation entirely. In [Parsing and Notes on the Design](ideas-for-a-new-language/parsing-and-notes-on-the-design.md) I mention having each separate block-level as a possible separate unit of execution. While a fun idea to play with, it would cause a very large number of problems, as programmers would then have to start putting in guards in the code to make sure different blocks had actually finished executing and various other things.

Not that a modification of that idea is not work thinking about. In [Go](http://golang.org) the language has, as a base component, the co-routine (named, perversely, 'go-routines', in the documentation) and Apple has added the ability to flag blocks of code in the languages supported by their fork of the Clang compiler for separate execution, bringing threading down from the function level. This shows that the idea of having different blocks of code able to be executed in a separate thread from their parent function is an older idea and that others also saw some of the issues and pushed them away by once more putting the onus of deciding what can and cannot run independently on the programmer.

Now... it might be that, without some form of Artificial Intelligence, such a decision must always rest on the programmer. As such, I think the language might do well with a keyword or some other form of marking such that the compiler has a hint that "this block can be run in a separate thread". Though I'd prefer to see some sort of heuristics so the compiler doesn't just blindly use that to create a thread for them all, but actually tries to meet some heuristic to decide if it actually needs to or not for performance, or memory use or whatever.

##Thoughts on Language Grammar
###Defined Token Types
* Operator   (see [Operators, Flow Control and Other Things](ideas-for-a-new-language/operators-flow-control-and-other-things.md))
* Identifier (variable names, function names, etc... Almost all languages have these)
* Keyword    (well, perhaps not - as what items would be keywords in other languages could be represented differently, but, for now...)
* Constant   (any item that is not part of the macro-language or core language and is meant to be included directly in the output in some form)
* Sugar      (any token which exists for a specific purpose, but does not fit the other categories - this does include things that are normally considered "syntactic sugar" - it also covers the comma used to separate items in a list, the end-of-line marker, whatever is chosen to mark separate blocks if paired keywords are not chosen, etc...)

That's right, five types of token that cover the entire spread of things the tokenizer will be sending to the parser. Yes, there are something like 20 types of operators just for Math, Comparison, Assignment and Binary combination/manipulation - but they all fit into one category of item. The final category - that of "Sugar" will probably see the least use in any program, as its contents are, generally, not meant to be used every line. (yes, the EOL marker is, but...)

###Structure
As noted elsewhere, the language should be structured so that every piece of it is actually an expression. Every function call, loop, conditional... it all returns a value of some sort. In most cases this value will be ignored completely and possibly even elided from the binary as a general step that happens even if no optimization is requested. However, this does allow for some structuring that people would find odd.

What follows is a few examples, in a style reminiscent of C that I will define later as a proposal for the actual syntax and grammar of the language.

  ```
  import stdout from System as Out;
  import constants.ExecutionSuccess from OS as SUCCESS;
  
  Object example_hello_world {
    function main() :: Integer
	{
		Out.writeln( "Hello World!" );
		return SUCCESS;
	}
  }
  ```
Yes, the above example is the most basic example any language could have.
```
  Object example_of_expressions {
	function main() :: Integer
	  {
	     Integer A;
		 
		 A := for( var i = 0; i < 10; i++ ) { /* do nothing */ };
		 return A;
	  }
  }
```
In the above example is a demonstration of exactly what "everything is an expression" actually means. The result of the 'for' loops run is put into a variable.

####Notes
What the actual result of a loop - or any other structure that does not return a value in classical languages - actually is has not been decided and is still an open question, so take the assignment of it to an Integer with a rather large grain of salt.

###Proposed Grammar and Syntax
This proposal is both simple and complex. From the above examples the general format of the proposed language can be seen - it is vaguely Java or C#-like with ideas for the format of function definition lifted from other languages. Of course... the "Object" keyword there could be exchanged for "class", but the definition of the function is done in that manner because it will allow for functions to return multiple values.

I have had other thoughts about function definitions, however, because of how complex they can get in some other languages. One of those thoughts is to have a pre-amble block that defines the inputs and outputs of a function, in depth - allowing for almost full use of the language in defining a functions parameters and return values. This was not included in the above example because I have not been able to figure out a way to do this in an easily understandable way.

The exact syntax of the for-loop itself is something that needs to be decided, as it is, in C, a hold-over from the predecessor languages in its format and doesn't fully conform to the same formatting as the rest of the language. Truthfully, the classic, iterative for-loop is just one form of the construct, and any modern language should probably have several different forms - at least the "for each" in addition to the classic form, anyway.

There are other looping constructs that it might be best to borrow from functional programming - such as the "map" and "reduce" functions. Both can be highly optimized and generate blisteringly fast code - and both are actually quite useful in some data manipulation algorithms. More than that, some form of list-comprehension along with lambda functions would be a major positive to include, as both can lift some heavy burdens from the programmer and make some complex tasks easier.

To that end, I would like to propose the following list of keywords/built-in root-namespace level functions:
1. for
  * classic, iterative for-loop
  * could use the classic C-style format or find a different method of specifying things
  * returned value is not yet fixed - top idea is to collect the result of the final expression of each iteration into an array while another is to return the terminal value held in the variable used for the iteration - the former is preferred, as the final form of this is not yet fixed and if it follows the C convention, there could be multiple variables being allocated and modified as part of the iteration
2. foreach
  * for iterating over arrays and other iterable containers
  * simple semantics, give it a variable to bind the current item to and the iterable to loop through
  * return value for this is not yet fixed - although having this work exactly like the baseline 'for' would be nicely orthogonal, it might be better to return the number of items iterated instead
3. while
  * while loops can be optimized quite a bit - in fact, in some functional languages the type of code a C compiler might generate for a while loop is actually what is generated for a call to map() across a function.
  * only executes if and when the expression it evaluates prior to each run is true
  * return value not yet fixed - no real ideas on what it could be, although it might be good to have this work like the top proposal for the 'for' loop. (ie: collect the value of the last statement of the block per iteration)
4. do...while
  * a variant of the while loop that will always run at least once, whether or not the condition expression is true.
  * this is a trick used in quite a bit of C code in systems-level and bare-metal programming
  * return value not yet fixed - as this is actually a variant of the while-loop, it should have the same return value semantics
5. if...else
  * as in C and related languages we elide the 'then' part of the construct.
  * The actual setup should be "if <condition> <code-block> else <code-block>" - where a code-block can be another "if", if needed.
  * return value not yet fixed - once more, there are no ideas yet as to what to have this return
6. switch-case
  * fast alternation through available options with, possibly, the same fall-through semantics as C. Even though said semantics can be a source of errors, it makes for a method of writing compact code without a lot of duplication.
  * Perl lacks (or lacked) this construct - it shows in the if/elsif trees and the fact that CPAN has a module to add it to the language
  * return value not yet fixed - the only real idea for this is to return the result of the last expression prior to encountering a 'break' or the end of the structure.
  * 'case' and 'default' - if these are decided as something to include (rather than, say, doing things similar to the bourne-shell, where the 'case' looks like: '1|2|3)' and 'default' looks like: '*)') they are two things that are not, technically, expressions are are not usable in an expression.
7. break
  * end the execution of the current block here, unless we are at the top-level of a function.
  * semantic nit: should have no effect inside an if-else and when encountered there, even if in a code-block, should act upon the next highest level that is not an if-else. This is to allow it to end a loop that was started as an "infinite loop" effectively, though there is no good reason to not have an end-condition on a loop.
  * Does not return a value - this is one of the few instances (along with a couple other pieces of syntactic sugar) where something is, technically, not an expression.
8. const
  * syntactic sugar - defines a variable as having a constant value
  * as with 'case', 'default' and 'break' is unusable as part of most expressions - it can, however, be used as part of an expression where a type-cast is in use.
9. var
  * basically what modern C++ calls 'auto' - basically matches most of what it is used for in 'swift'
  * is used to tell the compiler that the type of data being assigned should be inferred from the variables first use where such an inference can be safely made.
  * another piece of syntactic sugar - as with 'case', 'default' and 'break' (and unlike 'const') this cannot be used anywhere but at a variables declaration.
10. function
  * tells the compiler that what follows is a function definition
  * once more, just syntactic sugar and is only used when defining a function
11. separable (threadable?)
  * name/terminology is debatable
  * used to flag a block of code as being capable of executing in parallel with the rest of the process.
  * return value is whatever the return value for that block might be.
  * can be used with looping constructs and control structures.
  * when used with an if-else, all subordinate if-else's that might be chained onto the originally flagged one also carry the flag
12. import-from-as
  * used to pull in external modules/module definitions and provide them with a name in the namespace of the current compilation unit
  * this is, against the language specification, a statement - it cannot occur anywhere inside any code-structure as it is a meta-feature and not a part of the language meant for (or legal for) direct use inside code.
13. return
  * exit the current function and set its value to the result of the expression that follows.
  * its "return value" is the value of the expression, though I can't see any way in which a keyword/expression that actually *halts* the current codes execution could be a useful part of another expression.
  
Yep... 13 (okay, 17) keywords - although I'd call the three of the loop constructs "block acting functions" (for, foreach and while), as they can be viewed, by the compiler, as a hyper-specialized form of lambda-function.

And yes... I have also not, yet, had any real ideas on how to handle lambda functions, anonymous functions, closures, generators and a number of other things.
  
####Notes
Because I see terseness as saving a programmer time, I'd prefer there to be a short form for the import statement, but because of some difficulties of making this truly useful, such a short form cannot exist. This specifaction calls for modules to only modify their own namespace(s) and not the namespace of the user, so the 'AS' part, which gives the pulled in module a name, is required. The 'FROM' clause is also required because names are not unique and two different modules could have namespaces, internally, that are identical. (not that this should ever occur, just that it is possible and not against anything in this specification)

###Lambda Functions and Other Specialized Bits
While it is true I have not given much thought to how to represent lambda-functions, anonymous functions, closures or generators in this language, I do have some ideas that I've been working on that cover them.

For anonymous functions I've thought that it might be possible to have the 'function' keyword, when used without a name, generate a random name, internally, and return a "function pointer" that can then be assigned to a variable that would act as the functions name. This would make it very similar in function to Javascript - and this is one of the things I think Javascript gets right.

For lambda-functions there are a number of different notations I've seen (and used!) over the years, from the simple in Lisp and Python, which use a keyword to indicate such, to the "big arrow" notation of Javascript. None of them have any real draw for me, though the "big arrow" notation, with some modifications (since, unlike Javascript, we actually care about return types), is wonderfully expressive and beautifully terse.

When it comes to generators I only have my limited exposure to them - and use of them - in Python. But Python seems to have a good idea for how they work and just adds a single key-word that gets used in place of "return". The Python method already requires the 'yeild' keyword so it seems like it might be a good idea to add a touch of "syntactic sugar" to give the compiler a hint, ahead of time, that the function needs to have its code generated in a slightly different manner so that it can act as a generator properly. (the extra syntactic sugar is not required - when the compiler finds the 'yeild' keyword in a function it knows the function is a generator and can flag such, since the code isn't generated as the parse occurs but afterwards)

As to closures... I have no real experience with them and hence, have no opinion one way or the other as to what the best method of adding them to a language might be.

####Notes
For reference, the "big arrow" notation is (the following is (hopefully valid) ES6):
  ```javascript
  let add10 = (a) => a + 10;
  ```
  
And the same as the lambda example above, as an anonymous function:
  ```javascript
  var fun = function(a) { return a+10 };
  ```
  
My thoughts for lambda's and anonymous functions in this language would be:
  `Function add10 = (Number a):Number => a + 10;
  Function add10fun = function(Number a):Number { return a+10; };`
  
Note that they are both of type 'Function' and the differences are in the use of type-names and in the addition of the defined type being returned. The above example is nice, readable (once you grok the "big arrow" setup) and has a format that should be understandable and recognizable by a large number of programmers.

###Interfaces and Implementations
The compiler can and probably should be able to extract an "interface declaration" from any given compilation unit. This is, sadly, something that is necessary as we do not have the information available that byte-code compiled languages such as Java and the .NET family do. What this provides is a way of keeping the code modular and structured as well as providing the compiler the information it needs to do its job. (and by extracting the interface from the compilation units implementation, it can solve a host of errors)

The programmer should, however, compare the resulting interface to whatever specification is available - s/he may have made an error in writing the "implementation" and it is now different from what the specification calls for. Yes, this can be caught in automated testing, but it always pays for the programmer to review his/her work before pushing it off to the repository for the unit-tests and regression-tests to be run.

But there is another form of "Interface" that is sometimes referenced in these documents. That one is related to what Java uses the word to represent - a set of member-functions and variables that a class can implement and be treated the same as any other class that implements them. In fact, I used the term "iterable" in describing the 'foreach' keyword - "iterable" would be an "Interface".

So it seems that there is a need for another term - what to call these two related but different concepts, I have no clue. The two are not so different, although one describes a complete class and the other just a fragment they both convey essentially the same information. Perhaps a keyword could be added that would flag a given module as being a "pure interface" instead of a "module interface" is all that is needed and the two can otherwise be identical...

As to the other term that makes up this sections name? Every "compilation unit" is, since it contains all the code that isn't defined in a module, an Implementation. Simple as that.

###Not The End...
This file is a collection of thoughts and suggestions for the actual structure (Syntax and Grammar) of the language and, beyond that, its actual internal functioning and how the compiler can/should work.