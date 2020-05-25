##Strings, Translations and Internationalization
###Background and Reasoning
Historically a string has been nothing more than an array of small integers that, when mapped to whatever the local encoding for digital text was, produced an understandable stream of letters, numbers, punctuation and other textual features. In the modern world this is not something that can continue to exist, as programs written in the US or another English speaking nation might be picked up for use in Germany or another non-English speaking nation. Or, alternatively, picked up for use in a nation that does not use Latinate characters or has more than the base 26 latinate letters of English. This requires that the encoding be such that it can encode any given character set and other information about the language, such as ligatures - hence the decision to use UTF-32 as the internal encoding for all text.

UTF-32 provides full access to the entirety of the current Unicode standard - including parts that have not been accepted into the otherwise identical UCS-2 encoding standard - and also encodes ligatures and other information. This, combined with some localization information (are thousands separated by a comma? Is the decimal 'point' a period or a comma? Is text left-to-right, right-to-left or something even stranger, like a boustrophon ?) and a database of the different strings used in the program - and their translations into various languages - solves the problem of providing a program that is understandable by a wider, international community.

Moreover this class provides functionality that ties together with other classes to help with manipulating textual information and performing tasks such as parsing and extracting information from various types of data files. Because of the additional utility offered in all of these cases, this class (or something like it) is a required part of the language.

###The String Class
In a basic format, using my proposed grammar and syntax (with features not yet proposed elided), the String class would be something like:
```
/* Should implement 'Iterable' - proposal for inheritance and interface implementation does not, yet, exist so notation of such is not here */
Object String {
	function init() : String {
	   ... default initialization - nothing to really do here and how to return the new object has not been proposed ...
	} 
	
	function init(String other) : String {
	  ... copy constructor ...
	}
	
	function match(RE pattern) : Boolean {
	  if(pattern.matches(this.internal_store)) { return True; } else { return False; }
	}
	
	function toString() : String {
	  ... check localization language, get proper translation if it exists and return that otherwise return the value we currently store ...
	}
	
	... lots more implementing 'left', 'right', 'range', 'index' and all the functionality of 'Iterable' ...
}
```

Pretty much a lot of boilerplate wrapped around a few functions that actually work on the internal storage of the String itself. Because of the complexity and a lack of really solid ideas on what should be in 'Iterable' and 'String' itself, I cannot provide a full list of what the class should be. But note that it does have a utility wrapper such that you can hand it a regular expression as well as handing it to a regular expression - the function in this example is just a bare-bones wrapper around a quick test and the test itself should probably take a completely different form.

###I18N and L10N
For Internationalization and Localization I have no preference as to what is done in the back-end and how it is implemented. The people behind the GNU Project have a decent implementation of both, with translations of text handled through their 'gettext' system - utilizing this or replicating how it functions would not be a bad place to start on this whole section.

