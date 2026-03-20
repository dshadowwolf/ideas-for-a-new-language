## On the Most Minimal Form of Turned-C Possible
### An Informal Introduction

The author’s name is less important than the reason this document exists. About a decade ago (as of March 19, 2026), I set out to design a new programming language—one that would be simple at its heart, and extensible from within, using its own mechanisms. Over the years, that vision grew more complex, and I lost sight of the original idea: keep the core minimal, containing only what is needed for the language to extend itself through macros and libraries.

On a whim, I recently revisited some of my earliest notes. Rereading an old essay, I was struck by how far I’d drifted from that core principle. By stripping away the layers from the “great beast” I’d been building—`Turned-C`—and returning to its most essential pieces, I realized I was finally achieving the goal I’d set for myself a decade earlier.

The words that reignited this project are quoted below:
> The key points of this language are that the language itself is only going to provide the features that are needed for all required parts of the language itself to be implemented, with a few additions that are going to be there for the module writers and to assist in implementing the default runtime library. As part of keeping the core language as simple as possible, most of the syntactic sugar is actually going to be implemented in the full-featured macro-language that is defined as a companion – so that this language has a whole second level of utility.

### The "Bootstrap Core" Language Definition

#### An Overview

Turned-C is designed to be self-describing: the language should be able to define, extend, and eventually reconstitute itself using only its own native mechanisms. That goal imposes a hard lower bound on what must exist before any higher-level feature can be expressed.

This document defines that lower bound.

The Bootstrap Core is the smallest coherent subset of Turned-C that is sufficient to:
- create names and bind them to meaning
- describe structure and execution
- express control flow
- attach the metadata required to connect language-level definitions to backend primitives

Everything outside this subset may be useful, expressive, or eventually standard, but it is not required for the language to begin defining itself.

The purpose of this document is therefore not to describe Turned-C in its richest form, but in its most irreducible one: the minimal set of operators, keywords, and annotations required for the language to bootstrap from nothing into a functional, extensible system.

#### The Core Set of Features
##### The Operators

Only a small number of operators are strictly required for Turned-C to become self-hosting in principle. These operators form the structural machinery of the language: they create names, bind values, shape execution, and direct control flow.

For clarity, the Bootstrap Core operators are divided into three functional sets:
   1) The Symbolic Core:
      - `<-` (`Flow`)
      - `<->` (`Associate`)
      - `` ` `` (`Quote`/`Quasiquote`)
      - `#` (`Name`/`Genesis Marker`)
   2) The Structural Core:
      - `( ... )` (`Structured Data`)
      - `{ ... }` (`Function`)
      - `@[ ...]@` (`Definition Annotation`)
      - `,` (`Gather`/`List Generator`)
      - `;` (`Terminate`/`Separate`)
      - `->` (`Bind`)
      - `<=>` (`Immediate Call`)
      - `.` (`Member Access`)
      - `[ ... ]` (`Index`)
      - `*` (`Derefernce`/`Unary Prefix *`)
      - `&` (`Address-of`/`Unary Prefix &`)
      - `...` (`Postfix Ellipsis`/`Variadic Capture`)
      - `{( ... )}` (`Unquote`)
   3) The Control Core:
      - `[[ ... ]]` (`Predicate Lambda`)
      - `?` (`Alternation`/`Select`)
      - `&&` (`Logical AND`/`Guard`)
      - `||` (`Logical OR`/`Choice`)

##### The Keywords

The Bootstrap Core requires a similarly small keyword set. Each keyword exists because it contributes essential semantic identity to the system: type formation, symbolic definition, operator registration, execution environment setup, or backend linkage metadata.

As with the operators, these keywords are grouped by role rather than by surface syntax. The goal is not to maximize convenience, but to identify the smallest vocabulary necessary for the language to define its own executable and extensible form.

The Bootstrap Core keywords are divided into the following sets:
1) Symbolic Core:
   - `define` (add a "name" to the symbol table, operator table or macro table)
2) Value Core:
   - `pointer`
   - `array`
   - `boolean`
   - `macro`
   - `type`
   - `operator`
   - `annotation`
   - `function`
   - `true`
   - `false`
   - `void`
   - `mutable`
   - `capability`
   - `node`
3) Execution/Environment Core:
   - `ENTRY`
   - `INITIALIZE`
   - `EXIT`
4) Metadata Core:
   - `internal`
   - `target`
   - `link_name`
   - `foreign`
   - `signed`

#### Definitions and Reason for Inclusion

Each feature in the Bootstrap Core exists because removing it would eliminate some essential capability required for self-definition. The purpose of this section is to state, for each operator, keyword, and annotation, what it does in the minimal language and why it cannot be deferred to a later stage.

##### Operators
 1) The Symbolic Core:
   - `<-` (`Flow`)
     - Definition: A right-associative transfer operator that writes a resolved RHS value into a named mutable destination on the LHS.
     - Reason: Without Flow, symbols can be declared and associated but never updated; mutable state and incremental computation become impossible.
   - `<->` (`Associate`)
     - Definition: The definitive binding operator that marries a declared name and its type-specification to a concrete value, structure or logic block.
     - Reason: This is the "Marriage Contract" of the symbol table; without it, names remain "pending" and can never be resolved into usable entities.
   - `` ` `` (`Quote`/`Quasiquote`)
     - Definition: An escape operator that suppresses the standard evaluation of a token, returning its raw symbolic identity instead of its bound value.
     - Reason: Essential for metaprogramming and bootstrap definitions; it allows the language to talk _about_ its own symbols (like defining an operator) without accidentally _triggering_ them.
   - `#` (`Name`/`Genesis Marker`)
     - Definition: A prefix operator that asserts the creation of a brand-new, unique entry in the current symbol table scope.
     - Reason: This is the "Birth Certificate" of a symbol. It provides a hard boundary for the compiler to prevent accidental shadowing or collisions during the bootstrap phase.
 2) The Structural Core:
   - `( ... )` (`Structured Data`)
     - Definition: A container that promotes a flat list of expressions into a unified, named or anonymous data aggregate (a `NODE_STRUCTURED`).
     - Reason: The internal offsets and memory layouts of structures are handled by the Emitter (Pass 6); this complexity must be built-in to ensure the refinery can handle complex data.
   - `{ ... }` (`Function`)
     - Definition: A scope-limiting container that encapsulates a sequence of statements into a deferred execution block (a "thunk").
     - Reason: Functions require stack frame management and lexical isolation — low-level machine behaviors that cannot be synthesized from simpler parts.
   - `@[ ...]@` (`Definition Annotation`)
     - Definition: Attach persistent, name-level metadata that changes the semantic interpretation and backend binding of the defined identity.
     - Reason: Without this annotation and a very small sub-set of pre-defined annotations there would be no way to interact with the code-generation backend, functionality defined in libraries written in other languages or interface with the host OS itself.
   - `,` (`Gather`/`List Generator`)
     - Definition: A low-precedence aggregator that chains independent expressions into a sequential list (`NODE_LIST`).
     - Reason: Without the comma, there is no way to provide multiple arguments to a function or multiple members to a structure; it is the "glue" of the aggregate system.
   - `;` (`Terminate`/`Separate`)
     - Definition: The zero-precedence hard stop that delimits the end of a discrete expression or statement.
     - Reason: It is the "Commit" command for the Pratt Parser; without it, the recursion of the refinery has no deterministic way to unwind.
   - `->` (`Bind`)
     - Definition: The injection operator that maps a structural context (the "Template") to a deferred block (the "Logic").
     - Reason: This is the "Wiring" of a function; it defines how inputs are piped into logic. Without it, you have code and data, but no way to connect them.
   - `<=>` (`Immediate Call`)
     - Definition: The activation operator that immediately executes a logic block using the provided left-hand context as its environment.
     - Reason: Every language needs a "Go" button. This is the only way to trigger the Emitter to generate a physical call or jump instruction.
   - `.` (`Member Access`)
     - Definition: The navigation operator used to select a specific named member from a structured aggregate or module.
     - Reason: Essential for modularity; without it, every symbol would have to exist in a flat, global namespace, making modern software architecture impossible.
   - `*` (`Dereference`/`Unary Prefix *`)
     - Definition: A prefix operator that accesses the value stored at the memory location held by a pointer.
     - Reason: This is the Indirection Exit. Without it, a pointer is a dead link: the address can be carried, but the referenced value can never be read or mutated.
   - `&` (`Address-of`/`Unary Prefix &`)
     - Definition: A prefix operator that retrieves the memory location of a specific identity, yielding a pointer to it.
     - Reason: This is the Genesis of Indirection. Without it, pointer types can exist, but no pointer can be formed from an existing identity.
   - `[ ... ]` (`Index`)
     - Definition: A bracketed indexing operator used for positional access into an array or for scaled offset access through a pointer.
     - Reason: While `*` enables a single step of indirection, `[]` enables structured traversal of contiguous storage. Since `array` is part of the Bootstrap Core, the language also requires a core means of indexed access.
     - Overload-Frame Form: A sealed container that wraps the overload signatures for a given operator or function name.
     - Reason: This enables backend-specific dispatch binding, where different operand type signatures map to distinct generated instructions.
   - `...` (`Postfix Ellipsis`/`Variadic Capture`)
     - Definition: A postfix operator on a parameter identity (`name...`) that captures that argument position and all following arguments into a traversable list.
     - Reason: Without postfix ellipsis, the Bootstrap Core cannot define variadic macros or variadic functions in its own syntax, which blocks self-definition of core expandable constructs.
   - `{( ... )}`
     - Definition: The AST Injection operator converts a symbolic node reference into a concrete AST expansion result, allowing the enclosing macro to return that fragment as syntax.
     - Reason: This is required for selective substitution; without it, macros could construct new syntax but could not re-emit chosen input nodes directly into the call site.
 3) The Control Core:
   - `[[ ... ]]` (`Predicate Lambda`)
     - Definition: A strictly boolean result lambda function.
     - Reason: This is needed to simplify logic constructs and make implementing functionality such as `if` and `match` possible.
   - `?` (`Alternation`/`Select`)
     - Definition: The structural anchor that initiates a conditional branch based on a boolean predicate.
     - Reason: This is the "Switch" in the refinery; it creates the diversion in the pipe that allows the language to actually make decisions.
   - `&&` (`Logical AND`/`Guard`)
     - Definition: A short-circuiting conjunction operator; in logical form (`a && b`) it evaluates RHS only if LHS is `true`, and in alternation form (`condition ? && expr`) it acts as a guard branch that consumes the active alternation context.
     - Reason: Provides both boolean conjunction and structural gating; required to express safe guarded branches in the bootstrap control model.
   - `||` (`Logical OR`/`Choice`)
     - Definition: A short-circuiting disjunction operator; in logical form (`a || b`) it evaluates RHS only if LHS is `false`, and in alternation form (`condition ? lhs || rhs`) it acts as the branch-choice split between true-path and false-path expressions.
     - Reason: Provides fallback and branch selection; required to express minimal `if/else`-equivalent behavior in the bootstrap core.
  
##### Keywords
1) Symbolic Core:
   - `define`
     - Definition: Add a "name" to the symbol table, operator table or macro table.
     - Reason: Direct interaction with the symbol table and the semantics of how `Definition Annotations` work mean this must be a part of the `Bootstrap Core`
2) Value Core:
   - `pointer`
     - Definition: A type modifier representing a raw memory address of a specific underlying type.
     - Reason: Pointers are the "Axioms" of memory. The bootstrap requires them to handle the self-hosting parser's own data structures (like NBT tags).
   - `array`
     - Definition: A type modifier representing a contiguous sequence of elements with associated bounds metadata.
     - Reason: Required for buffer management and string handling; it represents a higher level of memory safety than a raw pointer.
   - `boolean`
     - Definition: `true` or `false`
     - Reason: Without this base type all other logic becomes impossible.
   - `macro`
     - Definition: A compile-time transformer that operates on raw AST nodes before they are lowered to IR.
     - Reason: Macros are how Turned-C "sugars" itself. If they weren't in the core, the compiler couldn't expand its own language extensions.
   - `type`
     - Definition: The keyword used to define a new classification of data or a structural contract.
     - Reason: Without type, the refinery has no way to validate its own inputs; it is the basis of the entire "Provably Correct" system.
   - `operator`
     - Definition: A meta-capability that allows a symbol to be inserted into the Pratt Parser's dispatch table.
     - Reason: Without this, the language's grammar would be fixed forever; this allows the bootstrap to define its own mathematical and logical symbols.
   - `annotation`
     - Definition: The meta-capability that defines a symbol as a bearer of metadata for other identities.
     - Reason: To avoid a fixed set of "magic" tags, the language must be able to define its own metadata categories to guide the emitter and optimizer.
   - `function`
     - Definition: The meta-capability denoting a symbol as an executable mapping between an input context and a result.
     - Reason: The fundamental unit of logic; it must be in the core to allow the bootstrap to perform any work at all.
   - `true`/`false`
     - Definition: The two discrete literal states of the boolean primitive.
     - Reason: They are the "1" and "0" of the logic core; without these constants, you have a boolean type with no values.
   - `void`
     - Definition: A type keyword denoting that an expression, function, or callable interface yields no value.
     - Reason: The Bootstrap Core requires `void` to represent effect-only operations and to bind cleanly to backend or foreign procedures whose result type is explicitly “no value.”
   - `mutable`
     - Definition: A qualifier that marks a defined identity as eligible for later value transfer or reassignment.
     - Reason: Since `<-` is part of the Bootstrap Core, the language must also provide a core mechanism for declaring which identities may be updated after definition.
   - `capability`
     - Definition: A keyword for defining named behavioral constraints/permissions that can be attached to identities and validated by the compiler.
     - Reason: While the Bootstrap Core only requires `mutable` immediately, it must also be able to define additional capabilities (for example lifecycle-related traits such as `static`) without hardcoding each one into the core grammar.
   - `node`
     - Definition: A compiler-recognized meta-capability representing a syntax-tree entity in abstract form.
     - Reason: Since bootstrap macros operate on AST structure, `node` is required to express their argument and result boundaries explicitly.
3) Execution/Environment Core:
   - All three of these quite literally control the entire lifecycle of a package, module or program and there is no way to denote these from within the `Bootstrap Core` itself as they represent conceptual "execution points" that the language itself has no other way to represent.
   - `ENTRY`
     - Definition: This should only ever truly appear on a standalone program -- this marks the core "execution loop" of the program, it is where the OS "jumps" to "start execution" of the code.
   - `INITIALIZE`
     - Definition: Functionally this is identical in concept to the `C++` "static constructor/initializer" segment(s) in that it serves as a way for a package, module or program to have setup and other work done before control is "handed off" to the code defined in a programs `ENTRY` block/method. For a `package` or `module` this can be thought of as being run `at library open`/`at library load`.
   - `EXIT`
     - Definition: This operates in a manner similar to a `C++` `atexit()` or `static destructor` in that it is code to be run at program exit, package/module unload to release held resources and generally do "cleanup work", especially if things were allocated/acquired during the execution of an `INITIALIZE` method. These happen after the `ENTRY` method has exited but before control is given back to the host OS.
4) Metadata Core:
   - `internal`
     - Definition: A visibility modifier that restricts a symbol's access to the current package/build-unit and excludes it from the public .tch header.
     - Reason: This is the "Security Valve" for the bootstrap; it allows the compiler to share deep parser hooks between modules while ensuring they never leak to user code.
   - `target`
     - Definition: The symbol or backend method/type "targeted" by the defined name/operator -- this allows for type and method aliases, references to backend defined types and references to backend defined methods.
     - Reason: Without this we would have to have fixed definitions for a lot more types and operators -- with this we can, for instance, "hook" LLVM backend methods to implement the `+` operator, the various `integer` types, etc...
   - `link_name`
     - Definition: The actual _name_ of a foreign symbol.
     - Reason: This allows for referencing system libraries and making system calls regardless of what language they were defined in and being able to give them proper names in `Turned-C`
   - `foreign`
     - Definition: The name so annotated actually references something defined in another language.
     - Reason: Together with `link_name` this allows `Turned-C` to reference code written in other languages with other calling conventions and potentially give those functions proper names according to `Turned-C` naming conventions and identifier rules.
   - `signed`
     - Definition: A boolean selector that informs the code generator how to interpret a signless scalar type.
     - Reason: Without this, backend-provided scalar types that do not inherently encode signedness could not be given portable signed or unsigned semantics within the Bootstrap Core.

#### Ideas and Sketches for Feature Implementation

This section collects implementation-oriented notes showing how the Bootstrap Core may be realized in practice. These sketches do not override the normative definition of the core; they exist to illustrate plausible parsing, semantic, and backend strategies for bringing the minimal language into existence.

##### Notes

- The lifecycle markers `ENTRY`, `INITIALIZE`, and `EXIT` are included in the Bootstrap Core because they are irreducible parts of Turned-C's execution model; however, their full runtime ordering and host-environment semantics are intentionally specified elsewhere.
- The examples in this section are illustrative rather than normative. They are intended to clarify implementation direction, not to serve as final or authoritative reference forms.
- The following sketches assume an LLVM-oriented bootstrap backend, since that is the primary intended code-generation target for the minimal implementation.
- Some control-flow sketches are intentionally schematic and may omit finalized surface syntax; operator/keyword definitions above remain normative.

#### Examples of Feature Implementation
##### Type Specification

The following examples demonstrate how `type` definitions and type aliases may be constructed within the Bootstrap Core by binding language-level identities to backend-provided primitives.

1) Definition of an Integer Type:
   ```turned-c
   /* define a raw backend-provided type */
   @[ target: LLVM::i64, signed: false ]@
   define #u64 : type <-> () ;

   /* define its signed counterpart */
   @[ target: LLVM::i64, signed: true ]@
   define #i64 : type <-> () ;

   /* define one of the few type-aliases that exist by default in Turned-C */
   @[ target: `i64 ]@ // metadata is inherited via this reference
   define #integer : type <-> () ;
   ```
2) The Use of Pointers and Arrays:
   ```turned-c
   /* Birth a named integer */
   define #x : i64 <-> 42 ;

   /* Birth a pointer to that integer using the & (Address-of) operator */
   define #ptr : i64 pointer <-> &x ;

   /* Birth an array of integers */
   define #list : i64 array <-> (1, 2, 3, 4, 5) ;

   /* Accessing the third element (Index 2) */
   define #val : i64 <-> list[2] ; 
   ```

##### Feature Specification

The following examples demonstrate how `macro` definitions may be used to implement core features of the languages entirely within the Bootstrap Core.
1) The `constant_if` "statement":
   ```turned-c
   /*
    * This is an example of a "conditional compilation" macro.
    * It is also the first sketch intended to demonstrate the `unquote` operator.
    *
    * Strong Note: This example assumes a much stronger macro-expansion model than the
    * C-hosted `Stage 0` minimal bootstrap compiler provides. Conditional compilation
    * of this kind should be treated as a `Stage 2` / self-hosted capability, not as
    * part of the minimal compiler's guaranteed behavior.
    *
    * used as `constant_if ( value, condition, condition truth branch, condition false branch) ;`
    * assume `print` has been defined as a function and `x` is a defined integer variable for the following more complete example:
    *   `constant_if ( x, [[ it == 0]] , { print( "Value is Zero!" ) ; } , { print( "Value is Non-zero!" ); } ) ;`
    */
   define #constant_if : macro <-> (value: node, condition: node, truth_branch: node, falsity_branch: node) -> {
      `{
         ({( value )}) <=> {( condition )} ? {( truth_branch )} || {( falsity_branch )} ;
      } ;
   } ;
   ```
2) The "match" feature:
   ```turned-c
   /*
    * This macro was written prior to the inclusion of the "unquote" operator in the language and should
    * be considered illustrative only and not used directly as the basis for anything. What it exists to
    * do is demonstrate how a variadic macro can handle things and recursively call itself to produce a
    * result.
    */
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
               // If it fails, recursively call 'match_macro' with the remaining arms.
               (predicate) ? (body) <=> eval || (subject, rest) <=> match ;
           } ;
       } ;
   } ;
   ```

The examples in this sub-section demonstrate how complex run-time features such as branching and looping can be implemented.

1) The `while` loop
   ```turned-c
   define while : function <-> (exit_condition: function boolean, core: function) -> {
      [[ () <=> exit_condition ]] ?  ( ) <=> { }  || (exit_condition, core) <=> while ;
   };
   ```
2) The `for` loop
   ```turned-c
   define for : function <-> (init: function, exit: function boolean, inc: function, core: function) -> {
      () <=> init; // run once

      define for_int: function <-> () -> {
         [[ () <=> exit ]] ? && () <=> {
            () <=> core; // run the loop code
            () <=> inc; // run the incrementor
            () <=> for_int; // recurse!
         } ;
      } ;

      () <=> for_int; // run the loop
   } ;
   ```
3) The `if` "Statement"
   ```turned-c
   define if : function <-> (condition: function boolean, truth: function, falsity: function) -> {
      [[ () <=> condition ]] ? () <=> truth || () <=> falsity ;
   };
   ```

The following examples demonstrate how to bind new operators based on backend primitive operations

1) Addition
   ```turned-c
   /* Provision '+' as a core infix operator over integer */
   @[ 
      target: LLVM::BuildAdd,
      precedence: 50,
      associativity: left,
      fixity: infix
   ]@
   define #+ : operator <-> () ;
   ```
2) Equality
   ```turned-c
   /* Provision '==' as a core comparison operator over integer */
   @[
      target: LLVM::BuildICmpEQ,
      precedence: 40,
      associativity: left,
      fixity: infix
   ]@
   define #== : operator <-> () ;
   ```
