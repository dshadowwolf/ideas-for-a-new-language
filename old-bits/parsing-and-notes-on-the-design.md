#On the parsing of the language

In the general overview, it is noted that the language should/will be a lot like C. In matters of punctuation, syntactic sugar, some keywords and formatting, this is correct. However, the language is also different enough to require a slightly different and somewhat cruder method of parsing.

The most basic concept to making this a simpler to parse language is that there are no statements. Everything returns a value, even if the value is ignored entirely. Even various loops and control structures return values - what that value is has yet to be defined.

Parsing steps:

1. Read until an "end of line" character (generally a semi-colon) is found.
  * Read next character
    * Is it a block opening character ( a curly-brace, currently ) - if so, we need to recurse the parse to handle this. The reason being that the block represents a single "value" to the top-level of the parsing. Each "block" is a separate syntactic construct with its own lexical scope for variables and other features.
	* Is it a double-quote? Parse the double-quoted value out - this is a string and needs to be handled as such
	* Is it a semi-colon? If yes, then we've reached the end of the input for the current expression.
2. Split that line at spaces/punctuation into "words" - these words may be simple or complex
3. Each "word" is one of the following:
  * Keyword
  * Identifier
  * Constant
  * Syntactic Sugar
  * Operator
  : The words should parse to an expression that makes some sort of syntactic/lexical sense. This gets stored in the AST and any updates to the symbol table for the current lexical scope needs to get made.

#Notes

1. What is above is a very general overview of the process. Quite a bit needs happen behind the scenes to cover the various language features working correctly. In essence the "everything is an expression" bit is the truth, but a lie at the same time. By making it so that there is no difference between a "statement" and an "expression" in the language quite a bit of orthagonality is added, but at the same time some complexity is added in the parser as more data has to be tracked so that the actual value returned from "statement-like expressions" can be handled in a uniform and documented manner.

The return value of a "for loop" should be the last, terminal value of the index, should the index exist. Failing the existence of the index it should be some other value that has some meaning - what that is has yet to be defined as this document is being written. Other looping constructs - such as a "while" loop could return the value of the last statement executed prior to the test failing that triggered the end of the loop. (That same concept could be applied to for-loops without an index variable)

For other constructs that would be statements in other languages there is not yet any real idea of what the returned value could be. Perhaps a very Pascal-like system of assigning a value to an identifier - or, in this case, assigning a value to the keyword - could be used. (Again, this would also be applicable to the loop constructs).

As far as returning values from a function, I am quite partial to the "return" keyword, which can easily be extended to cover almost any situation imaginable - including multiple values to return.

2. With some work it might be possible to elide some syntactic sugar in certain contexts. Perl does this decently well and it allows for some specialization of the language into a domain-specific one by which modules are in use. Though the requirement that modules do not/cannot modify the root namespace themselves makes this more difficult...

3. Generic operator overloading - such as some have requested for other languages - might not be a good idea. The system in Python - whereby each operator actually calls a specifically named function in the class - or even a system with some syntactic sugar to make it like C++ would be better. However... there are arguments against allowing general operator overloading for the simple reason that some have classically mis-used it (such as the changes of the shift operator in C++ to work for I/O). This is something that requires much thought and some discussion with people more knowledgable on the topic than I am.
