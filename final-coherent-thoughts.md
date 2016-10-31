##Last firm ideas

###Standard Library
  * Should cover a set of the following functionality:
    1. Filesystem Interface
	  * Synchronous and Asynchronous interfaces
	  * Format and Encoding agnostic
	  * Extensible to be able to do such things as treat archives and other files that present a filesystem-like structure internally as a part of the filesystem.
	  * Provide for watching filesystem components for modification
	    * Allows for triggering a read as new data becomes available in a file
		* Allows for a system to refresh the UI of a filesystem browser if the contents of the filesystem currently being viewed changes
		* On Linux this would be doable via the dnotify system - other OS's generally have comparable setups.
	2. Network Interface
	  * Ideal base setup would provide for generic, protocol agnostic read/write operations
	    * Extensible so various low-level protocols (TCP/IP, NetBios, etc...) are easily utilized
	  * Likely base setup would sit on top of OS provided abstractions and only worry about user-level protocols
	    * Extensible so that bindings for various user-level protocols are easy to implement
	  * Synchronous and Async operations
	  * Both Synchronous and Async operations should provide notification hooks for various states
	    * Would include:
		  1. New Connection From Remote Requested
		  2. New Connection From Remote Completed
		  3. Connection Lost
		  4. Data Available
		    * Should also have a method of finding out how much - in *Nix the FIONREAD ioctl can do this, Windows has something similar...
		  5. User Event
		    * Included as an expansion item and to cover for things I might have forgotten
	3. Reflection
	  * This should use the internal data tables that the compiler can build
	  * Actual details of this left deliberately un-specified at this time as this is something I have limited experience with
	4. User Interaction
	  * Here is where anything that is specifically for user-interaction should live
	  * Should TUI or GUI elements be in the standard library, they will also go here
	5. Process
	  * Threads, processes, co-routines, etc...
	  * Specific data about the current thread/process that is being executed should be here as well
	6. Debug
	  * Everything that is specific to debugging and testing goes here
	7. OS
	  * Anything that is OS specific goes here
	8. ENV
	  * All data about the operating environment goes in here... This includes the following:
	    1. Command Line
		2. Environment Variables
		3. Executable Name
		4. Working Directory
		  * this is either the Windows "Working Directory" or the *Nix "current working directory" (ie: what directory you were in when you started the program)
	  * More items may be added if they are found to be needed or useful.
	9. Streams
	  * A generic implementation of a "Stream" system for any other part of the system that might need item
	  * Should be used by FS and Network for handling all read/write operations
	10. Math
	  * This should contain any and all extended basic math functions that are commonly needed
	    * Basic trigonometric functions
		* Conic section functions
		* Mathematical constants
		  * PI is only really needed to 5 places for the most common uses, but some of the more common of the uncommon ones call for more precision
		  * TAU will be provided as it is somewhat better to have as a decent constant instead of having to recalculate
		* Base-10, Base-2 and Natural logarithms
		* Generalized Power function (perhaps this should be done as an operator - both the carrot (^) and double-star (**) notations are popular)
		* Square and Cube root functions
		  * A generic n-root function would be nice, but is somewhat prohibitively hard to produce one that is both execution time friendly and reasonably accurate.
	11. Exception
      * Though not discussed or mentioned, no modern language would be complete without an exception handling system
      * Should contain the basic Exceptions for the system and a generic Exception object that can be inherited from
	  
  ### Project Layout and Program Format
    * Each file shall contain exactly one Class/Object
	* Hierarchies of Objects can be created via directory trees, similar to C# and Java
	* At least one Object should include a function named "main" - this will be the programs entry point
	  * The name eventually chosen for the entry point function - or whatever means is chosen to specify it - should have some special semantic so that the final linking can easily find it and bind it to the initialization/closeout routines that will be used for the startup and shutdown of every program.
	  * ___NOTE___: The above exposition is only necessary if this specification allows for the user to provide their own code and/or object-files to provide the init/shutdown routines.
	* It is requested that programmers try to keep their code in some standard format, with module requirements at the start, followed by the object being defined in the file. Other than that, the only request is that people actually document the code they write.
  
  ### Closing Thoughts
    As a firm believer in Test Driven Development and its capacity to help make code adhere to its agreed upon specifications, I've defined the 'Debug' module of the Standard Library to include a sub-module just for testing code. Having used several frameworks, I can see the utility of them all, but in this case I think that better programmers than I will have to make decisions about it...
	
	Other than that... TAU is, just in case you don't know, 2*PI - it is actually used in a lot more places than PI alone is. Other values that would be good to include in the "constants" section of the Math library are 'e' and the square root of 2 - both of which are, again, rather commonly used. Another that might be a good choice is Phi - the so-called "golden ratio".
	