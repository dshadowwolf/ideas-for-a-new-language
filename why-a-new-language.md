### Why?

A programming language is a tool for telling a computer how to do any given task. Many languages that exist carry with them ideas or restrictions based on the machines of the time, the ideas that caught hold and the few things that people felt worked well. A lot of the issues the "computer scientists" think that the way forward with programming is to push their favorite paradigms at people as being "more correct" or "less prone to error" without thinking about the people actually writing or maintaining programs for a living.

This has led to situations where the computer scientists are telling people to move to a different paradigm because it has restrictions built-in that will stop certain common sources of errors, or where a lanugage meant, originally, for small, flexible applications that were to be embedded in web-pages has become a major enterprise language.

Back in the 1950's - during the baby years of computers and the field of computer science a well known professor stepped forward with an idea for a simple language that defined itself by encompassing an entire concept of computing. That language had no "keywords" (as a modern language would define them) and a minimal set of operators - everything beyond the very basic levels of it was defined in the language itself. When a full-fledged maco-language was added that could re-write the code the full power of this simple, expressive language was released. But... this language was relegated to "special use" over the years and, while descendents of it have started to catch on, the base language itself has not. That language is Lisp - called by most a "functional" language (because, I think, they don't understand that it has no preference for any programming paradigm) - and John McCarthy the genius behind it.

So... Why a new language, then?

Because there are no languages in common use - or even currently available - that attempt to provide a toolkit for the end-user that is unobtrusive and only intrudes enough to get the job done. Java has lots of boilerplate and odd requirements, the Lisp family is stuck in a niche, C and its descendents/relatives C#, C++, ObjectiveC and numerous others carry their heritage with them and the list goes on. Rather than trying to force an existing language to fit new ideas, crafting a language from the ground up that has ways of allowing the inclusion of new features without breaking the existing language is a better solution.

Hence this repository of ideas - and hopefully code implementing them. The only intrusion and dictation about the structure of the code is the forceful "everything is an Object" requirement. That requirements purpose is noted in the paper "On Object Requirement" in this repository. Forcing a base object-oriented nature into the language was a hard choice, but the best one available given the requirements of some features.

The key points of this language are that the language itself is only going to provide the features that are needed for all required parts of the language itself to be implemented, with a few additions that are going to be there for the module writers and to assist in implementing the default runtime library. As part of keeping the core language as simple as possible, most of the syntactic sugar is actually going to be implemented in the full-featured macro-language that is defined as a companion - so that this language has a whole second level of utility.

What will not be part of the core design - of either the standard library or the language itself - is the build system. While many ideas exist for how a build system should be properly implemented, it is my experience that a projects build-system is dictated by the requirements of the project itself and any language that has such a system defined as part of the languages specification faces issues in adoption as projects that might otherwise switch to that language find themselves unable to as the build-system is lacking in some required feature.