Ideas for a new programming language to address historical issues of languages and the requirements of the modern computer.

1. Full module system with distinct namespaces
  * Modules have a name that fits within a global namespace for lookup
  * Lookup happens along a given filesystem path (or paths) that can contain either archives containing a cryptographically signed version of the entire module or a directory tree of the module
  * No module shall insert itself into the root namespace - this can be done by the user of the module, but not the module itself
    * No exceptions!
	* The root namespace is for the end-user of the language and the language itself (for keywords only)
2. Modules are, at their root, a collection of objects
  * Everything is an objects
  * Compiler is free to reduce an object representing a primitive to that primitive if no extended functions are used
    * That is, if it otherwise passes all requirements of the type!
  * Everything has a type - these types are important and while you can convert between them, this should not be done unless absolutely necessary
3. Objects have a persistent lookup table that defines their functions and data members
  * Every "dot" operator is actually a call to a fixed lookup function that returns the location
    * This built-in function of the every Object can be overridden
  * Adding the postfix parentheses-list turns this into a function call
  * The actual function call itself is resolved by calling a different function
    * This is another built-in of every object that can also be overridden
4. The basic syntax of the language is based on C
  * Multiple pass system
  * First pass is the macro-language
    * Unlike the 'CPP' macro language, this should be more like the common-lisp macro-language
  * Second pass is to resolve/load the modules
  * Third pass is actual compilation
