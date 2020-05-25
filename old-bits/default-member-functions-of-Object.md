##On 'Object'
###Overview
'Object' is the base class from which all other ovjects in the language derive. It is always there, with no need for declaring the inheritance. This exists as more of a conceptual feature - 'Object' itself is not used in the language directly for anything. It acts, instead, as a place to collect "safe and sane defaults" for some features of the language. These features are included for a variety of reasons - said reasons will be discussed in later parts of this document.

###Currently Known/Required Features

1. dot-operator (aka: address lookup and "does this object have this name as a member ?")
  * The dot-operator is used to access members of an object - that lookup can actually pass through an overridden function at some point, so we provide a default implementation that the compiler is free to completely elide and optimize to a direct address use.
  * Used in reflection to find the address of any given member - as such, the default function is very useful.
2. call
  * sometimes you want to be able to use a variable as if it is a function - see how Perl's "Autoload Methods" feature is used most of the time. In that case, you can override this function to execute a related function with whatever params were to be passed to this one, as well as whatever other parameters you like. Base form of the function is 'function call( <method name>, <parameters> ) : Object' - first parameter is the name of the function to call, the rest is the parameters being passed to the function. An example and explanation of it follows:
    ```
	...
	function call( String methodName, Parameters params ) : Object {
	  if( lookup( methodName ) == MEMBER_VARIABLE ) {
		... use reflection to set the value of 'methodName' to the value of the first/only item in iterable 'params' ...
	  } else if( lookup( methodName ) == NO_SUCH_MEMBER ) {
		... return an error - item does not exist ...
	  } else {
	    return super.call( methodName, params );
	  }
	}
	...
	```
  I've elided most of the boilerplate that would surround this and have not included code that involves API's that have not been developed or fully proposed yet. What I have done is used the proposed 'super' keyword for accessing the immediate parent-object's implementation of 'call' - passing things up the line. As there has not, yet, been a proposal for an object modifying itself at runtime - even an iterable adding new items to its internal store - I've made that an error in the above short example. This is a (rather poor) example of how you can make a variable look like a function to others by overloading the 'call' function.
3. Data table for reflection/introspection/RTTI
  * simply put, this data table has to exist if Reflection, Introspection or RTTI is used at all in a program. This contains data about all the names of data and function members, their types, their return value/values, etc...
  * Is a simplistic database - consider it similar to a NOSQL json-based document store - users should actually be able to add their own data to this, for annotating code, etc...
4. toString/stringify
  * All objects should be able to provide some textual representation of themselves
  * This is part of debugging and also exists as a complement to the formatted output system, which can interpolate any given object into a string for output thanks to this function.
  * Should have hooks/connections to the L10N and I18N systems so the strings produced can be localized and translated as needed.
  
Yep - four required features that are definitively part of 'Object'. Three of these make up core parts of the language and help with its flexibility.

###Proposed Features

1. Comparison Operators
  * A set of sane defaults for the equality and inequality operators are a definite requirement.
  * Special-case handling in the compiler can make this highly optimizable for some types, if needed
  * Included here because all operators have a possibility of being overloaded and without a proposal for how this would work not yet available, it becomes necessary to include features that might be relied on by an operator overloading method.
2. Math Operators
  * Sane defaults that are basically no-ops
  * As with the Comparison Operators, safe and sane default implementations included because it is needed to back-stop one of the potential forms operator overloading could take.

These two features actually make up over 20 separate functions. But without these, one of the potential methods of handling operator overloading disappears. (and it is the method that I would prefer, as it keeps the language fully orthoganol - everything has the same sort of execution path)

###Notes
This acts more like an inherent interface that all Objects possess and can provide their own implementations for. It is defined as a base for everything so that there is no need to worry if something actually implements a feature - if the feature is a key part of the language or important enough that it should be implemented across the board, it will be added here.
