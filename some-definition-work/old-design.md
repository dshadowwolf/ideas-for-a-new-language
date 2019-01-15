***Goal***

This whole dump of ideas in this repo has been about trying to define a language that has a well defined idea of computation, provides it and otherwise gets out of the programmers way. In researching the actual basis of computational theory it has become clear that the most well defined (and likely most well tested) theory is Lambda Calculus, which forms the basis of the Lisp family of languages and is formed, in its notation, almost identically to the Lisp family of languages.

Therefore the goal of this attempt to define an implementation of these ideas is to not implement a Lisp, but to implement a language that follows the base precept of Lambda Calculus in that it provides a set of operators that act on chunks of data and that the operators themselves can also be data. To this end - and to meet a lot of the ideas espoused elsewhere in this repository - the following 20 goals are set for this design project:

 1) A somewhat freeform, flowing language ala Perl 5 but with the language properly defined and specified.
 2) First class functions
 3) Pass by Value except where a pointer makes sense (ie: "objects" are pointers, everything else is a value)
 4) Well defined inheritance scheme
 5) Well defined, java-like "interface" structure
 6) C++ or Java type "generic" system
 7) Full macro-integration
 8) There is only one "type" in the language semantics - the expression. There are no statements - every piece of code has the possibility of returning a value
 9) Well defined module system
10) Well defined namespace system
11) Easy to use and understand data encapsulation
12) Full introspection and reflection
13) Text processing as a first-class part of the language, ala Perl
14) Simple, easy-to-use extension system - perhaps a wrapper around reflection/introspection to allow monkey-patching or a system similar to ObjectWeb ASM...
15) Network interaction as a first-class part of the language - major protocols (HTTP, Telnet, SSH, FTP, SFTP, etc...) in core language modules
16) Garbage Collected
17) Async ala NodeJS or Promises ala Scala as core features (or both!)
18) JavaDoc-like documentation
19) Code should be terse but self-explanatory
20) No hidden gotchas like the "=="/.equals() dichotomy of "identity" versus "value" comparisons, perhaps an extension like the "===" operator to allow for direct, is identically equal in type and value or "is literally the same object"

As we are trying for a loosely typed system, overall, where the actual "type" doesn't matter as much as it does that the type provides a given set of oeprators/functions, we can define some of the "object" system in the following manner:

1) All non-primitive types can implement any number of interfaces that define its interactions with other types
2) The left-hand side of an operator, at all points where the order of operations is clear, is the source of the interface function regarding the operation in question
3) Order of operations, outside of well-defined mathematical rules, is:
``1) Left-to-right evaluation of parameters in any given parameter list
2) Method-call
3) Field lookup
4) Pointer de-reference
5) Array reference
6) Non-method function call``
\* NOTE: if the immediate object does not contain the functionality for the operator, the interfaces it implements should be checked for defaults, followed by its chain of super-classes. Perhaps this would be best resolved during compile, but leaving it for run-time gives room for the "extension system" of goal #14 to attach.

Idea for a simple program that reads a .INI style file: \*\*
```
import system.io;

function main(args : Array of String) : Integer {
  if( args.length < 1 ) return doUsage();

  args.forEach( filename : String -> {
    try {
      input = new File( filename );

      input.readLines().forEach( line : String -> {
        bind REGEX_BASE line; // bind the variable general regex matches will work against
	
        if( /\[[\w\s]+\]/ ) {
	  /\[([\w\s]+)\]/;
	  system.io.formatted( "Section Header, Name: {}", $$1 ); // '$$' is a macro - basically "get REGEX_MATCHES"
	} else if( /\w+\s+=[^;]*/ ) {
	  /([\w\s]+)=([^;]*)/;
	  system.io.formatted( StringTable( HEADER("Entry"), COLUMNS( "Name", "Value" ), $$1, $$2 ) );
	} else if( /^;.*$/ ) {
	  system.io.formatted( "Comment: {}", line.trim() );
	} else {
	  system.io.printline( "Line not recognized, possibly a blank or garbage" );
	}
      } );
    } except( exception ) {
      system.out.formatted( "Error in work: {}", exception.message() );
      system.out.rawprint( exception.stacktrace() );
    } finally {
      input.close();
    }
  } );

  return 0;
}

// yes, if it takes no args, you can omit the list
function doUsage : Integer {
  ... output the usage ...
  return -1;
}
```
\*\* NOTE: This is a rough pass at a very procedural means of doing the work, but highlights that functions can be passed around and some of the concepts to regular expressions being an integral part of the language. The "bind" and mentioned "get" syntax is for managing class-global values that are generally not things that need direct interaction outside of specific language features. They are likely going to go away in favor of an API, for the most part.

Another major feature is that there will be a Lisp-like macro system that isn't just simple string substitution, like the C/C++ system most are used to, but a full-fledged means of generating code during the parse to give a means of eliding boilerplate and making programmers lives easier. There is a possibility of extending this out to an actual compile-time system that allows for manipulation of the AST for generation of code, such as how Project Lombok will generate getters and setters for Java applications using the Project Lombok provided annotations and this could, potentially, carry through to the "pure metadata" of classic Java annotations, which could allow for easy plugin systems. This macro language has yet to be decided on, but might wind up being something like M4 or AWK.

