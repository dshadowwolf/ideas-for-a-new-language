#Built-in Types and Collections
###Types and Sub-types

By design the most basic type that exists in this specification is the raw "Object" type. It, however, is not a real type - you cannot instantiate a member of type 'Object', as it is not actually a type but a specification of a set of interfaces that all objects must possess and can override as needed.

For the widest degree of genericity and utility the following base types are specified as part of the language specification:
  1. Number
    * Stores a number in an implementation defined multi-precision format to remove machine-level constraints
	* By being stored in a multi-precision format, all values represented by the Number 'type' carry a sign
	* Can be integer, fixed-point, floating point, rational, irrational or complex
	* The underlying storage format shall support seemless conversion between floating-point, fixed-point and integer values
	* The multi-precision value shall have limits of capacity no less than a 64-bit integer or an IEEE 754 "binary64" (double precision) floating point value.
  2. Boolean
    * Stores a single value - either True or False - in an implementation defined format
  3. Glyph
    * Represents a single "character" of text input
	* Shall be able to store any Unicode character - ideally the encoding would not matter, but because leaving too much to the implementation causes problems with program portability, UTF-8 is selected as the internal format, as it conserves space and has a direct mapping to the encoding still hard-wired into most computers - ASCII.
	* Technically this should be called a "character" or "code-point", as the glyph is the visual representation of the symbol, but something that would uniquely identify this as not being the classical "can be simply cast to an integer and is the smallest addressable size of storage" that C and other languages use "char" or "character" for was wanted - and "glyph" is much more unique (and more terse) than "code-point".
  4. String
    * Owing to brevity of storage and how computers work, a string is, classically, an array of a languages "character" type, with some form of special delimiter markings its end if the language doesn't otherwise have a manner of specifying the length or a requirement that all arrays be specified with some maximum length.
	* In more modern languages a String is a class/object/type that has storage for the component "characters" and any number of utility member functions for examining the string, extracting sub-strings, comparing the string to other strings and even changing the contents of the string. This is how the String type of this language works. It stores any number of Glyphs (limited only by storage and the representational ability of the underlying 'Number' type) and provides all the types of functionality mentioned.
  5. Regular Expression
    * Intimately tied to the String type, this stands alone because the complexity of the "Deterministic Finite Automata" that makes up a regular expression engine requires it.
	* Also... by having it as a built-in type a regular expression can be inserted directly into the program text as-is without any of the requirements for it to be represented as a string and passed through a complex set of processing steps. (Basically... you can use regular expressions in this language as freely as you can use them in Perl)
	* The actual regular expression engine to be used is left to the implementation, but shall not have any less functionality than engine provided by the "Perl Compatible Regular Expression" (PCRE) library, "PCRE2 version 10.22". This requirement is to provide a stable, minimum set of features to be available across the board and also to provide a proven working regular expression engine that can be used as a source of provably correct solutions that can be input into a test system testing the any given implementation of this languages regular expression system.
	  * __NOTE__: Factually the specification of that specific version and engine is used as some languages (most notably, JavaScript/ECMAScript) that specify a regular expression system have one specified that is anemic and of only minimal utility when the solution at hand calls for heavy use of regular expressions.
  6. Function
    * For the generic utility of such, rather than having a "pointer" - inclusion of which, in this language, is still undecided - as the generic handle for passing around references to a function, there is a specific 'Function' type.
	* This type acts as both a handle that can be used to indirectly call a function, but also as the base type of a closure and lambda/anonymous function.
	* Is also a part of the reflection system and contains information about the function, its parent object, source-file, source-line, etc...
  * ___NOTICE___: Three of the types mentioned in the [Readme](ideas-for-a-new-language/README.md) that are not in the above list. Those will be covered in a bit, after some exposition, explanation and definition of terms.
  
The six above types - truthfully just 3 of them - can be used to represent something like ninety-five percent (yes, 95%!) of the data out there right now. However, to help the programmers and save them some time a few more types - really just specializations and sub-classes of one of the above types - is required.

That is where the "sub-type" specialization system comes into play. A sub-type is a type that shares most features with its parent type, but specializes off of them, either in restricting the range of values or by defining a specific storage format. These base sub-types are all of Number and they are:

  1. Integer
    * A sub-type of Number that stores only integers - no rational, complex, floating-point or fixed-point numbers.
  2. Float
    * Similar to Integer, but strictly stores floating-point numbers
  3. Fixed
    * Same as the other two, but solely fixed-point
  4. Real
    * A "real number" represents a quantity along a line - can be of any type except fixed-point or complex.
  5. Rational
    * Any number that can be expressed as the quotient or fraction 'p/q' where p and q are two integers and q is non-zero. This is basically a sub-set of Real that excludes irrational numbers.
  6. Complex
    * A complex number is any number that can be expressed as 'a + bi' where 'a' and 'b' are real numbers and 'i' represents the imaginary unit (i*i == -1)
  7. Irrational
    * Any real number that cannot be expressed as a ratio of two integers. This includes values such as PI and the square root of 2
	
Of those seven sub-types, only the first two are required in any generic programming language - and only those two (Integer and Float) are actually suggested as hard-requirements for this language. The other five - including 'Real' and 'Complex' (which are mentioned in the [Readme](ideas-for-a-new-language/README.md)) - are there for illustrative purposes, completeness and because they are useful in several fields of computer science.

###Collections

A collection is a classic idea in programming, and a useful one. Historically most languages have had just one collection - the Array. While this is good for a very large selection of use cases, some things are better with a different type of collection, such as the Hash-table (or Hash-map, dictionary, etc...)

For simplicities sake, and because almost all other data structures can be built from sub-classes of just two distinct types of collections, only those two types of collections are going to be specified for inclusion as built-in parts of the type-system. Those are:
  1. The Array
  2. The Dictionary

Truthfully the Array could be defined as a sub-type of the dictionary with the key restricted to just integers, but by providing two distinct types we allow for some easier optimization and different choices for data-structure design.

  1. Array
    * a storage type with integer keys, can be easily stored as a linear section of memory
    * for utility: pushFront() and append() for adding items to the start/end of the array
    * length member variable storing the number of items in the array
  2. Dictionary
    * Also called an associative array this uses a creation-time defined type as the key
    * Versions which use a string as the key are known, classically, as hash-tables or hash-maps
    * Some version of hashing should be used to turn the key into a value more machine friendly
    * Is not required to maintain any form of ordering for the keys/values
    * Implements a 'count' member-variable to report the number of unique items
    * Implement a 'keys' function for retrieving an array of keys in the dictionary
	
More specialized types of collections can be built by sub-classing these two to provide any required semantics. For instance, a 'stack' could be built from an array in a very easy manner. (I'd provide pseudocode, but don't wish to risk pushing the language in a specific direction without more thought)

###Other Thoughts
A pointer and/or reference type specialization is required in any language that is generic enough to be used for both freestanding (ie: bare-metal) work - like embedded or OS kernel level systems work - system library and utility and user program. That means that this language should include them - but what semantics they should have and what restrictions and requirements for their use should exist will either require someone with more experience at that level of programming than I have or a lot more research and thought on my part.

	
	