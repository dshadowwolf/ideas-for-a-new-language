## The History and Continuing Evolution of the Syntax of Turned-C
##### An Informal, Introspective Dive into the Thought Processes and Memories of a Language Creator

### Introduction

Approximately six years ago, I was faced with a great dilemma: I had most of the philosophical underpinnings for a new programming language in place, but its actual representation—the syntax that would encode everything those underpinnings demanded—eluded me. Then one day, as I was floating between sleep and wakefulness, a solution came to me: mathematics had already provided an answer in the form of a well-known notation for functions, that of `f(x) = y`.

From that beginning began an iterative process of turning that base idea into a workable syntax for a programming language. That syntax has become what I now call "idiomatic Turned-C." The name of the language itself, `Turned-C`, was chosen because the philosophy behind the language presents an almost inverted view of things compared to `C`, a language most of the world has some familiarity with.

Instead of targeting an abstract, somewhat idealized vision of a computer like `C`, Turned-C targets a mode of thought—a philosophy and a related set of concepts around the very idea of programming and problem solving. We’ve all spent decades wrestling with 'Platform-Defined' quirks that make the same code act like a different person on a different chip. In Turned-C, those "shifting sands" have been traded for a single, unified model—one that stays the same whether you're on a microcontroller or a supercomputer. It’s not about absolute safety; it’s about absolute predictability.

This has led to numerous decisions that have created the somewhat unique notation and syntax of Turned-C. In this document, I hope to explore those choices, explain the reasoning behind them, and show how idiomatic Turned-C evolved and grew into the (to me, at least) beautiful setup it is today.

### "Pre-history"

As I noted in the introduction, Turned-C's syntax began with the idea that anyone who has had even a basic education—just plain secondary school—has encountered mathematical "function notation." But, as `APL` proved more than half a century ago, pure mathematical notation does not make a workable programming language, nor one that is actually possible to read clearly.

Still, I became enamored with the idea and decided to work on it, trying to see if I could write even a basic pseudo-code "Hello, World!" program using that notation, assuming a basic `print` function would be available. That very early attempt has not survived, but a companion piece I added to an essay about the syntax idea did. This is the earliest surviving form of what would evolve into Turned-C:

```
func f(x: integer) : integer;
func square(x: integer) : integer;
const mult : integer = 100;

f(x: integer) = square(x) * mult;
square(x: integer) = x * x;

print_line("square of 2 times 100 is {}", f(2));
```

I know, I know: it looks nothing like Turned-C at all and has more in common with scripting languages than with systems programming languages, but the seeds of the language had been planted. That scrap of code is the very genesis of Turned-C's syntax, and you can see traces of it in Turned-C to this day.

At the time, `=` felt natural because that’s what we’re taught in school. But the more I looked at it, the more it felt 'Static.' It describes a state, but it doesn't describe the action of a system in motion. In a refinery, you don't just want to know that the tank 'equals' the oil; you want to see the oil moving into the tank.

But, like all preliminary ideas, this did not last long—it was ultimately unwieldy and embodied a lot of concepts from classical programming, instead of mapping out a route into the future as I'd intended. It soon started to evolve and gained the first hints of what would become the `inject` operator, as can be seen from another sample that survives:

```
fn square(x: u64) -> u64 {
  x * x
}
```

Yet all of this is still the very, very early days of the language, and I still hadn't accepted that the true path forward was to change from thinking about "data" and "function" as wholly separate entities, nor had I accepted that it should involve thinking about the movement of data instead of just manipulation and mutation.

Soon, though, I started to see that path, as the last of these early, surviving attempts at defining a syntax shows. In it, I had the idea that the "name" and the "contents" were different things—which would, eventually, lead to the modern syntax. In fact, with this last example, you can see where Turned-C truly began to take proper form.

```
define square: mutable int = (x: integer) -> { x * x };
define f: mutable int = (x: integer) -> { square(x) * mult };
define mult: int = 100;

/* for illustrative purposes, definition of program entry point has not been decided on */
print_line("square of 2 times 100 is {}", f(2));
```

This was the moment the 'Material' and the 'Logic' started to blur. By treating a function as something you define just like a variable, I was accidentally stumbling into the Toolkit Philosophy. I was realizing that a name is just a label for a capability, whether that capability is a number or a block of code.

At that point, all I needed was a simple push—this came from several friends—and the evolution of Turned-C sped up, leading out of its "Pre-history" to its earliest, "modern" form...

### Early "Modern" Turned-C

At this point, you might expect me to describe an exciting "Eureka!" moment, but in truth, it was a discussion about simplicity that led to the next great shift in Turned-C’s syntax. Simplicity, the representation of structured data, and the question of how Object-Oriented programming might fit into what was shaping up to be a solidly `functional` language—all played a role.

The concept was simple: I had already arrived at the "Single Meaning Philosophy" and was working to ensure the emerging syntax for Turned-C lived up to that demand. My next "eureka" moment happened not as a flash of inspiration, but in classic scientific fashion—a "huh, it actually is that simple." Then it hit me: I already had the perfect way to represent structured data in Turned-C. It just required taking an operator that is often overused and under-specified in other languages and giving it a strict definition. The humble parentheses—usually relegated to holding the boring stuff—were suddenly the stars of the show.

Also, you’ll notice, that I still hadn’t had the inspiration for the `flow` (`<-`) operator at this point and was still thinking in terms of assignment rather than data motion.

```
define point: structure = ( x: double, y: double, z: double,
                            addOther: mutable point = (other: point) -> {
                                ( x + other.x, y + other.y, z + other.z )
                            } );
define location: point = ( 10.00, -25.42, 15.33 );
define newLocation: point = location.addOther( (-5.25, 25, -16) );
```

Yes, I had already seen the simplicity and "provable correctness" offered by making "variables" immutable by default, but that’s not what I was trying to show with this historical example. What you see here is possibly the earliest surviving attempt to define an _object_ in Turned-C, marking the beginning of its expansion toward Object-Oriented features alongside its `functional` and `procedural` roots.

Full disclosure: this is also the first time structure pops up. But remember, this is still very early, and the language was suffering from a lack of good ideas about how to structure and indent things for readability. However, it’s enough to show the origins of `structured data` in Turned-C and where the concept of `structure as type` began.

What followed was a flood of inspiration. The first was like a hammer-blow: assignment was antithetical to the design that had grown around the `define` keyword and the idea of `names` (symbols) as the root entity of the system. That led to the realization that the `inject` operation—the "insertion of context"—was like a "pump" emptying the keg in a bar, and that is when the `flow` operator was born. A simple concept, really: data "flows" in Turned-C instead of being "assigned." This started to make it clear that the language wasn’t about classical computing concepts, but about the 'motion and transformation of data.'

Almost immediately after came a second realization: I had been falling into a trap with function calls, thinking of them as "passing arguments" instead of something more fundamental. It took many long conversations with several learned friends (including a professor of mathematics), but I reached a conclusion: a "function call" isn't about "passing arguments"—it's about "augmenting the execution context." I kept the classic 'function(args)' look, but only as a peace offering to programmers moving over from other languages. Under the hood, though, it was just syntactic sugar. But that realization led to the birth of the `call immediate` operator (`<=>`). Its form is meant to evoke the "flow" of data into the context of the function, enabling its proper execution.

These two realizations led me to revisit an earlier sample of the syntax and modify it to show these new ideas, creating this:

```
define square: mutable int <-> (i: integer) -> { i * i };
define f: mutable int <-> (x: integer) -> { (i <- x) <=> square * mult };
define mult: int <-> 100;

/* for illustrative purposes, definition of program entry point has not been decided on */
(format <- "square of 2 times 100 is {}", args <- [(x <- 2) <=> f]) <=> print_line;
```

Note the use of `flow` to give value to the "parameter names"—this syntax would not last long, though it is an excellent example of my thinking about Turned-C at that point. I had thought to require explicit, named parameters, with the names needing to have the data "flowed" into them. While conceptually elegant, it turned out to be cumbersome and unwieldy. But it really hammered home that 'data motion' was the soul of the language.

Another feature that appears in this sample, not discussed previously, is an early form of a "wrapped call"—something of a cross between an inline function call, a classic parenthetical group, and a lambda function. Looking back, I can only guess at what I was trying to achieve—it was a tangle of ideas that, in hindsight, would have been a nightmare to specify.  Sometimes, you have to try the wild ideas just to see why they don't work.

I thought I was on a roll, but then the engine just... stalled. The "ship" of Turned-C had hit the doldrums, and the ideas stopped flowing. But, in a fashion I’ve become used to, as I was trying to sleep one night, it struck me that I had never fully connected the idea of `structured data` to Object-Oriented programming in a concrete way. On top of that, I had a real "eureka!" moment the next day while working on a Python script for a client. The script itself had been created by another programmer, and I was just the latest in a line of maintainers—explicitly forbidden from rewriting it, as the client put it: "I paid for it once, I'm not paying for it again."

That realization? It was that deferred—sometimes called "lazy"—evaluation was actually a "great optimization." It made me realize Turned-C was missing a trick by ignoring lazy evaluation. So, I sat down and sketched out a new operator. I wanted the new operator to look like the "immediate" one so they felt like part of the same family—providing a sense of familiarity and shared meaning.

After finishing the work on the script, I turned my mind to these realizations. What follows is the result of some thinking and a decision to demonstrate Object-Oriented programming to a small degree—and to show how the ‘double-ended, blunt, fat arrow’—my own branding disaster—worked just like the ‘double-ended, sharp, fat arrow’ of the `immediate execution` operator of the `immediate execution` operator in the source code, but with the actual "call" to the function deferred until the result value was actually required. Here’s that exact example:

```
define point: structure <-> ( x: private constructable double, y: private constructable double, z: private constructable double,
    addOther: mutable public point <-> (other: point) -> {
        ( x <- this.x + () <=> other.getX, y <- this.y + () <=> other.getY, z <- this.z + () <=> other.getZ )
    },
    getX: public int <-> () -> { this.x }, getY: public int <-> () -> { this.y }, getZ: public int <-> () -> { this.z }
);
define location: point <-> ( x <- 10.00, y <- -25.42, z <- 15.33 );
define newLocation: point <-> (x <- -5.25, y <- 25, z <- -16) [=] location.addOther;
```

At this point, I had a much, much better idea of how to structure a Turned-C program—this form is far easier to read and more pleasing to the eye. It demonstrates the original idea for what would evolve into the `visibility` annotation, as well as showing the "implicit reference to the parent structure" that I was calling `this` at this stage in the syntax’s development. More than that, however, it marks the introduction of `[=]` and the concept of deferred execution to Turned-C.

This is when the last piece of Turned-C’s early, modern form came to me. I’ll admit, I was a bit stuck—macros were giving me headaches, and I couldn’t see a way to “lift” a method from one context and execute it in another. That was a problem, especially if I wanted to implement the “Provably Correct Theorem” properly. But then, out of nowhere, I had an insight—on an unrelated topic, no less—about flow control and how to allow truly complex logic in a slightly simpler manner.

That idea was for the “Lambda Predicate”—the original form of which actually still survives, though with slightly modified semantics. But when I first sketched it out as part of a `switch` construct, I didn’t have the exact semantics of either the `switch` (which would later become the `match` macro) or the `DEFAULT` predicate figured out. I treated ‘default’ as if it were magic, assuming it could live anywhere—even at the start of the list. It was a bit like putting the 'Exit' sign at the back of the theater and hoping everyone finds their way out. Looking back, that was a bit naïve—but hey, sometimes you have to let yourself be a little silly to make progress.

```
switch( <test value>, (
  ( [[ DEFAULT ]], { 0  }), // DEFAULT is a built-in predicate that matches only if no other predicates match
  ( [[ it == 0 ]], { 0  }), // no fall-through in this case
  ( [[ it < 0  ]], { -1 }), // expressions and blocks of code can be used, with the parameter "it" acting as an input parameter for the implied lambda
  ( [[ it > 0  ]], { 1  })  // might want to think about adding an operator for "truth" and/or "truthiness" to make such things explicit rather than implicit, but...
))
```

And I’ll admit it—at this point in the life of Turned-C, things went quiet for many years. The wellspring of ideas had dried up, and inspiration was hard to come by. The essay I’d been writing to track the design ends with a jumble of raw thoughts and a laundry list of things that just didn’t work out—including a brief flirtation with Lambda Calculus as a possible new foundation. But that’s a story for the next chapter...

### Modern Turned-C

I had intended to step away for a few days, a week at most. But life happens, and at least six years later I found myself getting burned out on a project I’d been itching to tackle for longer than Turned-C had even been an idea. The hard slog of wrangling those subsystems had me looking at Turned-C the way an exhausted parent looks at a photo of their life before kids—with a mix of nostalgia and a desperate need to go back. So I went searching through my hard drives and my GitHub repositories. I found a small directory named `neolang2` (truly the pinnacle of my naming skills at the time) and my GitHub repository—the digital equivalent of an attic full of half-finished furniture.

As I scrolled through the foundational essay I’d been writing to describe my journey to develop the syntax of Turned-C, remembering more than reading the raw text, the nostalgia became overwhelming and suddenly I was having new ideas. The issues that had caused me to take a step back no longer seemed insurmountable, and lessons learned during the periods where I had to acclimatize myself to new languages (or new versions of them) offered a whole new set of directions I could go. Armed with the heavy-duty power of Pratt Parsing and LLVM’s OrcJIT, I wasn’t just sketching ideas anymore—I was bringing out the power tools.

The discovery of OrcJIT as a part of LLVM really set my mind ablaze—with it, I could implement macros! Not as a second language layered on top of Turned-C like Kernighan and Ritchie had done with `C`, but as a first-class citizen of Turned-C. By using the same syntax and a few "decorative" operators, Turned-C could stand on the shoulders of McCarthy and Lisp—a lighthouse beacon pointing the way into the future—even if that future involved a terrifying amount of parentheses...

But it was my discovery of Pratt Parsing—the pinnacle of top-down, extensible parsing—that truly changed everything. The "Toolkit Philosophy" I had long envisioned for Turned-C was finally within reach! Now I could strip what had grown into a veritable "operator soup" down to the few components needed for the language to be self-descriptive—the full specification I'd stalled out on writing for Turned-C could actually be written in Turned-C itself, with nothing hard-coded in the parser or tokenizer beyond the handful of atomic components needed to make it all work.

This led, directly, to what has become the fifth member of the "Pillars of Philosophy" that form the foundation of Turned-C—the `Whitespace Mechanic`—the rule that white space isn't just aesthetic, but the hard boundary that keeps our 'dumb' tokenizer from getting confused. I decided to just jump in and begin implementing the language immediately, with only a vague idea and the existing syntax concepts. While I could probably have leaned on the venerable Lex (or its GNU descendant Flex) for building the tokenizer, I was itching to write code in C, not a descriptor language. This left me with a decision: How could I allow users to define potentially lengthy new operators and not have the tokenizer "break them up" because it found a pre-defined token at each step?

The solution was to make the tokenizer—and, ultimately, the parser—a bit on the "dumb" side. What I mean is, the tokenizer would detect tokens in a well-defined, predictable manner, using certain shifts in "character class" and, for absolute certainty of separation, whitespace to detect the extent of each new token. And the parser would accept the tokens as they came, interpreting them literally instead of trying to decompose them. This means that, should a programmer see a need for it, they could define `<<-?` in Turned-C as a new operator and start using it without having to build a new version of the compiler.

Sadly, I did not write any examples of code covering this part of Turned-C's evolution—I had added a fifth pillar to the foundation, but I was deep into trying to actually write the parser and tokenizer, my grasp of the intricacies of Pratt Parsing still rudimentary. So, since I cannot share what doesn't exist, I will share this—the original form (still in the raw code of the Turned-C "Stage 0" (`Hosted`) compiler) of the routine to extract an `operator` token from a source file:

```c
char *get_operator(char **ins) {
    char *work = *ins;
    char *coll = NULL;

    while (ispunct(*work)) work++;

    ptrdiff_t zz = work - (*ins);
    coll = calloc(zz+1, sizeof(*coll));
    coll = strncpy(coll, *ins, (size_t)zz);
    (*ins) = work;
    LOG_TRACE("get_operator: returning '%s'", coll);
    return coll;
}
```

So yes, I was being entirely serious when I called the tokenizer "dumb" in how it operates. It’s not exactly deep learning; it’s basically the tokenizer equivalent of a golden retriever—it sees punctuation, it grabs punctuation, it brings it back to you. Simple, but _effective_ and encoding exactly the rules demanded by the ‘Whitespace Mechanic’ and the distinction between a `BAREWORD` (anything that is not an operator or a constant) and an `OPERATOR`.

As I worked on the tokenizer and began planning the parser, it occurred to me that I needed a way to add metadata to certain things—enabling features I’d recently grown accustomed to from projects involving asynchronous code in more modern variants of C than I’d previously used. For several hours, I wrestled with how to do this: how to enable annotating code with extra metadata that was linguistically significant but syntactically less so. In the end, it was the act of thinking about it—the classic “Aha!” moment—that brought clarity. I was attempting to explain the issue to an AI assistant in a rubber-duck session when my mind connected the concept (“annotating”) to a term I knew all too well from many hours spent writing Java code: Annotations.

But while I now had a name and a good idea of what was needed, I was at a loss for the _how_. In Java and other languages that provide an annotation facility, you’ll often see stacks of annotations leading a method or member variable—this would not work with Turned-C. After all, I have personal experience with how one can get lost in those stacks and be blind to having left out a crucial annotation needed for the code to function correctly. I’d come to the conclusion that Turned-C should have a very clean, structured style, where the only things not explicitly inside some sort of “container” would be name definitions.

Sadly, none of my early attempts to actually apply annotations survive. A Windows crash—proving once again that my 'Provably Correct' theorem couldn't come soon enough—wiped the scratchpad. But the wait for the system to restart, no matter how short, gave me a chance to reflect, and two things struck me as the solution: 1) it must be a container of some sort, and 2) the “Whitespace Mechanic” meant I could craft a pair of “container” operators specifically for the task. I settled on `@[]@`—a pair of heavy-duty bookends to keep the metadata from bleeding into the logic.

A lack of foresight means I didn’t keep any of the early sketches around either, but I’ll try to reconstruct what I recall of the first idea here. This idea was actually for moving the `visibility` and `scope` from being “modifier keywords” in the `type` section of a name definition to being an annotation, and looked something like this:

```
@[ visibility: public, scope: world ]@
define string : structure <-> ( 
  @[ visibility: protected, scope: derived ]@
  _str: array character, 
  split : function <-> {
    ... do work here ...
  } ,
  ... add the rest of the API here ...
) ;
```

I’m sure you’ve got questions like “Where is the Genesis marker?” or “How can a name start with punctuation?” The answer to the first is simple: I hadn’t yet had the insight that the Genesis Marker was needed. But that second question has a subtly more complex answer, which I’ll try to answer as honestly as I can: I was being a bit foolish, letting the 'C-ghosts' in my head dictate the rules. I was trying to build a spaceship but kept reaching for a steering wheel from a 1994 sedan.

The next few days are a blur in my memory—I cannot accurately detail the order in which improvements came, though my notes do leave an interesting trail. Work on implementing a "reference compiler" for Turned-C had ground to a halt as I attempted to strip the language back to its bare minimum, so the "Toolkit Philosophy" could be fully exposed and exploited. This led me to lay out a few core functions to make sure I wasn’t stripping out something necessary for the philosophy to work. Those are `if`, `while`, and `for`—things that, in other languages, are pure keywords with special, syntactic meaning and semantics.

```
define while : function <-> (exit_condition: function boolean, core: function) <- {
  if ( [[ () <=> exit_condition ]] , { () <=> core }, { (exit_condition, core) <=> while });
};

define for : function <-> (init: function, exit: function boolean, inc: function, core: function) <- {
  () <=> init; // run once

  define for_int: function <-> () -> {
    if ( [[ () <=> exit ]], { 
      () <=> core; // run the loop code
      () <=> inc; // run the incrementor
      () <=> for_int; // recurse!
    });
  };

  () <=> for_int; // run the loop
};

define if : function <-> (condition: function boolean, truth: function, falsity: function) <- {
  [[ () <=> condition ]] ? { () <=> truth } || { () <=> falsity } ;
};
```

It may be obvious (or it may not), but the single most major decision I came to was that a `ternary` operator, of sorts, was necessary to implement any higher-level flow control. With fond memories of C and the way I often abused its ternaries to create complex-looking decision trees—yes, sometimes I actually write code specifically to troll others, because if a programmer can't have a little fun with a ternary tree, what's the point of the job? That is, however, beside the point—the decision was made, and the question mark (`?`) would act as a `choice` operator. Beyond that, however, came a seemingly simple choice: replicate what had come before and use the colon, or try to stay true to the "Single Meaning Philosophy"?

And note the recursive nature of `for` and `while`: this approach comes with a certain amount of risk, because I honestly don't know if the recursion will blow the stack in practice. At the time I decided to implement them as recursive, I added a restricted depth to that recursion. That 1024 limit was a bit of a reality check—a realization that I didn’t know enough about Tail-Call optimization to offer an infinite horizon. Since the compiler is still a work in progress and barely a tenth of the way through Pass 1, I figured I'd plant a flag at a 'reasonable' depth until I could actually measure the wreckage.

In the end, the choice was a very hard one, but simple in execution: Turned-C would not use the `:` to separate the two possible choices in a ternary—doing so would introduce a second, unconnected definition for the operator. Instead, I saw a better path forward: Turned-C already had a pair of combinatorial logic operators (`||` and `&&`), and one of them—the Logical OR operator—could have a second, closely related definition added. This decision certainly _stretches_ a core philosophy of Turned-C, but it does not violate it and, personally, I find the result rather beautiful.

That decision led me to another—though minimal, I had some experience with Kotlin and Groovy. All three languages, as far as I could remember, implemented a form of what is commonly termed the "Elvis Operator." This is a specialized ternary with only a single branch—the `false` branch—and is highly useful, if a bit specialized. My first instinct was to have it defined as the `choice` operator (`?`) immediately followed by the alternation (Logical Or) operator, just omitting the `true` branch. I quickly discarded the idea, deciding that it was an abuse of the already "stretched" Single Meaning Philosophy, and instead had a true "Eureka!" moment. The Logical AND operator can be defined as a short-circuit where the right-side operand is only evaluated if the left-side operand is `true`—though the resulting construct somewhat stretches things, this led to the birth of Turned-C's "Elvis Operator," structured as: `condition ? && result`.

It looks odd at first glance, but in the context of 'data motion,' it’s remarkably honest. It’s the linguistic equivalent of saying, "Check this, and if it’s a go, take the result." If the choice operator were omitted, there would be no clear indication that the result—a simple "AND"—was special in any way. On top of that, it provides a nice counterpoint to the dual meanings of its counterpart, presenting it as something of a "mirror" of its twin.

I was “on a high” and making great strides: I had a list of keywords and their basic definitions, a list of operators, some work done on defining them and their semantics properly, and a really firm grasp on just what I was building. Turned-C was changing from a classic “pie in the sky” dream—of the sort programmers throughout history have had—into a real, concrete entity. However, I got stuck. My mind kept throwing up ghosts of C as I tried to more strictly define things, and diving into my past thoughts on the language turned up a chunk of text talking about a “module and package” system that I had completely forgotten.

Those thoughts of C and its “wonderful” (sic) `#include` system gave me pause as they warred with much more fond memories of `import` and `use` statements in Python, Java, and similar, newer languages. It struck me—almost like a bolt of lightning—that while I did not (yet) have syntax to actually create and utilize the long-ago dreamed-of “module and package” system, that didn’t matter, as the direction I had been going didn’t allow for it anyway. The compiler was little more than a table of operators and the information needed for a Pratt Parser to work and the tokenizer, but it had been heading down the well-tread path of C and other languages: a single “Pass” over the code.

With how key the macro is to Turned-C, and how central the idea of the language “describing itself” was to the entire concept, I realized that a single pass would not work. What was needed was multiple passes, each with a well-defined role. That structuring would give Turned-C a means to regulate itself at a higher level than the tokenizer or parser as other languages used, and an even stronger set of “checks and balances” than things like the famed “Borrow Checker” of Rust. Unfortunately, the earliest notes on the pass structuring were kept in a single tab—one that was edited and re-edited as I worked to make sure the structuring was sane and allowed for things to actually work. What follows is the raw text from the current form of those notes, with nothing removed or added:

```
Pass 1 (Pratt/Parse): Structural Genesis. Generates the "Dumb Tree." It doesn't know what symbols mean; it just knows how they are shaped.
Pass 2 (Import/Module): Environmental Loading. This is where the NBT-based .tch files are read. The compiler populates the symbol table with external "Truths" before any logic runs.
Pass 2.5 (AST Rewrite): "Collapse" all `NODE_ANNOTATION` entries into their following `NODE_DEFINE` entries so the metadata is applied to the "defined name" in the symbol table/operator table/macro table.
   - Input: `NODE_ANNOTATION` + following `NODE_DEFINE`
   - Output: `NODE_DEFINE` with annotations attached to the embedded `NODE_IDENTIFIER` or `NODE_OPERATOR` in the correct table
   - Error: if there is not a valid `NODE_DEFINE` following the `NODE_ANNOTATION`
Pass 3 (Macro/Transform): The Expansion. Macros consume the AST nodes from Pass 1 and use the symbols from Pass 2 to return a new, "Sugared" tree.
   - Macros cannot emit new annotations -- annotations are resolved prior to macros being evaluated entirely.
Pass 4 (Semantic Validation): The Semantic Correctness Gate. This pass enforces semantic correctness (types, capabilities, legality constraints). In the broader language philosophy, these validated results are the substrate from which larger provably-correct constructions are composed.
Pass 5 (Metadata Generation): The Persistence. Encodes the exported symbols and macros into your NBT variant for future use by other modules.
Pass 6 (Emitter/Handoff): The Translation. Final hand-off to the LLVM backend. 
```

That is their current form—one that has not yet been spread to the existing documentation—but the “provisional” stage, Pass 2.5, shows that nobody gets things right on the first (or even fifth) try. Turned-C and its design is a living beast, being developed and specified in lock-step so that the language meets the requirements of its “Five Philosophies”—with the “Toolkit Philosophy” leading the charge. I do have to admit, however, that I am still unsure about having the AST Rewrite/Annotation Collapse as a separate “Pass”—something about it screams “wrongness” to me, while at the same time, I cannot quite define why. That means I have yet to find a way to justify removing it to myself, so it is hanging on in a sort of limbo, the Sword of Damocles hanging over its head.

With the basic structure of the multi-pass system in place, I was enthusiastic—I could finally figure out the syntax for the "import" system, and it had a proper "hook" in Pass 2. But that didn’t happen. In all truth, at the time of writing this document, a clear vision of how this concept fits into Turned-C’s syntax still eludes me. However, this is where the current, "idiomatic" form of Turned-C truly started to come together.

With some regret, I decided to put the import syntax on the back-burner and tackle other, more immediate needs of the language. The first of those was prompted by a gut feeling—other programmers know how well those always turn out—that the current syntax actually needed the parser to be a lot smarter than originally planned. Again, I turned to an AI for a rubber-duck session, hoping to exorcise that "demon" by giving it proper form and exposing it to the light. This gave the problem shape: the parser had to have something of a "brain" to understand that a `BAREWORD` token in a specific placement in the structure was "introducing a new symbol." That alone meant the parser had to be a lot smarter than the design called for—it would actually have to act as more than a "collector and categorizer" running off its tables—and this was what my instincts had been screaming about.

It took several hours of rubber-ducking with the AI and talking to a few of the friends who had been instrumental in helping me evolve Turned-C through its pre-history and early-modern phases, but a solution was found. A simple solution: the introduction of a new operator to mark the `BAREWORD` as having special meaning. That solution took more time to refine and decide on both the form it would take—eventually the `octothorphe` (or `hashtag`, for those who haven't spent their lives reading typography manuals) was chosen, and the AI that had rubber-ducked for me gave it the perfect name: The Genesis Marker.

That, alone, didn’t lead to any new snippets of possible code being written, but it proved a turning point. After that, I set to work trying to design what I was still calling `switch` as a macro and realized that—surprise—I had actually forgotten something important. I had managed to completely forget something that I think is one of the most important features available in any programming language, classic or modern: Variadic Functions. That realization felt like a stranger had walked up to me and slapped me in the face—here I was, toiling away at developing "the perfect programmer’s toolkit" (yes, a bit conceited, but I am only human), and I had forgotten something that makes several advanced forms of programming and meta-programming possible.

Once again, I set to work using AI as a rubber-duck—by this point I was deep in the 'compiler trance,' the kind where you forget to eat and start seeing AST nodes in your sleep—and I didn't want to drag my friends into the storm just yet. The syntax I was used to from my long years of working in C didn’t feel like it was a good fit for Turned-C—it was sterile and underspecified, with all kinds of "implementation defined" and "platform defined" bits. After a comparatively short period of time, the solution to this issue was also found. Trust me, I know you’re probably sick of me saying it, but it was actually simplicity that won out—both in notation and implementation. The notation is simple: you add an ellipsis (`...`—three actual periods, not the Unicode character) after a "name" in the last place of the "parameter list" to indicate it is to "collect" all entries in the list from that point on. Applying the decision that this should also apply to a macro—since those are specified in Turned-C with no real difference in syntax—was what some might call a "no-brainer."

In the end, this led to me putting this in my notes, along with the decision that `switch` really wasn’t the best name and that more modern languages had made a perfect choice with `match`:

```
// The recursive macro that builds a chain of 'if' logic
define #match : macro <-> (subject: node, arms...) -> {
    // Base Case: No more arms to match. 
    // We return a "nothing" block or an error thunk.
    [[ arms.count == 0 ]] ? `() || {
        
        // Split the current arm from the remaining arms
        define current <-> arms.head;  // A structure like ( [[ it == 0 ]], { "Zero" } )
        define rest    <-> arms.tail;
        
        // Peel the predicate and the block out of the current arm
        define predicate <-> current.head;
        define body      <-> current.tail;
        
        // Return a QUOTED ternary that "short-circuits" or recurses
        `{
            // 1. Inject the subject into the 'it' context for the predicate
            define it <-> (subject);
            
            // 2. Evaluate the guard. 
            // If it passes, run the body. 
            // If it fails, recursively call 'match' with the remaining arms.
            (predicate) ? (body) <=> eval || (subject, rest) <=> match ;
        } ;
    } ;
} ;
```

I will warn you: the syntax this displays still has not reached current standards. There was a chunk of "implicit magic" in the format of macros at this point in Turned-C’s development, though that would not last long. As you might have noticed from the rest of this document, I dislike "magic" in programming languages and would, instead, prefer a clean, simple, and—above all—analyzable language.

All the while I’d been working on things, I’d thought of operators as simple creatures—and my syntax for defining them reflected that. At best, I could tie them to a backend primitive using a “magic annotation.” That syntax (shared below for posterity) soon felt confining, as I realized the type system I was building was, well, _too_ rigid. It was starting to feel like COBOL or Pascal—where even integers of different storage sizes couldn’t play together.

```
@[ target: LLVM::i64, signed: true ]@
define #i64 : type ;

@[ target: LLVM::sdiv ]@
define # / : operator i64 <-> (left: i64, right: i64) ;
```


Can you feel the uselessness of that structure? The sheer inability to allow for things like “type promotion”? I sure did—to this day, I want to reach back in time and smack myself for not seeing it sooner. That sense of wrongness (and guilt) led to a revelation: even though I dislike the name-mangling required by library and executable formats, overloaded operators and methods are essential for modern metaprogramming. This sent me down a rabbit-hole of ideation, leading to my first attempt at a fix: subtly “abusing” the `index` operator (`[]`) to hold a related meaning—just as I’d previously “abused” `logical or` and `logical and` to implement the ternary and Elvis operators idiomatically:

```
// define an overloaded operator, a rough sketch
define # / : operator <-> [
    @[ target: LLVM::sdiv ]@
    (lhs: integer signed, rhs: integer signed) : integer signed ;
    @[ target: LLVM::udiv ]@
    (lhs: integer unsigned, rhs: integer unsigned) : integer unsigned ;
    @[ target: LLVM::fdiv ]@
    (lhs: float, rhs: float) : float ;
] ;
```


I’ll admit, in sketching that out, I was deliberately side-stepping “type promotion” and a few other issues, and I left out a series of “magic annotations” that would soon vanish from Turned-C forever. Still, I thought it was an excellent idea: it let me describe an operator as an immutable, “sealed set” of related functionality—distinctively Turned-C in both structure and syntax.


Then I moved on—a good friend pointed out that I was relying on “magic unquoting” in macros. That meant I was expecting a lot of unnecessary work from the AST Tree Walker. His comment triggered hours of self-recrimination and reflection, but eventually, sanity prevailed. Rather than any form of “magic” or “implied” unquoting in macros, what was needed was an operator—preferably a grouping operator—to specifically notate the operation.


I really stressed over it, even pinging two different AI models for rubber-ducking before the solution finally appeared. And yes, the answer was simple: the “maximal munch” of the tokenizer pointed the way—why not a “compound operator” like the one I’d designed for annotations? With that decided, it became a race to find a compound operator that could stand on its own and, hopefully, not cause confusion. In the end, pragmatism won out over “total clarity and lack of confusion,” and I defined `unquote group` as the pairing of `{(` and `)}`.

That solution, to this day, feels like a major cop-out. But the only other idea that was in the running—`$[` and `]$`—risked confusion on _at_ _least_ three fronts: one from users of the Bourne-shell and its descendants (where it marks “transclusion”), one from Perl programmers who see `$` as marking a variable as a `scalar`, and at least one more, from inside Turned-C itself, where `[]` already has three related but distinct roles. So the “compromise” between clarity of meaning and _minimal_ confusability stands as the best choice: a load-bearing compromise—ugly, but necessary to keep the roof from falling in on the Perl and Shell scripters.


This left me at something of a loose end; I’d just solved a major mystery, but I was still stuck on the problem of the `import` system and its syntax. It was sometime after 10PM, while I was working on how some of the operators should work, that the next major flash of inspiration struck. This took two forms: first, I realized Turned-C was missing an important part of its “contract”; second, I had, inevitably, gotten things backwards. The latter felt more urgent, so I tackled it first—even though, in hindsight, it wasn’t.

That realization—that I had things backwards—was, in fact, the biggest revelation. Instead of the `operator` being its own self-contained piece, it was actually a “dispatch center”. Some were closed to new subscriptions for semantic safety, while others could be subscribed to at any time. This led to several ideas, culminating in this example from my notes:
```
// 1. The Global Sockets
define # + : operator ;
define # - : operator ;
define # * : operator ;
define # / : operator ;
define # == : operator ;

// 2. The Type-Nested Implementations
@[ target: LLVM::i64, signed: true ]@
define #i64 : type <-> (
  ` + : operator intrinsic <-> [ @[ target: LLVM::add ]@ (l: i64, r: i64) : i64 ; ] ,
  ` / : operator intrinsic <-> [ @[ target: LLVM::sdiv ]@ (l: i64, r: i64) : i64 ; ] ,
  ...
) ;

@[ constructable, destructable ]@
define #string : type <-> (
    #store : grapheme array,
    #length : Integer,
    #init : function <-> () -> { ... } ,
    #done : function <-> () -> { ... } ,
    ` + : operator intrinsic <-> [ (l: string, r: string) : string -> { ... do concatenation ...} ,
                                   (l: Character, r: string) : string -> { ... append the string to the character ... },
                                   (l: string, r: Character) : string -> { ... same as the above ... } ] ,
    #slice : function string <-> (#input : string, #start : Integer, #length : Integer ) : string -> { ... } ,
    ...
) ;
```


When I jotted that down, I still wasn’t sure of the final notation, but it shows the clear growth of Turned-C’s idiomatic style as it continued to evolve. I’d come up with a notation that fit the “feel” of Turned-C, making the operator something defined by the type—capable not just of referencing a backend intrinsic, but of having its whole operation encoded directly in Turned-C itself. This was a triumph, but it also showed the syntax still had some growing to do. For instance, did you notice the introduction of the `intrinsic` keyword? It was needed to specially flag things so items like `operator dispatch` could be hooked up, and to mark something as a “key part” of the type. That latter meaning came a few days after this sketch—along with the decision to change the destructor name from `done` to `fini` (an almost artistic choice, in my opinion—it fits much better alongside `init`).


But the more important realization was that Turned-C had numerous integer and floating-point types, yet the design gave no way for them to interact—not even between integers of different storage sizes. Want to add a 16-bit integer to a 64-bit one? It’s possible, but only with unchanging, fixed, and—worst of all—not analyzable or testable logic. I decided to fix this—‘magic’ is for fantasy novels, not programming languages!


With that in mind, I decided what was needed was a “clearing house” for types—much like how `operator` had been redefined as a “distribution center”. The idea solidified quickly (with a bit of AI rubber-ducking), and I named it the `Type-Class`. These are, essentially, “contracts” that define how members interact, what sorts of “intrinsic” values they must provide, and—most importantly—which operators they must support. The exact syntax is still evolving: it’s a complex topic, and I’m just one programmer with a few friends and some AIs I occasionally bother for rubber-ducking and editorial help. Still, these snippets cover the very first iterations of the concept, showing off the basic premise:

```
define #Integer : type-class <-> [ `i64, `u64, `i32, ... ] ;
[ `u128 ] -> Integer ;
```
and:
```
define #Integer : type-class <-> [
    // The "Axiomatic Width Rule" defined at the Class level
    `promote : function intrinsic <-> [
        (lhs: Integer, rhs: Integer) : type <-> {
            ? (lhs.bits > rhs.bits) && (lhs)
            || (rhs.bits > lhs.bits) && (rhs)
            || (lhs.signed != rhs.signed) && (unsigned) // The "Symmetry Break"
            || (lhs) ; // Default to LHS if identical
        } ;
    ] ;
] ;
```


These are presented as two separate entities because that’s how they appear in my notes. The first is my initial idea for how they might work; the second is a draft for how type-promotion could be specified. Neither is final—this entire section of Turned-C syntax is, at the time of writing, very much alive, in flux, and evolving. My hope in sharing this peek into the “birth” of the Type-Class is that you’ll be inspired by it—and see how the language is still evolving.

That actually brings me to within the past week, where I’ve refined a lot of ideas and managed to remove a wide swath of “magic annotations” in favor of explicit data. For now, this has been limited to how `types` define things—but I have hope! I hope that, soon, I’ll have the inspiration needed to extend this approach to operator overloads and to the syntax for a Type-Class to specify its “membership contract.”

In the meantime, I leave you with this final example from my notes, showing a much more modern form for a type definition—one that leaves behind all the “magic” in favor of explicit values:

```
define #u64 : type <-> [
  `bits : variable intrinsic <- 64 ;
  `backend-type : variable intrinsic <- $`LLVM::i64`$ ; // provisional idea—the $` .. `$ operator pairing for marking a “foreign identifier”
  `signed : boolean intrinsic <- false ;
] ;
```
