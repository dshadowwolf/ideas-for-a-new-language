### General Overview
Turned-C is designed as a toolkit language, not a monolithic platform. Its purpose is to provide a minimalist, irreducible set of "Physical Laws" for systems programming while empowering the user to build their own higher-level paradigms.

To alleviate boilerplate and enable domain-specific optimizations, Turned-C implements a Macro Meta-Capability. Unlike the text-replacement of the C Preprocessor, a Turned-C macro is an AST-transformer. It informs the compiler's Pass 1 (The Macro Pass/The Transformer) to hand a specific AST subtree to a compile-time logic block, which then returns a newly created subtree that is a mutation or expansion of the original subtree to be spliced back into the program.

### A conceptual difference
In Turned-C a macro is not a text-replacer; it is a Compile-Time Identity. When a symbol is associated with the `macro` meta-capability it informs "Pass 1" (The Macro Pass/ The Transformer) that this name does not resolve to a runtime address but to a structural transformation.
1) The Input: AST as a Structure
   - Unlike a `function`, which receives values (integer, floats, objects), a `macro` receives AST Nodes. Because Turned-C represents the AST using the same `NODE_STRUCTURED` and `NODE_LIST` forms used for data, the macro simply sees its input as a "Tree of Identities".
2) The Back-tick: The Quoting Mechanism
   - To facilitate this, Turned-C utilizes the Back-Tick Operator. This operator acts as a "Shield" against immediate evaluation:
     - `x`: Evaluates to the value held by the name.
     - `` `x``: Evaluates to the identity of the name (the `BAREWORD` node). (note: the extra space before the `x` in this item is for markdown compatability)
     - `{ x + 1 }`: Evaluates to a callable thunk
3) The Execution: Transformation
   - The execution model is inspired by LLVM’s OrcJIT. During Pass 1, the compiler snapshots the AST arguments, executes the macro logic internally, and replaces the macro call with the returned AST node. This happens before Pass 2 (Module Resolution) or Pass 3 (Emitter) ever sees the code.
   - Pass 1 (The Macro Pass / The Transformer): This is the compile-time transformation pass that runs after the initial parse has produced an AST. During this pass, macro invocations are identified, expanded, and spliced back into the tree before later semantic or emission stages.

### Contracts and Safety
To maintain Provable Correctness, every macro must adhere to a strict Structural Contract.

#### The Meta-Capability Contract: `macro`
- Surface:
  - LHS: Parameter Template
  - RHS: Implementation Block
  - Result: Transformation Unit
- Compiler:
  - LHS: `NODE_STRUCTURED`
  - RHS: `NODE_FUNCTION`
  - Result: `NODE_MACRO`

#### Architectural Contracts
1) Immutability of Input: A macro may read its input AST nodes but must return _new_ nodes rather than mutating the input in-place, preventing "spooky action at a distance" during the transformation pass.
   - Transformation Rule: A macro may conceptually derive its result from the input subtree, but it must return a distinct AST node or subtree. The input nodes may be inspected and reused as structural source material, but the macro must not mutate the original input nodes in place.
   - Parameter Substitution Rule: Within a quoted macro result, direct use of a macro parameter name denotes structural substitution of the AST node bound to that parameter. The parameter name is not emitted as an identifier; its bound AST node is pasted into the returned tree at that position.
   - Body Normalization Rule (Provisional): In macro-generated control constructs, a branch body must be normalized to `NODE_FUNCTION`.
     - If the provided body is already `NODE_FUNCTION`, it is used directly.
     - If the provided body is `NODE_EXPRESSION`, the compiler wraps it as `{ <expression> }` during Pass 1 expansion.
2) Hygienic Isolation: By default, names defined inside a macro do not leak into the caller's scope unless explicitly "unquoted" or returned as part of the new AST.
   - Hygienic Scope Rule (Provisional): Names introduced during macro expansion are local to the returned AST and do not implicitly bind names in the caller's surrounding scope. The only effects a macro has on the caller are those explicitly represented in the AST subtree it returns.
3) The Return Contract: A macro _must_ return a valid AST node/subtree. Returning a "`null`" or an incomplete node will trigger a Transformation Error and halt compilation.
   - Stage 0 Note: In the current C implementation, this contract is represented by the internal type `ast_node_t`. This implementation detail is not itself part of the language-level specification.

### A Semantic Example: The `unless` Macro
To demonstrate the power of the "Toolkit" we can define a common logic gate that does not exist in the core grammar:
```turned-c
@[ scope: package ]@
define unless: macro <-> (condition: node, body: node) -> {
    // We return a "Quoted" if-statement that inverts the logic
    // Note: this is not guaranteed to be the final syntax and has an issue
    //       it unconditionally wraps the "body" parameter as a `NODE_FUNCTION`
    //       regardless of it it was a `NODE_FUNCTION` to begin with.
    `if ( [[ ! (condition) ]] , { body } ) ;
} ;

// usage:
unless ( ` ( it == 0 ) , { "Success" <=> print ; } );
```

#### A proposed "logic" path
A macro doesn't just have to "paste nodes" together to create the "new" subtree; it can use its own, internal logic to "normalize" them. See the following example of defining `if` as a macro:
```turned-c
define if : macro <-> ( condition : node, success : node, failure: node ) -> {
   // Check if 'success' is an expression or a block
   define s_node <-> [[ success.type == NODE_FUNCTION ]] ? success || `{ () <=> success };

   // Check if 'failure' is an expression or a block
   define f_node <-> [[ failure.type == NODE_FUNCTION ]] ? failure || `{ () <=> failure };

   // Return the expanded Template
   `[[ () <=> condition ]] ? s_node || f_node ;
} ;
// Note: This example relies on provisional macro introspection (`node.type`) and provisional control-flow lowering rules for `?` / `||`.
```
#### Macro Introspection Surface (Provisional)

The following node-introspection features are exposed only to macro evaluation during Pass 1 (The Macro Pass).

- `node.type`:
  - Returns the node-kind tag for a macro-visible AST node.
  - Valid examples include `NODE_FUNCTION`, `NODE_EXPRESSION`, `NODE_STRUCTURED`, and other Stage 0 node tags.
- Scope of availability:
  - Legal only inside macro-evaluation context.
  - Use outside macro expansion is a semantic error.
- Stability:
  - This interface is provisional and may be narrowed or renamed in later revisions.
  - To maintain portability between Stage 0 (C) and Stage 1 (Self-hosted), macro-visible node tags shall remain consistent with the internal node_type_t enumeration.

##### Another possible expansion of form for macros
```turned-c
// A recursive macro to expand a variadic print
define print_macro : macro <-> (args...) -> {
    [[ args.count == 0 ]] ? `() || {
        define first <-> args.head;
        define rest  <-> args.tail;
        `{
             (first) <=> print_single ; // Print the first one
             (rest)  <=> print_macro ;  // Recurse on the rest (at compile-time!)
        } ;
    } ;
} ;
```
As I hope is obvious in the above "print" macro, this introduces the concept of "variadic arguments" that use a variation of `Variadic Tail-Packing` which can be used to provide for a function without a fixed number of arguments. This method requires only a way to "extract" the elements from the `structured data` that the `variable arguments` are packed into and, thanks to the language definition clearly stating that values are to be passed as "fat pointers" (that is, they contain both the raw data and a chunk of extra information) means they also carry just enough "type information" for run-time detection of the type to be entirely within the capabilities of even the `stage 0` compiler.

Or, as can be seen in the following example, we can replace the originally proposed `switch-case` construct with a macro that expands to a chain of ternary selection statements similar to the previously suggested `if` macro, if on a slightly more grand scale:
```turned-c
// The recursive macro that builds a chain of 'if' logic
define match_macro : macro <-> (subject: node, arms...) -> {
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
            (predicate) ? (body) <=> eval || (subject, rest) <=> match_macro ;
        } ;
    } ;
} ;
```
Take everything within this section as provisional and barely more than a rough suggestion -- this is the beginning of a thought of how to provide most system primitives as macros to make the language look a touch more "classical" than the existing "everything that isn't an operator or keyword is a function provided by the Turned-C standard library" definition.

Note 1) There _must_ be a limit of some sort to the depth of the recursion -- what this limit is is TBD, but exceeding it should result in an error and compilation halting.
  - A safe and sensible solution might be to limit _all_ structured data to have no more than 1024 members at any given level - this would, of course, apply explicitly to var-args lists as well, capping their length at `1024 - number-of-preceding-arguments`

Note 2) The `args.count`/`args.head`/`args.tail` API is a rough sketch of an idea and the base name -- be it `args` or `arms` -- is taken from the "name" of the item given in the `structured definition` that has the `variadic` operator attached to it. These are not guaranteed to last for even the lifetime of this document as the idea behind things is iterated on and work done to make sure that things properly "meld" with the rest of the language. (it would not do for anything to do with such a core feature as the macro system to have bits that feel tacked on and the var-args syntax can actually extend beyond just macros)

Note 3) The `head` and `tail` accessors for the "list of nodes" is also provisional, but is solid enough that it may survive. This is because it is simple, robust and doesn't require any real work to have function properly.

Note 4) While the preliminary proposed sketch for `match_macro` may look like its all set to "wander off into a Null Pointer Dereference" the truth is that when `args` has nothing for `tail` (and hence `subject` is exhausted as well) the final call will arrive with an "empty list" -- triggering the `[[ arms.count == 0 ]] ` alternation that returns an "empty list".
