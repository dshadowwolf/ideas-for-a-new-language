## On the Philosophies of Turned-c

### Introduction

Turned-C is a programming language based on several core concepts that can be considered its defining philosophy. These concepts have driven the entire design and have contributed to the development of its own, idiomatic syntax as work on the specification and a "reference implementation" of the compiler has progressed.

### The Concepts

#### The "Toolkit Philosophy"

Every programmer I know—including myself—has, at some point, wanted to strangle someone who insists on solving a problem in Java when Perl would be the better choice. The "Toolkit Philosophy" is my attempt to make that frustration a thing of the past: Turned-C is designed as a toolkit you can use to solve any problem, in any domain, with just a little work—so you can define and solve the problem entirely in Turned-C.

This was the very first thing I ever formally codified about Turned-C, nearly 10 years ago, when I set out to design a language that fit my own ideas and beliefs about programming. The form it took then can be summed up simply: "A programming language is a tool that a programmer uses to solve problems. It should do that job and otherwise get out of the programmer's way."

That definition has never changed, but the implementation has evolved. Now, it’s realized as a language core that provides exactly the pieces needed for the language to describe and extend itself—without ever getting in the programmer’s way. This is what Turned-C calls the "Bootstrap Core"—think of it as a Swiss Army knife: always useful, but never in your way.

#### The "Single Meaning" Philosophy

There are a myriad of symbols available to a programmer—some as simple as the humble plus-sign (`+`), others far more complex. In the more than three-quarters of a century since the invention of the first formal programming languages, a plethora of meanings have been attached to almost all of them: from the strongly mathematical basis of `APL` to the teeming morass of operators in `F#`. It can be hard to remember a half-dozen meanings for a single symbol, or to avoid issues where an operator has one meaning by itself, but a different one when next to another of the same type—as plagued `C++` for many years in its template definitions. The "Single Meaning" Philosophy is my attempt to solve that issue.

This idea took shape about five or six years ago, as I worked to introduce a syntax for Turned-C that would be distinctive—one that would force the programmer to actually think about the language’s concepts and how to apply them, while also removing the ambiguities in operator meaning that have long haunted programming languages.

In short: "An operator should have, preferentially, just one meaning; if it must have more than one, then all the meanings should be closely related." In practice, this has led to Turned-C almost feeling like an "operator soup" language, with a breadth of operators rivaling languages such as `F#`. But it has also led to a uniformity of expression that makes idiomatic use both understandable and predictable.

By giving each symbol a singular, dedicated purpose, we move the complexity out of the parser's 'guessing game' and into the programmer’s intentional design. You don't have to wonder what `*` does in this context; in Turned-C, a 'Socket' only accepts its matching 'Plug'.

#### The "Provably Correct" Theorem

Who, in the past two or three decades, hasn’t felt that moment of (understandable) rage when a bug in Windows or the latest AAA game brings everything to a crashing halt? Turned-C aims to address this problem at the language level—but in a way that’s different from `Rust` and other modern languages. Instead of putting "guardrails" and "safety bumpers" around everything, Turned-C tries to define the individual pieces so they can be "proven correct," through testing and other means.

I know I’m not alone in this desire—all programmers want their creations to be free of bugs and errors. Unit testing, integration testing, regression testing... it’s all very familiar, and it led me to a simple idea: if you build something out of small pieces that can each be proven correct when used properly, and then build larger pieces from those, by the time you’re done with the iterative process of building "bigger and bigger pieces," the end result should be easy enough to "prove correct."

That’s the "Provably Correct" Theorem in a nutshell: if you can prove the individual pieces are correct, provided you use them in the semantically correct manner, the final whole should be "Provably Correct"—and that truth should be easy to establish. This is why the language starts with just a small number of "Atomic Units". If we can prove the 'Physics' (the "rules and guarantees") of those units are sound, every macro, type, and function built on top of them inherits that same structural integrity.

We won't stop you from making a mistake, but we've made the pieces so simple that your mistake will be glaringly obvious.

#### The "Extensibility Gambit"

Extensibility: possibly the easiest thing to specify, and the hardest thing to get right, for any language designer. It stands out as a simple truth—a language can be made extremely fast to compile and produce highly optimized code, but if it isn’t extensible, it rapidly fails to gain a foothold. On the other hand, a language can be so extensible that it never truly finds its stride because it’s seen as impossible to use properly.

That’s why I call this a "gambit"—it’s the trickiest part of the entire language, and a core part of the philosophy. A language that is useful but static falls out of use; one that is not useful but extensible falls out of use as well. But languages that are both extensible and useful tend to stick around—look at Lisp and its whole family of languages. The original design can be traced back to the mid-1950s and, while its popularity has waxed and waned, it’s still around, still used, and even has new family members being added 70 years later!

Lisp and its descendants have survived because it was a liquid—it could flow into any shape. Turned-C wants to be the Refinery: a solid, dependable structure that still lets the user swap out the pipes and re-route the flow without tearing down the whole building. That is why Turned-C is also designed to be easily extended—by the _user_ of the language, no less—as well as being designed to be useful. That is the "Extensibility Gambit": if either part fails, then everything fails. The danger of the 'Everything is Extensible' path is that the language can become a Rorschach test—different for every programmer who looks at it. The Gambit is providing the power to extend the language without losing the shared ground of 'Provably Correct' code.

That is the Extensibility Gambit: I'm betting that if we give the user the keys to the foundry, they won’t just build more sugar—they’ll build a better machine. The 'Bootstrap Core' provides the foundry; the 'Extensibility Gambit' provides the keys. It is my belief that the community that eventually grows around Turned-C will use them to build a language that evolves faster than any single designer ever could.

#### The "Whitespace" Mechanic

Most programming languages that support operator overloading restrict it to the language’s predefined set of operators. Over time, this has led to oddities—like the `C++` standard library redefining the "shift left" and "shift right" operators (`<<` and `>>`) to perform IO operations instead of arithmetic.

Turned-C takes a different approach, built on Pratt Parsing. Here, the tokenizer and parser treat any collection of punctuation—a continuous, unbroken run—as a potential operator in its own right. This parsing philosophy led to the latest addition to Turned-C’s guiding principles: the Whitespace Mechanic.

Put simply: all operators must be separated from their surroundings by whitespace to make their identity clear. Think of it as 'The Hard Edge' rule: By exiling punctuation like `_` and `-` from the start and end of names (while allowing them inside), we create a mandatory visual gap. It ensures the eye—and the tokenizer—never has to guess where the 'Label' ends and the 'Action' begins.

#### `;` -- The Terminator

This document has been a joy to write, and I hope it leaves you enlightened as to why Turned-C was designed the way it is—and why it contains what some might find to be odd choices. Turned-C is not meant to be a "classic" programming language, nor is it meant to be a "modern" language of the sort that the authors of 'The Jargon File' called 'Bondage and Discipline' languages—those that sacrifice expressiveness for the sake of rigid, often intrusive, safety.

In designing it, I strove to take things in a subtly different direction: not aiming for guards against "risky behavior," nor standing in the way of programmers with a "You Shall Not Pass!" like Gandalf facing the Balrog. The goal was to provide a new way of thinking about programming—a mode where things are built to be tested, where each piece has a clearly identifiable meaning, and where the language stands aside unless the programmer is doing something truly terrible, like trying to access beyond the end of an allocated block of memory.

Turned-C is not just a programming language—it is an attempt to embody an entire philosophy of programming. That philosophy is what I hope has been clearly passed on to all the readers of this document. I hope this has provided you with a thought-provoking look into the mind of a programmer who spent a decade distilling the 'Noise' of the industry into a single, clean 'Signal.' Enjoy, and may the Refinery serve you well.
