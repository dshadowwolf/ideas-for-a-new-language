Turned-C is built as a minimalist toolkit where most language features are implemented as functions within the language itself. However, a small set of Keywords is required to define the irreducible "Physical Laws" of a program. These are categorized by their impact on the compiler and the machine:

   1. annotations: Metadata that does not occupy space in the final binary, but instructs the compiler how to generate code or what rules to enforce.
      * The "Post-It Note": Attached to a definition to modify its translation.
   2. types: Physical blueprints for memory. They define storage size and how the CPU should interpret the resulting bits.
      * The "Data Shape": Defines the dimensions of the container.
   3. capabilities: Qualifiers that restrict or enable actions on an identity. They do not change bit-width, but they dictate the validity of operations and lifecycle rules.
      * The "Permissions": Defines the lifecycle and mutability of the container.
   4. meta-capabilities: High-level classifications that determine a symbol's Fundamental Role. They exist between Types and Capabilities to define the "Identity" of a name.
      * The "Job Description": Is this a math rule (`operator`), a storage bin (`variable`), or a recipe (`function`)?
   5. other: Hard-coded primitives necessary to bootstrap the language's grammar and structural logic.
      * The "Glue": Without these, the parser could not identify definitions or navigate core control flow.

annotations:
  - atomic         : basic `seq_cst` guarantee with no specific memory order guaranteed
    - use on a `NODE_FLOW` and this triggers an `atmicrmw` instruction in LLVM
  - sync           : guarantees `acq_rel` memory ordering as an atomic
    - except on a `bind` where this acts as `release` (e.g. `x <- 10`)
    - except on a `deref` where this acs as `acquire` (e.g. `*x`)
  - relaxed        : works like `memory_order_relaxed` in C11/C17
  - precedence     : for defining operator "precedence"
    - should be in the 40-60 area so as to not strongly conflict with anything
    - value actually spans 0-100 and any level in there can be used
  - fixity         : for defining operator "fixity" -- can be "infix", "prefix", "postfix" or "any"
    - "any" means the operator may define one or more of "infix", "prefix" or "postfix" as separate parse forms
  - association    : for defining operator "fix" -- can be "left", "right" or "none"
  - foreign        : not defined in Turned-C but in another language
  - visibility     : for "OO" use, this defines item "visibility" from among a large set of choices, this is then modified by the `scope` value
    - `class`      : similar to `private` in C++ or other "classical" OO languages
    - `subclass`   : similar to `protected` in C++ or other "classical" OO languages
    - `public`     : classic meaning
    - `private`    : classic meaning, but for areas where there is "greater than class visibility" via the `scope`
    - `protected`  : classic meaning, but for areas where there is "greater than class visibility" via the `scope`
  - scope          : for "OO" use, this defines the visbility of an entire section and not just one part
    - `class`        : using this brings the full, classic meaning to `public`, `private` and `protected`
    - `package-only` : restricted to access by the containing "package"
    - `package`      : the containing package and any sub-packages
    - `module-only`  : a "module-level" variation of `package-only`
    - `module`       : a "module-level" variation of `package`
    - `assembly`     : the containing "assembly"/"library"/"program" but not beyond that level
    - `world`        : "everything" scope
  - capture        : for "execution environment capturing" to help "reparenting" actually work
    - `context`    : the "implicit default" -- this captures the whole of the "parent environment" (which _might_ be the "global scope")
      - implicitly this just "injects" the captured values into the scope as needed and defined by the `Closure, Captures and Scope` document
      - explicitly this "injects" a `context` "object" that contains the values needed to be referenced. This can "short circuit" the resolution process detailed in the design
    - "name"       : specifically capture a given "name" from the parent environment
  - requires       : this is a "counterpart" to `capture` which gives extremely fine-grained control over the process
    - `capture`    : the only 'value' accepted at the moment, this specifically says "capturing parts of the defining environment is necessary"
  - required-items : an array of `name: type` pairs giving the "name" and "required type" information of what to capture from the defining environment
types:
  - integer        : "largest" integer storage - equal to `i64`
    - i8, i16, i32, i64 : integer types guaranteed to be that many bits wide
  - float          : "largest" floating point storage - should be equal to `f128`
    - f32, f64, f128 : floating point storage matching IEEE specs for floats of the given bit-width
  - character      : a single byte of storage representing a classic "ASCII Plane" utf-8 character
  - code_point     : a "rune" -- a single UCS-2 "code point"
  - grapheme       : proper storage for a single, human-percieved "character" -- this might be an emoji with a skin-tone modifier or a complex bit of Devanagari
  - void           : A primitive with dual semantics: either the absence of value or the omission/erasure of type information
    - As a Return Type: A signal that a function is executed solely for its Side Effects and does not associate a resulting value with the caller's scope.
    - As a Pointer Target (`pointer void`): This is the "Universal Opaque Handle". It denotes a memory address where the "Storage Shape" is intentionally erased, allowing for the transport of raw data across type-safe boundaries (essential for `malloc` and `foreign` C-API interaction)
    - Think of this as the "Universal Socket". It's either an empty plug (nothing) or a plug that fits any socket (anything).
  - boolean        : classic true/false
  - pointer        : A type modifier that transforms the associated type into an address-bearing identity. Rather than storing a value directly, it stores a "Fat Pointer" containing the memory address, bounds, and capability metadata of the target. (think of this as a "Look Over There" instruction -- it doesn't describe the data itself, but the path to it and the permissions allowed upon arrival)
  - structure      : A meta-type used to define heterogeneous data aggregates. It acts as a blueprint for a named scope where multiple identities (Variables, Functions or other Structures) are packed into a single, contiguous block of memory with deterministic offsets. (think of this as a generic "Container" -- it defines a private universe of names that can be instantiated as a single unit of data or an `object`)
capabilities:
  - mutable        : A capability that explicitly permits an identity to undergo state changes after its initial association. In `Turned-C` all names are _Immutable by Default_ (Write-Once); the `mutable` keyword is the only mechanism that enables the "Flow Operator" (`<-`) to modify a value.
    - Applied to Variables:
      - Behavior: Signals that the associated memory is "mutable" and may be overwritten.
      - Optimization: Prevents the compiler from performing "Constant Folding" or "Global Value Numbering" on the name, ensuring the CPU always performs a fresh `load` from memory.
    - Applied to Functions:
      - Behavior: Identifies a "Non-Pure Function". It informs the compiler that the function interacts with the "Outside World" (e.g. I/O, Global State or Hardware)
      - Optimization: Disables Memoization. The compiler is forbidden from eliding subsequent calls to the function even if the input parameters are identical, as the "Side Effect" is a required part of the program's execution.
  - static         : this is a "lifetime" marker and a "storage" marker 
    - `static` data is "initialized" before the entry point gets hit and lives in the data segment instead of on the stack.
    - something flagged `static` and `constructable` has the `constructor` called to initialize it at the same time all `static` items are initialized
    - `static` items that are `destructable` have a lifetime matching that of the program itself and will have the "destructor" called on program exit
  - constructable  : this `name` will define a "default constructor" to build new, default versions of itself
  - destructable   : this `name` has a defined "default destructor" to tear down copies of itself when they have left the scope
meta-capabilities:
  - operator       : defines that a given "symbol" is an operator -- precedence and other features are given as an annotation -- while the associated "parameters" and "code" are what actually executes.
  - function       : defines that a given "symbol" is going to be associated with code instead of "data" -- this is mostly a "compiler hint"
  - variable       : defines that a given "symbol" is only ever going to hold data and not code
  - object         : defines that a given "symbol" is host to both data and code in a structured form
other keywords:
  - define         : literally "define" a name in the symbol table
  - DEFAULT        : A universal, pre-defined predicate lambda that always evaluates to `true`.
    - The "Catch-All" Rule: It is designed to capture any input not intercepted by a preceding specialized predicate.
    - The "Terminal" Constraint: Because it is "unconditionally greedy", it should be placed as the final branch of a `switch` or matching block. Placing it earlier will "short-circuit" the logic, causing all subsequent branches to be ignored (unreachable code).
  - `true`         : exactly what it says -- "boolean true"
  - `false`        : the complement of `true`
