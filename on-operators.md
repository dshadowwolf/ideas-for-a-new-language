# On Operators

Turned-C is a minimalist systems language defined by a high-density operator grammar and a unified model of data and execution. This document serves as the Normative Design Specification, providing the formal semantics, precedence, and intended behavior for the language's core operators.

While this specification establishes the theoretical framework of Turned-C, the 'Stage 0' Pratt Parser and LLVM-backed implementation serve as the final canonical authority. In any instance where the abstract design and the physical implementation diverge, the behavior of the compiler shall be considered the definitive source of truth.

## Core Lexical Principles
- Operator Primacy: In Turned-C, operators are not mere sugar; they are the primary syntax for composition, association, execution and structural grouping.
- Extensible Grammar: Through the `operator` meta-capability and the annotation sub-parser, the language supports User-Defined Operators. These are integrated into the Pratt Parser with the same precedence and fixity rules as the core set.
- The "Punctuation Glue" Rule: The tokenizer is a Whitespace-Delimited Muncher. A change in character class (e.g. Alphanumeric to Punctuation) signals a token boundary, but consecutive punctuation is treated as a single compound operator token.
  - Example: `);` is lexed as a single Compound Token and must resolve as a defined operator token (core or user-defined) to be valid. To treat them as a "Closing Structure" followed by a "Terminator" they must be separated by whitespace: `) ;`
- Semantic Minimalism: The grammar is designed to minimize context-sensitive exceptions. The meaning of a token is determined by its Identity in the Symbol Table and its Precedence in the Pratt loop, not by surrounding "magical" keywords. Structural delimiters are first-class operators with full parse semantics, their behavior is still determined by operator identity, fixity, precedence and declared semantics.
  - Explicit Exceptions:
    - `@[ ... ]@`: the parsing of the contents of a "Definition Annotation block" is handed to a dedicated sub-parser
    - `![ ... ]!`: the parsing of the contents of a "Directive Annotation block" is handed to a dedicated sub-parser
    - `[ ... ]`: This operator has three independent meanings, two that are tightly coupled and one that is related in concept if not execution. The different variations are all explained in this document, as are their semantics.
  
## Core Design Operators

These operators are central to the language design.

| Operator | Precedence | Associativity | Meaning | Note |
| --- | ---: | --- | --- | --- |
| `:` | 95 | left | Type/capability separator | Maps a symbol to its Typespec or Capability set; binds tighter than flow or association |
| `->` | 1 | right | Inject/apply | The "Wiring Diagram"; binds a structural template to a deferred code block. |
| `<->` | 10 | right | Associate/"put" | The "Marriage"; comples a `define` by binding a name to a value or logic. |
| `<-` | 30 | right | Flow/Bind | Moves data into a mutable slot; respects `atomic` and `capability` constraints. |
| `<=>` | 20 | left | Immediate execution/application | The "Trigger"; activates a logic block using the provided LHS context |
| `[=]` | 20 | left | Deferred execution/application | The "Thunk"; captures context into a serializable structure for late-bound activation. |
| `[[ ... ]]` | 80 | right | Predicate lambda form used in matching/switch-like constructs | Specialized Boolean-only lambda; automatically injects the `it` symbol into its internal scope. |
| `( ... )` | 90 / 0 | left / none | Structured data form, grouping, or postfix call/apply | Pass 1 NUD/LED; promotes a `NODE_LIST` to a `NODE_STRUCTURED` for context or templates |
| `{ ... }` | 92 / 0 | left / none | Code block / closure / execution group | Defined a Deferred Block; encapsulates a sequence of statements as a single executable unit. |
| `[ ... ]` | 92 / 0 | left / none | Operator / method overload group | Parses only after a definition bind (`<->`); `]` seals overload set. |

- **Note**: Method overloading via `[ ... ]` is provisional for the Bootstrap Core. Operator overloading (type-dispatch) is the primary motivating use case. Method overloading introduces name-mangling complexity (cf. `C++`) and may be deferred or excluded from the release specification of `Turned-C`.
- **Note**: The actual format of how `[ ... ]` and operator / method overloading works has slightly changed to have a more idiomatic form and a much stricter definition. Actual documentation of this is planned for some time in the up-coming week. (Note added 19-MARCH-2026)  

#### `->`

The slim right arrow is the language's inject/apply operator. It is a right-associative operator of low-precedence (just above the minimum) representing the composition of a data structure with an execution context. It "injects" the identities and values of the left-hand side into the scope of the right-hand side. In Turned-C, a function is not a standalone "magic" entity; it is a Mapping. The -> operator facilitates this mapping by binding a Structural Template (parameters/context) to a Deferred Block (the thunk or closure).

Operator Contract:
- Surface:
  - LHS: structured context/template.
  - RHS: executable block/callable logic.
  - Result: injected mapping (callable definition/contextual function).
- Compiler:
  - LHS: `NODE_STRUCTURED`
  - RHS: `NODE_FUNCTION`
  - Result: `NODE_INJECT`
- Architectural Contract:
  - Contextual Binding: The LHS provides the "Symbolic Identity" (parameters) that the RHS logic requires to execute.
  - Scope Injection: The members of the LHS structure are "unpacked" into the top-level lexical scope of the RHS block, shadowing any identical names in the parent scope to ensure lexical isolation.
  - Result (`NODE_INJECT`): Represents a complete "Function Template". It is the prerequisite for an "Association" (`<->`) to a name.

Examples:
  
```turned-c
define square: mutable int <-> (x: integer) -> { x * x };
```

Semantically, this can be read as:
- Contextual Injection: "Bind these named identities into the following execution scope."
- Structural Application: "Apply this data-shape constraint to this block of logic"
- Functional Mapping: "Define a transformation where the left-hand structure provides the environment for the right-hand execution"

Technical Behavior:
- Pass 1 (Macro/Parsing): Identifies the boundary between a function's "Input Shape" and its "Implementation"
- Pass 3 (Emitter): Informs the LLVM backend to prepare a stack frame or closure object where the left-hand members are mapped to the local registers or offsets accessible by the block.
  
#### `<->`

Definition:
- A right-associative operator (precedence 10) that establishes a definitive symbolic binding between a name and its value, structure or code block.

Architectural Role:
- In Turned-C, the `<->` operator is the primary mechanism for Declarative Binding. Unlike the temporal "assignment" found in purely imperative languages, Association is a lexical assertion that completes a `define` statement. For immutable identities, this mapping is invariant; for mutable identities, subsequent value flow or rebinding is governed by the Flow Operator (`<-`) operator and its associated capability constraints.

Operator Contract:
- Surface:
  - LHS: typed defined identity (`define name : type`)
  - RHS: value, structured aggregate, or code block compatible with declared type/capabilities
  - Result: completed declaration-time association.
- Compiler:
  - LHS: `NODE_DEFINE` + `NODE_TYPESPEC` (a bare `NODE_DEFINE` without a `NODE_TYPESPEC` is a corner case; behavior is currently undefined)
  - RHS: `NODE_CONSTANT` | `NODE_FUNCTION` | `NODE_STRUCTURED`
  - Result: `NODE_BIND`
- Architectural Contract:
  - Definitive Genesis: This is the "Anchor" of a name's existence within a scope.
  - Immutability Rule: For standard identities, this mapping is _invariant_. For those with the `mutable` capability, this sets the initial state.
  - Result (`NODE_BIND`): Finalizes the symbol table entry. It transitions a name from a "pending definition" to a "resolved entity".

Examples:

```turned-c
define mult: int <-> 100;
define square: mutable int <-> (x: integer) -> { x * x };
define point: structure <-> ( x: double, y: double, z: double );
```

Semantic Interpretations:
- Symbolic Genesis: "This is the point of origin for this name's identity in the current scope."
- Initial Mapping: "Associate this symbol with this specific data or logic to complete its definition."
- Scoped Resolution: "Finalize the declaration of this entity by providing its physical or logical manifestation."
  
Technical Behavior:
- Pass 1 (Macro/Parsing): Informs the parser that the left-hand `NODE_DEFINE` is now fully resolved and ready for use in subsequent expressions.
- Pass 3 (Emitter): 
  - For Constants/Functions: Often results in zero-cost aliasing or metadata-only entries in the object file.
  - For Structures: Emits the one-time initialization (e.g., `memcpy` or stack setup)
  
#### `<-`

The left-pointing arrow is the nominal association operator. A right-associative operator of middling precedence used to explicitly map a value to a named "slot" within a structure, parameter list or execution context.

Architectural Role:
In Turned-C, the `<-` operator is the primary tool for Structural Population. While `<->` defines what a thing _is_ globally, `<-` defines how data "flows" into a specific instance or "hole" in a template. It eliminates the fragility of "positional" programming by making the relationship between data and its destination explicit and self-documenting.

Operator Contract:
- Surface:
  - LHS: named slot or mutable identity
  - RHS: expression resolving to compatible type
  - Result: targeted update/population
- Compiler:
  - LHS: `NODE_IDENTIFIER`
  - RHS: `NODE_EXPRESSION`
  - Result: `NODE_FLOW`
- Architectural Contract:
  - Identity Mapping: Specifically targets a named "hole" within a structure or a `mutable` variable in the current scope.
  - Imperative Flow: Unlike `<->`, this operator implies a "temporal movement" of data from a source to a destination.
  - Result (`NODE_FLOW`): Triggers a side-effect. In the LLVM backend, this results in a `store` instruction or a structural offset update.

Examples:

```turned-c
(x <- 10.00, y <- -25.42, z <- 15.33)
(format <- "square of 2 times 100 is {}", args <- [(x <- 2) <=> f])
```

Semantic Interpretations:
- Targeted Injection: "Direct this specific value into this named parameter."
- Field Population: "Initialize this member of the structure with this result."
- Explicit Mapping: "Bind this value to this internal name for the duration of this call or instance."
  
Technical Behavior:
- Pass 1 (Macro/Parsing): Identifies a Key-Value Pair within a `NODE_STRUCTURED` or `NODE_CALL`.
- Pass 3 (Emitter): In the LLVM backend, this resolves to a specific Offset Store within a struct or a Symbolic Mapping in a function's stack frame. If used on a mutable variable, it triggers the machine's Physical Store instruction.
  
#### `<=>`

The fat double arrow is the immediate execution operator. A left-associative operator of relatively low precedence that triggers the instantaneous evaluation of a callable identity using a provided structural context. It represents the transition from a "Deferred Block" to a "Resultant Value".

Architectural Role:
In Turned-C, code blocks and data structures exist in a state of "potential" until they are bound together and "Activated". The `<=>` operator is the primary mechanism for Function Invocation. It signals to the compiler that the left-hand context must be mapped to the right-hand logic immediately, blocking further progression until the transformation is complete.

Operator Contract:
- Surface:
  - LHS: call context/argument aggregate
  - RHS: callable identity
  - Result: immediate evaluated expression value
- Compiler:
  - LHS: `NODE_STRUCTURED`
  - RHS: `NODE_FUNCTION` | `NODE_IDENTIFIER`
  - Result: `NODE_CALL_IMM`
- Architectural Contract:
  - The Activation Trigger: This operator serves as the synchronous "bridge" between static data and dynamic logic. It forces the immediate resolution of the RHS logic using the LHS context.
  - Temporal Guarantee: Evaluation is blocking and immediate. The program counter will not advance to the next expression until the transformation is complete and a result (even a `void` result) is returned.
  - Side-Effect Serialization: If the RHS is a `mutable` function, this operator creates a deterministic sequence point in the program flow, ensuring that global state changes occur exactly at this point in the instruction stream.
  
Examples:

```turned-c
(x <- 2) <=> f
(format <- "{}", args <- [(x <- 2) <=> f]) <=> print_line
```

Semantic Interpretations:
- Immediate Activation: "Evaluate this block now using these parameters."
- Synchronous Transformation: "Pass this data through this logic and return the result to this point in the expression."
- Contextual Trigger: "Apply this specific environment to this function and resolve the value."
  
Technical Behavior:
- Pass 3 (Emitter): Directs the LLVM backend to emit a standard `call` instruction. It prepares the stack frame (the `NODE_STRUCTURED` on the left), jumps to the entry point (the `NODE_FUNCTION` on the right), and retrieves the return value.
- Determinism: Unlike the deferred version (`[=]`), this operator guarantees that the side effects (if any) and the computation occur exactly at this point in the program flow.
  
#### `[=]`

The blunted fat double arrow is the deferred execution operator. A left-associative operator of the same level of precedence as the `<=>` operator that captures a structural context and a callable target into a stable closure, delaying evaluation until the resultant value is explicitly required by a subsequent expression.

Architectural Role:
In Turned-C, the `[=]` operator is the primary mechanism for Lazy Evaluation. It allows a programmer to define a "Potential Result" without incurring the immediate cost of CPU cycles or memory-state changes. The operator "blunts" the trigger of activation; it binds the data to the logic and stores the pair as a Thunk (a self-contained execution package).

Operator Contract:
- Surface:
  - LHS: call context/argument aggregate
  - RHS: callable identity
  - Result: lazy evaluated expression value
- Compiler:
  - LHS: `NODE_STRUCTURED`
  - RHS: `NODE_FUNCTION` | `NODE_IDENTIFIER`
  - Result: `NODE_CALL_DEF`
- Architectural Contract:
  - The `NODE_CALL_DEF` serves as a "Potential Value":
    - Encapsulation: It wraps the logic and its arguments into a single object (the Thunk).
    - Deferred Activation: It remains inert until a "Strictness Point" (like a math operator or a `mutable`  flow) forces its evaluation.
    - Caching: Once evaluated, the result is typically cached within the thunk to prevent re-computation, fulfilling the "call-by-need" requirement.

Example:

```turned-c
define newLocation: point <-> (x <- -5.25, y <- 25, z <- -16) [=] location.addOther;
```

Semantic Interpretations:
- Lazy Resolution: "Prepare this calculation, but do not execute it until the data is actually needed".
- Contextual Capture: "Snapshot these variables and this function into a single identity for later use".
- Potential Value: "Represent the result of this operation as a deferred state".
  
Technical Behavior:
- Pass 3 (Emitter): Instead of a direct `call`, the LLVM backend emits a Closure Object. This object stores the address of the code and a copy (or reference) to the left-hand structure.
- Hidden Activation: When a name associated via `[=]` is used as an operand in a math, bitwise or flow operation, the compiler implicitly triggers the execution to "realize" the value.
  
#### `[[ ... ]]`

Definition:
- This operator defines a predicate lambda.
- A right-associative operator of decently high precedence that defines an anonymous, high-performance boolean guard. It creates a hyper-specialized execution scope with a single, implicit input -- the subject of evaluation -- traditionally defined as `it`.

Architectural Role:
- In Turned-C, the `[[ ... ]]` operator is the primary tool for Declarative Dispatch. Unlike a standard function, a Predicate Lambda is designed for Introspection and Classification. It allows the compiler to treat conditional logic as a "Filter" that the backend is free to optimize how it wishes, including, where applicable, jump tables or branch-prediction-friendly branch sequences within a `switch` or a `match` construct.

The `DEFAULT` Predicate:
- The language provides a universal, pre-defined predicate -- `[[ DEFAULT ]]` -- which serves as the system's "Unconditional Match".
  - The Catch-All Rule: It evaluates as a constant `true`, designed to intercept any subject not captured by a preceding specialized predicate.
  - The Terminal Constraint: Because it is unconditionally greedy, it serves as a logical sink. It must be placed as the final branch of a dispatch block; placing it earlier will "short-circuit" the match logic, rendering all subsequent branches unreachable code.
  
Operator Contract:
- Surface:
  - LHS: None.
  - RHS: Boolean expression or logic block.
  - Result: Anonymous boolean guard.
- Compiler:
  - LHS: None
  - RHS: `NODE_EXPRESSION` | `NODE_FUNCTION`
  - Result: `NODE_PREDICATE_LAMBDA`
- Architectural Contract:
  - LHS: None -- As a prefix operator (or "null denotation" in Pratt terms), it does not take an operand on its left. It initiates a new, hyper-specialized scope.
  - RHS: This is the "Filter Logic".
    - If the RHS is a `NODE_EXPRESSION` (e.g. `it == 0`), the compiler wraps it in a function body that returns a `boolean`.
    - If the RHS is a `NODE_FUNCTION` (e.g. `{ it == 0 }`), the block is validated to ensure its terminal expression is a boolean.
  - Result (`NODE_PREDICATE_LAMBDA`)
    - Implicit Identity Injection: This node type is responsible for injecting the symbol `it` into the immediate local symbol table.
    - Strict Return Type: Unlike a standard `NODE_FUNCTION`, this node carries a rigid "Boolean Contract". It cannot be associated with any name or slot that expects a non-boolean result.
    - Dispatch Optimization: This specific node type allows the "Pass 3 Emitter" to identify it as a "Guard" and potentially lower it into an optimized jump-table entry rather than a full, overhead-heavy function call.
- The "Special Case": `[[ DEFAULT ]]`
  - Compiler Logic: When the RHS is the keyword `DEFAULT`, the compiler produces a `NODE_PREDICATE_LAMBDA` whose internal logic is a constant `true` return.
  - Terminal Rule: `[[ DEFAULT ]]` must be terminal in an ordered dispatch chain; non-terminal placement is a semantic error. The emitter uses the presence of this node to signal the end of a match-chain, preventing any subsequent branches from being evaluated.
  
Examples:

```turned-c
// the universal catch-all predicate
[[ DEFAULT ]]

// A numeric equality check against the implicit subject "it"
[[ it == 0 ]]

// A range check for signed integers
[[ it < 0 ]]

// Using a member access on a structured subject
[[ it.x > 10.0 && it.y < 0.0 ]]
```

Semantic Interpretations:
- Logical Guard: "Evaluate the following condition against the current subject".
- Symbolic Classifier: "Check if the subject (`it`) satisfies this specific requirement".
- Truth-Value Filter: "Provide a boolean result based on the shape or value of the input".

Technical Behavior:
- Implicit Identity: The operator automatically injects the name `it` into the local symbol table, associated with the value being matched.
- Boolean Enforcement: The Semantic Pass (Pass 2/3) requires that the internal expression resolve to a `boolean` type; any other result is a type-mismatch error.
- Match Optimization: In Pass 3 (Emitter), the LLVM backend can often inline these predicates into a single `cmp` and `br` sequence, ensuring that the "Pattern Match" is as fast as a C `if` statement

#### `( ... )`

Definition:
- The `( ... )` operator pair is a dual-form structural operator in Turned-C.
- Prefix form creates a grouped or structured aggregate context.
- Postfix form (`name(...)`) is convenience syntax for immediate application and is semantically equivalent to explicit form (`(... ) <=> name`).
- In all forms, `()` denotes a heterogeneous aggregate of expressions and/or nominal associations.

Architectural Role:
- In Turned-C, `()` is the primary aggregate-construction and context-packaging form.
- It turns a list of expressions or nominal associations into a single logical unit that can be passed, stored, injected, or immediately applied.
- Whether this unit is materialized as contiguous storage or optimized away is implementation-dependent.

Operator Contract:
- Surface (prefix form `(...)`):
  - LHS: None.
  - RHS: Comma-separated expressions and/or nominal associations.
  - Result: Structured aggregate/context.
- Surface (postfix form `callee(...)`):
  - LHS: Callable identity.
  - RHS: Structured aggregate/context.
  - Result: Immediate-apply call form (sugar over explicit apply path).
- Compiler (Stage 0):
  - LHS: None (prefix parse).
  - RHS: List of `NODE_EXPRESSION` and/or `NODE_FLOW` entries.
  - Result: `NODE_STRUCTURED`.
  - Note (postfix form): The postfix `callee(...)` form is provisional syntactic sugar whose long-term status in the language is undecided. If retained, it will be lowered to the equivalent `(...) <=> callee` form at parse time (Pass 1).
- Architectural Contract:
  - Inner content is an ordered comma-separated sequence (zero or more entries).
  - `NODE_STRUCTURED` is a logical aggregate layout: it may be materialized contiguously, stack-laid out, or optimized away depending on usage.
  - In call context, it is an argument bundle.
  - In definition/context-template use, it is the type/slot template.
    
Examples:

```turned-c
(x: integer) -> { x * x }
point.addOther( (-5.25, 25, -16) )
```

Semantic Interpretation:
- Data Aggregation: "Pack these distinct values into a single, physical shape"
- Call Context: "Prepare this specific environment to be applied to a logic block"
- Tuple Realization: "Treat this collection as a unified result"
  
Technical Behavior:
- Pass 1 (Macro/Parsing): The `(` opener begins collection of a comma-separated entry list. Each entry is parsed as either a `NODE_EXPRESSION` or a `NODE_FLOW` (nominal association). The matching `)` closes the aggregate and finalizes the `NODE_STRUCTURED` node.
- Pass 3 (Emitter): This may map to an aggregate value, stack layout or be optimized away depending on usage.
  
#### `{ ... }`

Definition:
- Defines a "Deferred Execution Scope" -- it captures a sequence of expressions and associations into a "Logic Package" that remains inert until triggered.
  
Architectural Role:
- In Turned-C the `{}` operator is the primary tool for Algorithmic Encapsulation. It represents "Code as Data". Unlike a standard C block it does not execute simply by being "passed over" by the instruction pointer; it must be "Activated" via an execution operator (`<=>` or `[=]`).

Operator Contract:
- Surface:
  - LHS: None.
  - RHS: Sequence of expressions and associations.
  - Result: Deferred execution logic (thunk/closure value).
- Compiler:
  - LHS: None.
  - RHS: Ordered internal list of expression/association nodes.
  - Result: `NODE_FUNCTION`.
- Architectural Contract:
  - Inner content is a semicolon-separated execution sequence.
  - Result (`NODE_FUNCTION`):
    - Encapsulates logic into a deferred-execution unit with lexical scope.
    - The terminal expression in the sequence defines the implicit return value.
    - Creates a new lexical symbol-table layer (local scope).
  
Semantic Interpretations:
- Deferred Logic: "Package this sequence of actions into a reusable recipe"
- Execution Thunk: "Represent this code as a single value that can be associated with a name or passed to a function"
- Private Scope: "Establish a new lexical layer where local definitions are protected from the outside world"

Technical Behavior:
- Pass 1 (Macro/Parsing): The `{` opener triggers a new deferred-scope boundary. The parser records the inner expression sequence, identifies the terminal expression as the implicit return value, and pushes a new lexical scope layer. The matching `}` closes the scope and finalizes the `NODE_FUNCTION` node.
- Pass 3 (Emitter): Typically lowered as a closure/function-like unit; the code-generation backend may inline or lift.

#### `[ ... ]`

Definition:
  - A "locked" scope for defining operator and method overloads. As part of a `name definition binding` it provides a space where the "symbolic name" is mutable and can have multiple, related but distinct meanings given to it via differences in parameter type.
  - The primary and fully-specified use of this form is **operator type-dispatch**: defining distinct machine-instruction targets for a single operator symbol based on operand types. Method overloading (via `: function` definitions) shares this syntax for structural consistency but remains provisional; it introduces name-mangling requirements that are not yet designed for Turned-C and may be excluded from the release specification. See also the note in the Core Design Operators table.

> **Pending:** `NODE_LIST_GROUP`, `NODE_OVERLOAD_SET`, and `NODE_ASSOCIATE` are forward-declared node types required by this operator. They are not yet present in `ast.h` and must be added before this form can be implemented.

Operator Contract:
- Surface:
  - LHS: A bound symbol or operator identity (via `<->`).
  - RHS: A sequence of annotated Signatures — each of the form `@[ target: ... ]@ ( lhs: type, rhs: type ) : return_type ;` — terminated by `]`.
  - Result: A `NODE_OVERLOAD_SET` (A dispatch table).
- Compiler:
  - LHS: `NODE_BIND` | `NODE_ASSOCIATE`.
  - RHS: `NODE_LIST` of `NODE_STRUCTURED` (the signatures), each carrying an attached `NODE_ANNOTATION` and a return-type specifier.
  - Result: `NODE_OVERLOAD_SET`.

Example:
```turned-c
define #/ : operator <-> [
    @[ target: LLVM::sdiv ]@
    (lhs: integer signed, rhs: integer signed) : integer signed ;
    @[ target: LLVM::udiv ]@
    (lhs: integer unsigned, rhs: integer unsigned) : integer unsigned ;
    @[ target: LLVM::fdiv ]@
    (lhs: float, rhs: float) : float ;
] ;
```

Architectural Role:
  - The "Sealed" Contract: Once the closing `]` is parsed, the overload set is immutable. This prevents the "Ad-hoc Overloading" mess where new meanings can be injected from distant modules.
  - Dispatch Mapping: It transforms a single symbolic name into a Multi-Target Map based on type-signatures.

Semantic Interpretation:
  - Capability Mapping: "Define the specific logic-targets available for this symbol across different type-contexts."
  - Signature Registry: "Establish the valid input/output contracts for this operator or method."

Technical Behavior:
  - Pass 1 (Pratt/Parsing):
    - Dispatched via the LED handler (Precedence 92).
    - List Collection: Consumes the `[` and enters a nested parse_expression loop that treats `;` as a signature separator and `]` as the zero-precedence terminator.
    - Signature Unit: Each signature is parsed as `@[ ... ]@ ( ... ) : return_type ;`. The `(` opens a `NODE_STRUCTURED` for the parameter list; the `:` following the `)` is parsed as a return-type annotation (distinct from the type-capability separator in definition context — the Pratt parser does not need context-awareness to parse it correctly as it only appears inside an overload frame following a `( ... )` block).
    - Neutral Output: Produces a generic `NODE_LIST_GROUP` of signature units; semantic promotion happens in Pass 4.
  - Pass 4 (Semantic Validation):
    - The Context Check: Inspects the LHS. If the LHS is part of a `define #name :` operator chain, promotes the `NODE_LIST_GROUP` to a `NODE_OVERLOAD_SET` and validates each annotation's `target:` value against the operand types. If the LHS is part of a `define #name :` function chain, promotes under provisional method-overload rules (see note above).
    - Return Type Validation: Verifies that each signature's declared `: return_type` is compatible with the `target:` annotation and the LHS's declared type-class.
    - Signature Collision: Verifies that no two signatures in the set are identical (same parameter types, same target).
    - Illegal Context: Any `NODE_LIST_GROUP` not attached to a `define `: operator` or `define `: function` LHS is a semantic error: "Dispatch frame requires an `operator` or `function` target."
  - Pass 5 (Metadata):
    - Serializes the entire set into a `TAG_List` of `TAG_Compound` (`NBT`) within the `.tch` file, ensuring the dispatch table is available for external importers.

## Structural Operators

These operators shape declarations, grouping, or parse boundaries rather than directly modeling arithmetic or comparison.

| Operator | Precedence | Associativity | Meaning | Notes |
| --- | ---: | --- | --- | --- |
| `#` | 99 | none | "Genesis" marker | Marks an identifier as a brand-new symbol; fails if already exists in scope. |
| `` ` `` | 99 | none | Raw symbol marker | Accesses a raw symbol handle; bypasses evaluation to allow redefinition or reference. |
| `@[ ... ]@` | 98 | left | Definition Annotation | State metadata (precedence, target, etc.) that attaches to the following definition. |
| `[ ... ]` | 92 | left | Array access / indexing | Balanced LED; performs bounds-checked array access or raw pointer offsets. |
| `.` | 85 | left | Member access | Navigates the internal namespace of a structure or module (Public/Internal). |
| `,` | 5 | left | Expression list separator / list composition | The "Collector"; aggregates siblings into `NODE_LIST` |
| `;` | 0 | none | Expression/statement terminator | The "Hard Stop"; finalizes the statement and commits the symbol metadata. |
| `=>` | 15 | right | Destructuring | The "Unpacker"; maps execution results into structured patterns for extraction. |
| `![ ... ]!` | 98 | left | Directive Annotation | Instructional hints (atomic, expect, etc.) for the compiler; can be advisory or mandator. |
| `&&` | 35 | left | Logical AND/Guard | Performs short-circuiting coalescence; evaluates the RHS only if the LHS is `true`. |
| `\|\|` | 35 | left | Logical OR/Choice | Consumes the alternation path; short-circuits on a true LHS, otherwise resolves the RHS choice branch. |
| `?` | 82 | left | Alternation | The structural branch point; triggers a conditional lookup for a following `&&` (guard) or `||` (choice) |
| `{( ... )}` | 99 | none | Unquote | Macro-only prefix form; expands a quoted node into the surrounding quoted structure |

#### `#`

Definition:
  - The Hash/Genesis Marker is a high-precedence, non-associative prefix operator (Precedence 99).
  - It signals the Birth of an Identity. It explicitly instructs the compiler that the following identifier is a brand-new entry being introduced to the current scope.
  - It leaves the symbol table entry in a "Fluid/Mutable" state for immediate decoration (Typespecs, Capabilities, Metadata) until a structural terminator (like ;) is encountered.

Operator Contract:
-Surface:
  - LHS: None (Prefix only).
  - RHS: A raw identifier or operator symbol token.
  - Result: a `NODE_SYMBOL_GENESIS` handle.
- Compiler:
  - LHS: None.
  - RHS: `NODE_IDENTIFIER` or `NODE_OPERATOR`
  - Result: `NODE_SYMBOL_GENESIS`

Architectural Contract:
  - Exclusivity Rule: The `#` operator fails if the RHS identifier already exists in the current lexical scope. It is the "Uniqueness Assertor."
  - Initialization State: Upon capture, the symbol is entered into the table with a "Pending" flag. It is technically mutable only within the context of the current define statement to allow for attribute stacking (e.g., `mutable int`).
  - Post-Terminator Lock: Once the statement terminator (`;`) is hit, the symbol’s initial "Genesis Mutation" window closes. Its future mutability is then governed strictly by whether the mutable capability was attached during this phase.
  - Architectural Guardrail: The Genesis Marker is prohibited only on System-Level Primitives (e.g., define, mutable, operator, macro). These represent the "Language Substrate." All other identities—including those traditionally seen as keywords in other languages (like if, for, or match)—are eligible for Genesis if they do not yet exist in the current scope.

Semantic Interpretation:
  - Declaration of Intent: "I am introducing a new name to the language's vocabulary at this specific scope."
  - Birth of Identity: "This symbol has no prior history; prepare a new slot in the registry."
  - Contextual Anchor: "Treat the following name as the root of a new contractual definition."
  
Technical Behavior:
  -  Pass 0 (Pratt/Parsing):
  -  Dispatched via the NUD (Null Denotation) handler.
     -  Atomic Capture: With precedence 99, it immediately binds the following token before any LED operator (like .) can attempt to claim it.
     -  Symbol Table Injection: Immediately creates a placeholder in the symbol table to prevent name-clashes within the same expression.
  -  Pass 2 (Semantic):
  -  Verifies that all metadata attached during the definition (via -> or <->) is compatible with the symbol’s intended meta-capability.
  -  Diagnostics:
  -  Collision Error: Thrown if # is used on an existing name.
     -  Type Error: Thrown if the Genesis chain ends without a valid type or value association.

#### `` ` ``

Definition:
  - The backtick is the raw symbol marker (quasiquote operator).
  - It requests direct symbolic reference rather than normal interpretation.
  - When applied, the parser treats the following token as a symbol identity lookup target, not as a value to evaluate or a declaration to introduce.
  - In macro contexts, it also acts as a quote/preservation marker for AST shape, preventing reduction of the quoted form.

Operator Contract:
- Surface:
  - LHS: None (prefix form).
  - RHS: Symbol token (identifier or operator symbol identity), or a macro-context form that must be preserved structurally.
  - Result: Direct symbol reference handle, or quoted/preserved AST form (macro context).
- Compiler (Stage 0):
  - LHS: None.
  - RHS: `NODE_IDENTIFIER`, operator-token identity, or macro-quoted AST-bearing form.
  - Result: Symbol-reference expression form or quote-preserved AST node (implementation-specific node shape; currently parser-sketch level).

Architectural Contract:
- Non-Creating Lookup Rule: Backtick never creates a symbol-table entry; it only addresses an existing one.
- Interpretation Suppression Rule: Normal contextual reinterpretation of the RHS token is bypassed for this operation.
- Macro Preservation Rule: Inside macro expansion logic, quoted forms are preserved at their current AST shape unless explicitly transformed by macro code.
- Node Stability Rule: A quoted `NODE_FUNCTION` remains a `NODE_FUNCTION` (no implicit lowering, eager activation, or normalization into another node form due solely to contextual interpretation).
- Namespace Access Rule: Enables direct reference to symbol identities that may otherwise be awkward to address through ordinary expression forms.
- Mutability Boundary: Backtick does not bypass symbol mutability or redefinition rules; it only changes how a name is referenced.

Semantic Interpretation:
- Raw Identity Access: "Treat this token as a symbol-table identity, not as a value-producing expression by itself."
- Quoted Symbol Form: "Preserve the symbolic name as a first-class reference target."
- AST Shape Freeze (Macro Context): "Preserve this syntax as this exact node shape until macro logic explicitly rewrites it."
- Explicit Lookup Intent: "I am referring to this exact symbol entry directly."

Technical Behavior:
- Pass 1 (Macro/Parsing):
  - Parsed as a high-precedence prefix operator (table precedence `99`).
  - Consumes the immediate RHS token as a raw symbol identity.
  - Suppresses normal declaration/interpretation flow for that RHS token within this operator form.
  - In macro context, records the RHS as quote-preserved AST material and skips implicit node-shape reduction for that fragment.
  - Performs (or schedules) symbol-table resolution for the referenced identity.
- Diagnostics:
  - If the referenced symbol does not exist in accessible scope, this form is a semantic error.
  - If the RHS is not a valid symbol identity token, this form is a parse error.
- Pass 2/3 (Semantic/Emitter):
  - The resolved symbol reference is carried forward for type/capability checks and backend lowering according to the referenced symbol class.
  - For macro-preserved forms, downstream passes operate on the node shape produced by macro expansion, not on an implicitly reduced variant.

#### `@[ ... ]@`

Definition:
  - This defines a set of "annotations" -- "attributes" is what most languages call them -- that define a set of metadata attached to the name that is not directly lexical, but has other meaning that applies to other parts of the parsing process. All of this data is stored in the `.tch` "header file" with the "name" itself as it is required for said "name" to be properly understood and utilized.

Operator Contract:
- Surface:
  - LHS: None (annotation opener form).
  - RHS: Annotation payload (`key: value` entries and/or flag-style keys).
  - Result: Metadata bundle that decorates the immediately following definitional target.
- Compiler:
  - LHS: None.
  - RHS: Annotation token stream handled by a dedicated annotation sub-parser.
  - Result: `NODE_ANNOTATION` (or equivalent metadata node attached to a target node/symbol record).

Architectural Contract:
- Attachment Rule: Annotation metadata does not stand alone; it must attach to a valid following target (definition, operator definition, macro definition, etc.).
- Non-Execution Rule: Annotation payload never evaluates as runtime expression logic; it only influences compile-time interpretation, validation, optimization, or backend mapping.
- Persistence Rule: Accepted annotation metadata is serialized into translation-unit metadata (`.tch`) together with the decorated symbol/operator so imports can reconstruct intent without reparsing source.
- Scope Rule: Annotation validity is context-sensitive; unsupported annotation keys for a target kind are semantic errors.

Semantic Interpretation:
- Metadata Declaration: "Apply these compile-time attributes to the next symbol/operator definition."
- Compiler Guidance: "Adjust how this identity is interpreted, checked, optimized, or lowered."
- Cross-Unit Contract: "Persist this metadata so downstream translation units can consume the same meaning."

Technical Behavior:
- Parsing Stage:
  - `@[` opens an annotation block and transfers control to the annotation sub-parser.
  - The sub-parser consumes payload entries until matching `]@` and validates annotation syntax.
  - On success, the parsed metadata bundle is staged for immediate attachment to the next valid target node.
- Validation Stage:
  - Unknown keys, malformed values, or target-incompatible annotations produce semantic diagnostics.
  - An annotation block with no valid following target is a semantic error.
- Metadata Packaging Stage:
  - Resolved annotation data is embedded in the translation unit metadata (`.tch`) alongside the decorated item.

#### `[ ... ]`

Definition:
  - This is the "index operation" and should be familiar to anyone who has used most higher-level languages developed since the "birth" of `C`, at a minimum. When attached to a pointer this has the effect of "adding" the index value to the pointer base, `define #a: integer pointer ; a[10] <- 10;` is the same as `define #a: integer pointer ; *(a+10) <- 10;`). But that is a secondary use and a subtle nod towards the utility found in that very use in `C` by this documents author. The intended use is as a bounds-checked index into a "name" that has its type defined to be an "array".

Operator Contract:
- Surface:
  - LHS: any item with a "resolved type" that is either "array" or "pointer"
  - RHS: An index expression (anything that resolves to an integer)
  - Result: `NODE_INDEX_ACCESS` (if used on an `array`) or `NODE_DEREF` (if used on a `pointer`)
- Compiler:
  - LHS: `NODE_IDENTIFIER` or other `NODE_XXX` where the "type" contains either the "array" or "pointer" modifier
  - RHS: `NODE_MATH` | `NODE_FUNCTION` | `NODE_CONSTANT` | `NODE_IDENTIFIER` where the `type` resolves to "integer"
  - Result: `NODE_INDEX_ACCESS` | `NODE_DEREF`

Architectural Contract:
- The Safety Boundary: For symbols carrying the `array` type-modifier the `[ ... ]` operator implicitly includes a Bounds Check against the array's metadata. For symbols carrying the `pointer` type-modifier it performs a raw C-style offset calculation (`base + (index * sizeof(type))`).
- LHS Dependency: This operator requires a resolved type from, ideally, Pass 2 (Import/Module) to determine its final lowering strategy with Pass 4 (Semantic Validation) as a fallback.
- RHS Integer Constraint: The index expression must resolve to a discrete integer. Non-integer types (unless they carry a specific `cast-to-int` capability (none are defined for Turned-C a this time)) are a semantic error.
- LHS L-Value Preservation: If the LHS is `mutable`, the resulting `NODE_INDEX_ACCESS` or `NODE_DEREF` acts as a valid Target for the Flow (`<-`) operator.

Semantic Interpretation:
- Positional Access: "Retrieve the element at this specific offset within the identified structure or memory block."
- Pointer Arithmetic (Sugar): "Treat this pointer as the base of a contiguous sequence and access the N-th element."
- Bounds-Verified Fetch: "Safely extract a value from this array, ensuring the requested index is within the defined limits."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Dispatched via the LED (Left Denotation) handler (Precedence 92).
  - Balanced Capture: The handler consumes the `[` token, calls `parse_expression(p, 0)` to capture the index (allowing full math/logic inside), and then strictly expects the closing `]`.
- Pass 4 (Semantic Validation):
  - Inspects the LHS Type.
    - if `array`: Verifies the index is a valid integer type and flags for a bounds check.
    - if `pointer`: Calculates the memory offset based on the `sizeof` the pointed-to type.
    - if neither: "Throws" a "Type Not Indexable" error.
- Pass 6 (Emitter):
  - For `array`: Emits a range-check instruction (or branch) followed by a `getelementptr` in LLVM
  - For `pointer`: Emits a direct `getelementptr` followed by a `load` or `store`


#### `.`

Definition:
- This is the "member access" operator. As a left-associative operator of a decently high precedence it is used to "select/navigate" the named members of a piece of structured data for access purposes without "overloading" the raw "index operator" (`[...]`) by making it have to handle non-integer indexes.

Operator Contract:
- Surface:
  - LHS: A symbol or structure with a resolved "composite" or "module" type.
  - RHS: A `NODE_IDENTIFIER` representing the named member, field or method.
  - Result: A direct reference to the identified member within the LHS context.
- Compiler:
  - LHS: `NODE_IDENTIFIER` | `NODE_STRUCTURED` | `NODE_CALL_IMM` | `NODE_ACCESS`
  - RHS: `NODE_IDENTIFIER` (Strictly).
  - Result: `NODE_ACCESS` -- resolves to the member's specific node type (excluding control-flow or "Meta" nodes like `NODE_EXIT`)
  
Architectural Contract:
- Static Resolution: Member shape information is expected from Pass 2 (Import/Module); final compatibility and visibility checks are enforced in Pass 4 (Semantic Validation).
- The "Dot-vs-Bracket" Rule: The dot operator is reserved for symbolic offsets (names); the bracket operator is reserved for numeric offsets (indices).
- Encapsulation Boundary: This operator respects visibility metadata. Accessing a member not marked `public` from an external module produces a semantic error.
- Chainability: Being left-associative, `a.b.c` is parsed as `(a.b).c`, allowing deep navigation of nested structures.

Semantic Interpretation:
- Namespace Navigation: "Drill down into this entity's internal hierarchy to find a specific sub-identity."
- Field Extraction: "Access the named value stored within this structured record."
- Path Resolution: "Identify the specific logic or data residing at this symbolic coordinate."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Dispatched via the LED (Left Denotation) handler (Precedence 85).
  - Strict RHS: The handler consumes the `.` token and strictly expects a single `NODE_IDENTIFIER`. It does not recurse into `parse_expression` for the RHS to prevent ambiguous forms like `a.(b)`
- Pass 4 (Semantic Validation):"
  - Verifies the LHS is an entity with "Members" (a struct, module or package).
  - Performs a lookup in the LHS's internal symbol table (often loaded from an NBT `TAG_Compound` in a `.tch` file).
  - Calculates the memory offset or logic-pointer for the Emitter.
- Pass 6 (Emitter):
  - Emits the appropriate LLVM `getelementptr` for data fields or a direct symbol reference for callable members. 

#### `,`

Definition:
- A very low-precedence, left-associative infix operator used to compose multiple expressions into a single, ordered aggregate.
- It is the fundamental builder of Contexts and Templates, facilitating the transition from individual values to a `NODE_STRUCTURED`.

Operator Contract:
- Surface:
  - LHS: Any valid expression or `NODE_FLOW` (`<-`)
  - RHS: Another valid expression or `NODE_FLOW`
  - Result: Pass 1 result is a `NODE_LIST`; enclosing constructs may reinterpret/promote it as `NODE_STRUCTURED`.
- Compiler:
  - LHS: (Strictly) any node that resolves to an expression value
  - RHS: (Strictly) any node that resolves to an expression value
  - Result: `NODE_LIST` (A collection of siblings)

Architectural Contract:
- Binding Priority: Because its precedence is so low (5), it allows high-precedence operators like `<-` (Flow) to resolve fully. In `x <- 1, y <- 2`, the comma only "sees" the two completed bindings.
- The "Structural" Role: When found inside parenthetical boundaries `( ... )`, the comma-delimited list is promoted to a `NODE_STRUCTURED`, forming the "Contract" for functions or data records.
- Variadic Support: In macro definitions, the comma is the delimiter that separates the elements of a variadic "node-array".

Semantic Interpretation:
- Aggregation: "Collect the individual pieces of data/logic into a single container."
- Position Sequencing: "Define an ordered sequence of members where position implies relationship."
- Context Construction: "Build the environment required for a subsequent mapping (`->`) or activation (`<=>` or `[=]`)."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Dispatched via the LED (Left Denotation) handler (Precedence 5).
  - Recursive Collection: It takes the `left` node and calls `parse_expression(p, 5)` for the RHS. This builds a left-leaning tree of nodes that can be flattened into a list or structure.
- Pass 4 (Semantic Validation):
  - Verifies that the resulting list is compatible with its target (e.g., matching the parameter count of a function).

#### `;`

Definition:
- The final "Hard Stop" of a Turned-C expression.
- It defines the boundary of a complete transaction, signaling to the parser that the current tree is finished and should be committed to the AST.

Operator Contract:
- Surface:
  - LHS: A fully resolved expression or definition.
  - RHS: None (Post-fix/Terminator form).
  - Result: A completed expression node ready for the next pass (nominally Pass 2).
- Compiler:
  - LHS: Any node that resolves to an expression value.
  - RHS: None.
  - Result: The `NODE_xxx` identity that best represents the expression.

Architectural Contract:
- The Zero-Gravity Rule: With a precedence of 0, the semicolon is the only "stand-alone" operator that cannot be "captured" by any other. It forces the `parse_expression` loop to exit and return the accumulated `left` node.
- Scope Finalization: As was discussed with the "Genesis Marker" (`#`), the semicolon acts as the "Baking Agent" -- it closes the window for immediate metadata mutation and finalizes the symbol table entry.
- Sequence Point: In the emitter, the semicolon marks a deterministic sequence point where all side effects from the preceding expression must be resolved.

Semantic Interpretation:
- Completion: "This logical transformation is complete; finalize the result."
- Boundary: "Separate this discrete instruction from the next in the instruction stream."
- Commit: "The contract described by the preceding nodes is now signed and sealed."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Loop Breaker: It is not usually handled as a LED in the traditional sense, but the Exit Condition for the `while` loop. When `peek()` sees `;`, the precedence check `0 > min_precedence` fails, the loop ends and the parser consumes the `;` before returning the tree.
- Pass 5 (Metadata):
  - Triggers the final "Packaging" of any symbols defined within the terminated statement into the module's NBT metadata.

#### `=>`

Definition:
  - The Destructuring Operator is a medium-low precedence, right-associative infix operator (Precedence 15).
  - It acts as the "Sieve" of the refinery, taking a complex resultant value from the left and mapping its internal members into a specific pattern or structure on the right.

Example:
```turned-c
define #data: structure <-> (a: integer, b: integer, c: integer) ;
(10, 20, 30) <=> f => (a: integer, b: integer) -> data;
```

Operator Contract:
- Surface:
  - LHS: A value-producing expression (typically a NODE_CALL_IMM from <=>).
  - RHS: A Pattern Template (a NODE_STRUCTURED or a NODE_INJECT template).
  - Result: A structured aggregate mapped to the right-hand shape.
- Compiler:
  - LHS: ANY_VALUE_NODE
  - RHS: NODE_STRUCTURED | NODE_INJECT
  - Result: NODE_DESTRUCTURE
  
Architectural Contract:
  - The "Sieve" Rule: The RHS defines the "Filter." If the LHS contains more data than the RHS requests, the excess is discarded. If the LHS contains less, it is a Contract Violation (Semantic Error).
  - Type Transformation: `=>` can perform implicit "shaping." In the given example, it takes the results of f, ensures they match the (a: integer, b: integer) pattern, and then maps them into the #data structure.
  - Precedence Logic: At 15, it correctly binds after the function has executed (20) but before the result is associated with a new global name (10).

Semantic Interpretation:
  - Structural Extraction: "Take the result of this work and pull out exactly these named pieces."
  - Shape Alignment: "Ensure the output of this logic fits into this specific data-template."
  - Data Refining: "Discard the noise from the execution and keep only the signal required for the next stage."

Technical Behavior:
  - Pass 1 (Pratt/Parsing):
    - Dispatched via the LED (Left Denotation) handler (Precedence 15).
    - Right-Associative: It recurses into parse_expression(p, 15) to allow the RHS pattern to resolve its own internal structure fully before being captured.
  - Pass 4 (Semantic Validation):
    - Performs the Arity and Type Check. It compares the "Output Signature" of the LHS (the function's return structure) against the "Input Signature" of the RHS pattern.
  - Pass 6 (Emitter):
    - Generates the mapping code to move values from the execution registers/stack into the structured memory block defined by the RHS.

#### `&&`

Definition:
  - Logical AND (short-circuit): evaluates to `true` only when both operands evaluate to `true`; otherwise `false`.
  - Evaluation Rule: The RHS is evaluated only if the LHS is `true`.
  - In an alternation chain (`condition ? && expr`), `&&` is a guard-branch operator that consumes the condition established by the preceding `?` operator rather than introducing its own local LHS operand.
  - The branch expression after `&&` is evaluated only when that upstream condition resolves to `true`.

Operator Contract:
Operator Contract:
- Surface:
  - Logical Form (`a && b`):
    - LHS: boolean expression.
    - RHS: boolean expression.
    - Result: boolean.
  - Alternation-Guard Form (`condition ? && expr`):
    - LHS: none (local); consumes the active condition context created by the preceding `?`.
    - RHS: guarded branch expression.
    - Result: branch-context continuation (not a standalone boolean result).
- Compiler:
  - Logical Form:
    - LHS: ANY_BOOLEAN_NODE
    - RHS: ANY_BOOLEAN_NODE
    - Result: `NODE_BOOL`
  - Alternation-Guard Form:
    - Context: `NODE_ALTERNATION`
    - RHS: ANY_VALUE_NODE
    - Result: `NODE_ALTERNATION` (guarded branch registered into the active alternation context)
  
Architectural Contract:
  - Short-Circuit Invariant: The RHS is strictly never evaluated if the LHS state fails the gate (`false` for `&&`). This is a Mandatory Directive for the Emitter (Pass 6).
  - Contextual Consumption: When following a `?`, the operator does not "lookup" a new LHS; it binds directly to the Branch Registry established by the `?`.

Technical Behavior:
  - Pass 1 (Pratt/Parsing):
    - Dispatched via the LED (Left Denotation) handler (Precedence 35).
    - Recursive Branching: It captures the `left` (the "condition") and calls `parse_expression(p, 35)` for the `right` to allow complex expressions within the gate.
    - Dual Parse Mode Rule:
      - Standard LED Mode: If no active alternation context exists, parse as a normal binary logical operator.
      - Alternation Context Mode: If a `?` has established an active `NODE_ALTERNATION` context, parse `&&`/`||` as branch operators that consume that context rather than introducing a new standalone boolean LHS.
      - Cross-reference:
        - The `&&` (guard) and `||` (choice) operators are most commonly used in conjunction with the `?` (alternation) operator. In this context, `?` establishes a branch point, while `&&` and `||` define the guarded and alternative execution paths, respectively. See the documentation for `?` for details on alternation and branch context.
  - Pass 4 (Semantic):
    - plain logical mode: boolean type enforcement
    - alternation mode: branch compatibility/type-join rules and valid `?` context checks
    - Branch Type Join: In alternation-choice form, `lhs` and `rhs` must resolve to a compatible common type (or require an explicit cast).
  - Pass 6 (Emitter):
    - Maps to an LLVM Condition Branch (`br`) and a Phi Node.
    - Jump to RHS if `true`; else jump to Exit.

#### `||`

Definition:
  - Logical OR (short-circuit): evaluates to `true` when either operand evaluates to `true`; evaluates to `false` only when both are `false`.
  - Evaluation Rule: The RHS is evaluated only if the LHS is `false`.
  - In an alternation chain (`condition ? lhs || rhs`), `||` acts as the choice split between true-path and false-path expressions.

Operator Contract:
- Surface:
  - Logical Form (`a || b`):
    - LHS: boolean expression.
    - RHS: boolean expression.
    - Result: boolean.
  - Alternation-Choice Form (`condition ? lhs || rhs`):
    - LHS: true-path expression (selected when upstream condition is `true`).
    - RHS: false-path expression (selected when upstream condition is `false`).
    - Result: selected branch value.
- Compiler:
  - Logical Form:
    - LHS: ANY_BOOLEAN_NODE
    - RHS: ANY_BOOLEAN_NODE
    - Result: `NODE_BOOL`
  - Alternation-Choice Form:
    - Context: `NODE_ALTERNATION`
    - LHS: ANY_VALUE_NODE
    - RHS: ANY_VALUE_NODE
    - Result: `NODE_ALTERNATION` (resolved to the selected branch value during semantic/emission)

Architectural Contract:
  - Short-Circuit Invariant: The RHS is strictly never evaluated if the LHS state fails the gate (`true` for `||`). This is a Mandatory Directive for the Emitter (Pass 6).
  - Contextual Consumption: When following a `?`, the operator does not "lookup" a new LHS; it binds directly to the Branch Registry established by the `?`.

Technical Behavior:
  - Pass 1 (Pratt/Parsing):
    - Dispatched via the LED (Left Denotation) handler (Precedence 35).
    - Recursive Branching: It captures the `left` (the "condition") and calls `parse_expression(p, 35)` for the `right` to allow complex expressions within the gate.
    - Dual Parse Mode Rule:
      - Standard LED Mode: If no active alternation context exists, parse as a normal binary logical operator.
      - Alternation Context Mode: If a `?` has established an active `NODE_ALTERNATION` context, parse `&&`/`||` as branch operators that consume that context rather than introducing a new standalone boolean LHS.
      - Cross-reference:
        - The `&&` (guard) and `||` (choice) operators are most commonly used in conjunction with the `?` (alternation) operator. In this context, `?` establishes a branch point, while `&&` and `||` define the guarded and alternative execution paths, respectively. See the documentation for `?` for details on alternation and branch context.
  - Pass 4 (Semantic):
    - plain logical mode: boolean type enforcement
    - alternation mode: branch compatibility/type-join rules and valid `?` context checks
    - Branch Type Join: In alternation-choice form, `lhs` and `rhs` must resolve to a compatible common type (or require an explicit cast).
  - Pass 6 (Emitter):
    - Maps to an LLVM Condition Branch (`br`) and a Phi Node.
    - Jump to RHS if `false`; else jump to Exit.

#### `?`

Definition:
  - The "Decision Anchor"
  - Alternation operator: evaluates a left-hand condition (must resolve to `boolean`) and routes control to a subsequent guarded/choice branch expression.
  - Structural Role: `?` establishes a branch context consumed by following `&&` (guard path) and/or `||` (choice path) forms.
  - See `&&` and `||` for the two defined branch forms enabled by alternation.

Operator Contract:
- Surface:
  - LHS: A boolean expression (The Predicate).
  - RHS: None (It is a marker for the following Gates).
  - Result: An active "Branch Context".
- Compiler:
  - LHS: `NODE_PREDICATE_LAMBDA` | `NODE_BOOL`.
  - Result: `NODE_ALTERNATION`

Architectural Contract:
  - Contextual Genesis: The `?` operator is the only mechanism that can transition the parser into "Alternation Mode".
  - Predicate Integrity: The LHS must resolve to a `boolean` primitive. Non-boolean types trigger a Pass 4 (Semantic) error.
  - Structural Requirement: A `?` that is not followed by at least one `&&` or `||` is a "Dangling Decision" (Parse Error).

Semantic Interpretation:
  - Path Genesis: "Begin a conditional transformation; prepare to divert flow based on this truth-state."
  - Valve Initiation: "Open a pressure-check on this expression to determine which pipe is active."
  - Logic Branching: "Evaluate this condition and prepare to route control-flow to the matching guard or choice branch."

Technical Behavior:
  - Pass 1 (Pratt/Parsing):
    - Dispatched via the LED (Left Denotation) handler (Precedence 82).
    - Context Injection: Sets a parser-state flag (or pushes to a stack) indicating an active NODE_ALTERNATION.
    - Wait State: Returns the alternation node, allowing the Pratt loop to continue and find the lower-precedence (35) `&&` or `||` tokens.
  - Pass 4 (Semantic Validation):
    - Verifies the LHS is boolean.
    - Ensures the context was closed correctly (i.e., it found its gates before expression termination or closing delimiter).

#### `{( ... )}`

Definition:
  - The Structural Unquote / Splice Operator.
  - A high-precedence prefix form used only inside macro-expansion contexts.
  - It extracts a quoted AST fragment from a node-bearing identifier and splices that fragment directly into the surrounding quoted structure.

Operator Contract:
- Surface: `{( node_identifier )}`
  - LHS: None.
  - RHS: An identifier bound to a quoted AST/node value in the current macro-expansion environment.
  - Result: The contained AST subtree, inserted in-place at the expansion site.

Architectural Contract:
  - Macro-Only Rule: `{( ... )}` has no runtime meaning. It is valid only during macro expansion and must not survive into ordinary semantic analysis as an executable operator.
  - Splice Rule: The result is not "evaluated"; it is structurally inserted into the surrounding AST.
  - Node Fidelity Rule: The inserted subtree preserves its existing node shape. It is not reparsed, normalized, or coerced merely because it was unquoted.
  - Lexical Boundary Rule: `{(` and `)}` are treated as distinct structural delimiters for macro syntax and must not be merged into surrounding operator text.

Technical Behavior:
  - Pass 1 (Lex / Parse): The lexer recognizes `{(` and `)}` as distinct tokens; the parser builds a `NODE_UNQUOTE` wrapper around the enclosed identifier reference.
  - Pass 3 (Macro Expansion): The expander resolves the identifier in macro scope, verifies that it denotes a quoted AST value, and replaces the `NODE_UNQUOTE` wrapper with the referenced subtree.
  - Pass 4+ (Semantic / Emit): No unquote operator remains; later passes operate on the spliced tree as if it had appeared in that position originally.

## Computational Operators

These are the arithmetic, comparison, boolean, identity, and bit-level operators currently represented in the Pratt table.

### Unary / Prefix

| Operator | Precedence | Name | Meaning |
| --- | ---: | --- | ---: | ---: |
| `*` | 75 | Dereference | Prefix NUD; accesses the value stored at a `pointer` address. |
| `&` | 75 | Make reference/Address-of | Prefix NUD; returns a `pointer` to the RHS operand | 
| `!` | 70 | Logical not | Prefix NUD; inverts the `boolean` state of the operand. |
| `~` | 70 | Bitwise not | Prefix NUD; performs a bitwise complement (inversion) on the numeric operand |
| `+` | 70 | Unary plus | Prefix NUD; signals that the following numeric constant is positive |
| `-` | 70 | Unary minus | Prefix NUD; signals that the following numeric constant is negative |

#### `*`

Definition:
  - The Dereference Operator accesses the value stored at the memory address held by a `pointer`.
    - Type Resolution: The resulting value's type is determined by the pointer's declared underlying type.
    - Address Following: Dereferencing retrieves the data at the target address; the pointer itself remains unchanged.
    - Validity Constraint: The pointer must be valid (non-null, properly aligned, pointing to live data). Dereferencing an invalid pointer is undefined behavior.
    - Mutability Preservation: If the underlying pointer is `mutable`, the dereferenced l-value is also mutable and can serve as the target of a Flow (`<-`) operation.
    - Function Pointers: When applied to a function pointer, dereferencing yields the callable function; however, in most contexts, function pointers may be called directly without explicit dereference.

Operator Contract:
- Surface:
  - Operand: A symbol, member or expression of a `pointer` type
  - Result: An l-value representing the data at the address held by the operand.
- Compiler:
  - Operand: `NODE_IDENTIFIER` | `NODE_ACCESS` | `NODE_INDEX_ACCESS` (Type must contain the `pointer` modifier)
  - Result: `NODE_DEREF` (inherits the base type from the pointer's type specification)

Architectural Contract:
- Type Extraction: The resulting node inherits the base type of the pointer. A `mutable integer pointer` yields a `mutable integer` l-value.
- Identity Maintenance: Dereferencing a pointer to a structure (`structure pointer`) allows immediate subsequent navigation via the Member Access (`.`) operator.
- Memory Safety (Stage 0.1+): While raw pointers allow undefined behavior, symbols flagged with specific safety capabilities may trigger implicit null-checks during dereference.

Semantic Interpretation:
- Value Extraction: "Follow this memory address to its destination and retrieve the identity stored there."
- Indirection Resolution: "Resolve the link between a location in memory and the actual data it represents."
- Target Acquisition: "Identify the physical memory slot required for a subsequent Flow (`<-`) operation."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Dispatched via the NUD (Null Denotation) handler (Precedence 75).
  - Prefix Capture: Consumes the `*` and calls `parse_expression(p, 75)` to capture the pointer expression.
- Pass 4 (Semantic Validation):
  - Verifies the operand carries the `pointer` type-modifier.
  - Ensures the base type is not `void` (unless the result is immediately cast or used in a raw-memory context).
- Pass 6 (Emitter):
  - Emits an LLVM `load` instruction for r-values.
  - For l-values (targets of `<-`), the dereferenced address becomes the target of a `store` instruction.
  - Function Calls: If the target is a function pointer, it prepares an indirect `call`.

#### `&`

Definition:
  - The `Address-of` Operator. It retrieves the memory location (`pointer`) of a specific identity.
  - Identity Capture: It transitions a value or symbol from a "Stored Data" state to a "Location Reference" state.
  - Resulting Type: The operation always yields a `pointer` to the operand's base type.
  - Constraint: The operand must be an L-Value (a symbol, member or index) that possesses a physical memory address. You cannot take the address of a raw numeric constant or a temporary result (R-Value).

Operator Contract:
- Surface:
  - Operand: a `mutable` or `immutable` symbol, member or index.
  - Result: a `pointer` to the operand's identity.
- Compiler:
  - Operand: `NODE_IDENTIFIER` | `NODE_ACCESS` | `NODE_INDEX_ACCESS`.
  - Result: `NODE_REF` (adds the `pointer` modifier to the operand's type).

Example:
```turned-c
define #ptr: integer pointer <-> &x;
define #member_ptr: integer pointer <-> &obj.field;
```

Architectural Contract:
  - Static vs. Dynamic: For global symbols, this results in a constant address. For stack-based symbols, it generates a relative offset.
  - Capability Propagation: Taking the address of a `mutable` item yields a `mutable pointer`.
  - Memory Residency: Forces the compiler to ensure the operand is "Addressable" (e.g., preventing it from being stored solely in a CPU register).

Semantic Interpretation:
  - Location Discovery: "Identify the physical coordinates of this data within the system's memory map."
  - Reference Creation: "Generate a link that can be passed, stored, or deferred for later access."
  - Pointer Origin: "Transition from a direct value to an indirect reference."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Dispatched via the NUD (Null Denotation) handler (Precedence 75).
- Pass 4 (Semantic Validation):
  - Verifies the operand is a valid L-Value.
- Pass 6 (Emitter):
  - For globals: emits a reference to the symbol's global address.
  - For locals: emits an LLVM `getelementptr` or equivalent stack frame offset.

#### `!`

Definition:
  - The Inversion Operator (Precedence 70, Prefix NUD).
  - It flips the truth-state of a boolean identity.
  - Strict Typing: In Turned-C, `!` only operates on the boolean primitive. It does not perform "C-style" integer-to-boolean coercion unless an explicit cast or predicate is used.

Operator Contract:
- Surface:
  - Operand: A boolean expression.
  - Result: The inverse boolean value.
- Compiler:
  - Operand: ANY_VALUE_NODE (type must resolve to boolean — enforced in Pass 4).
  - Result: `NODE_BOOL`

Architectural Contract:
  - Truth Inversion: `true` becomes `false`; `false` becomes `true`.
  - Pure Transformation: Does not mutate the operand; yields a new resultant value.

Semantic Interpretation:
  - Negation: "Assert the opposite of this condition."
  - Logical Flip: "Transition this truth-state to its polar opposite."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): Dispatched via NUD (75/70).
  - Pass 4 (Semantic Validation): Verifies the operand type is strictly boolean.
  - Pass 6 (Emitter): Emits an LLVM xor with true or a specific logical-not instruction.

#### `~`

Definition:
- The Bitwise Complement Operator (Precedence 70, Prefix NUD).
- Performs a bit-by-bit inversion of a numeric identity.

Operator Contract:
- Surface:
  - Operand: An integer or bit-field.
  - Result: A numeric value with all bits inverted (1s to 0s, 0s to 1s).
- Compiler:
  - Operand: ANY_VALUE_NODE (resolving to integer — enforced in Pass 4).
  - Result: `NODE_BITWISE`.

Architectural Contract:
  - Bitwise Fidelity: The operation must be performed on the raw binary representation of the operand. For signed integers, this typically results in the one's complement, where `~x` is mathematically equivalent to `-(x + 1)`.
  - Pure Transformation: Like all unary math operators in Turned-C, this is a non-mutating operation. It yields a new resultant value and does not modify the original symbol in memory.
  - Width Preservation: The resulting value must maintain the exact bit-width of the operand (e.g., a `u8` operand yields a `u8` result) to prevent unintended type promotion during bitwise logic.

Semantic Interpretation:
  - Bit-Flip: "Invert every individual bit within this numeric representation."
  - Complement: "Yield the one's complement of this value."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Dispatched via the NUD (Null Denotation) handler (Precedence 70).
  - Prefix Capture: Consumes the `~` token and calls `parse_expression(p, 70)` to capture the following numeric expression.
- Pass 4 (Semantic Validation):
  - Verifies the operand is of a numeric or bit-field type.
  - Ensures the operation is not being applied to a boolean (which requires `!`) or a pointer.
- Pass 6 (Emitter):
  - Emits an LLVM `xor` instruction between the operand and a bitmask of all ones (`-1` or `0xFF...`) of the same bit-width.

#### `+`/`-`

Definition:
  - `+` (Unary Plus, Precedence 70, Prefix NUD): An identity operation on a numeric value. It asserts explicit positivity but does not alter the value or type of the operand. The compiler elides it entirely; it leaves no trace in the emitted code.
  - `-` (Unary Minus, Precedence 70, Prefix NUD): Performs arithmetic negation. For integers, this is two's complement negation (`0 - x`). For floats, this flips the sign bit.
    - Constant Folding: When applied to a numeric literal, the negation is resolved at compile time. `-100` is not "negate 100 at runtime"; it is the constant value `-100`.
    - Unsigned Caution: Applying `-` to an unsigned integer type is technically defined (two's complement wraparound) but is a semantic warning — almost always a programmer error.

Operator Contract:
- Surface:
  - Operand: A numeric expression (integer or floating-point).
  - Result (`+`): The operand, unchanged.
  - Result (`-`): The arithmetic negation of the operand.
- Compiler:
  - Operand: ANY_VALUE_NODE (type must resolve to a numeric primitive — enforced in Pass 4).
  - Result (`+`): The operand node itself; no wrapping node is emitted.
  - Result (`-`): `NODE_MATH` (carrying the `negate` operation).

Architectural Contract:
  - Identity Rule (`+`): Unary plus is a compile-time-only annotation. It must not generate any runtime instruction and must not alter the type or value of the operand.
  - Negation Semantics (`-`): For integers, negation is defined as two's complement (`0 - x`). For floats, it is a sign-bit flip. The result preserves the bit-width and type of the operand.
  - Constant Elision: When the operand is a compile-time constant, the negation is folded into the constant and no runtime instruction is emitted.
  - Unsigned Warning: Negating an operand of an unsigned integer type is a semantic warning (not an error) — the result is well-defined by two's complement but is almost certainly unintentional.

Semantic Interpretation:
  - Explicit Sign (`+`): "Assert that this value is to be treated as a positive quantity."
  - Inversion (`-`): "Yield the additive inverse of this numeric identity."
  - Constant Integration (`-`): "Fold this sign into the literal value at compile time, producing a single signed constant."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Both dispatched via the NUD (Null Denotation) handler (Precedence 70).
  - Prefix Capture: Consumes the `+` or `-` token and calls `parse_expression(p, 70)` to capture the following numeric expression.
- Pass 4 (Semantic Validation):
  - Verifies the operand resolves to a numeric type.
  - For `+`: Marks the node for elision.
  - For `-`: Issues a warning if the operand type is an unsigned integer.
- Pass 6 (Emitter):
  - For `+`: No instruction emitted; the operand value is forwarded directly.
  - For `-` (integer): Emits LLVM `sub i<width> 0, <operand>`.
  - For `-` (float): Emits LLVM `fneg <operand>`.
  - Constant operands are folded at compile time regardless of operator; no runtime instruction is emitted.

### Dual Prefix/Postfix

| Operator | Precedence | Name | Meaning |
| --- | ---: | --- | ---: | ---: |
| `++` | 85 | Pre-increment | Prefix NUD/Posftfix LED; the incrementor |
| `--` | 85 | Pre-decrement | Prefix NUD/Posftfix LED; the decrementor |

#### `++`

Definition:
  - A general purpose increment operation that takes two forms.
    - Prefix:
      - (`++x`): mutate the target first, then yield the updated value.
    - Postfix:
      - (`x++`): capture the pre-mutation value of the target, then mutate the target and finally yield the captured value

Operator Contract:
- Surface:
  - Operand: a `mutable` symbol, member access or indexed l-value
    - Postfix Form: the operator appears on the right-hand side of the operand (`x++`)
    - Prefix Form: the operator appears on the left-hand side of the operand (`++x`)
  - Result: The value of the operand (either pre- or post-mutation, depending on fixity)
- Compiler:
  - Operand: `NODE_IDENTIFIER` | `NODE_ACCESS` | `NODE_INDEX_ACCESS` (must carry the `mutable` capability).
  - Result: `NODE_MATH` (carrying the `increment` operation directly and positioning the operand as the right child (prefix) or left child (postfix) within `NODE_MATH`'s tree. Fixity is thus self-documenting: the operand's position in the node structure encodes whether the operator was prefix or postfix.

Architectural Contract:
- This has two constraints:
  1) Single-read/single-write
     - Postfix must not permit duplicate evaluation of the target expression (this is required for member/indexed l-values to work correctly)
  2) Sequencing guarantee
     - The mutation is part of the operator's own evaluation; no reordering across the operator boundary.

Semantic Interpretation:
- Unit Advancement: "Increase the value of this numeric identity by exactly one unit of its defined type."
- Temporal Capture (Postfix): "Snapshot the current state for the immediate expression, then advance the state for all subsequent operations."
- Immediate Mutation (Prefix): "Advance the state of this identity before utilizing it in the current transformation."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Prefix: Dispatched via the NUD (Precedence 85). Captures the following expression unit.
  - Postfix: Dispatched via the LED (Precedence 85). Attaches to the preceding expression unit.
  - Validation: The parser ensures the operand is not a constant or a void result.
- Pass 4 (Semantic Validation):
  - Verifies the operand is mutable.
  - Checks if the type supports incrementing (integers, pointers). For pointers, the increment is scaled by sizeof(type).
- Pass 6 (Emitter):
  - For Prefix: Emits a load, add, and store in sequence, yielding the add result.
  - For Postfix: Emits a load (to a temporary register), add, and store, yielding the initial load value.
- Atomic Note (Stage 0.1+): If the operand is defined as `@[ atomic ]@`, the emitter must use an atomicrmw add instruction to satisfy the architectural contract.

#### `--`

Definition:
  - A general purpose decrement operation that takes two forms.
    - Prefix:
      - (`--x`): mutate the target first, then yield the updated value.
    - Postfix:
      - (`x--`): capture the pre-mutation value of the target, then mutate the target and finally yield the captured value

Operator Contract:
- Surface:
  - Operand: a `mutable` symbol, member access or indexed l-value
    - Postfix Form: the operator appears on the right-hand side of the operand (`x--`)
    - Prefix Form: the operator appears on the left-hand side of the operand (`--x`)
  - Result: The value of the operand (either pre- or post-mutation, depending on fixity)
- Compiler:
  - Operand: `NODE_IDENTIFIER` | `NODE_ACCESS` | `NODE_INDEX_ACCESS` (must carry the `mutable` capability).
  - Result: `NODE_MATH` (carrying the `decrement` operation directly and positioning the operand as the right child (prefix) or left child (postfix) within `NODE_MATH`'s tree. Fixity is thus self-documenting: the operand's position in the node structure encodes whether the operator was prefix or postfix.

Architectural Contract:
- This has two constraints:
  1) Single-read/single-write
     - Postfix must not permit duplicate evaluation of the target expression (this is required for member/indexed l-values to work correctly)
  2) Sequencing guarantee
     - The mutation is part of the operator's own evaluation; no reordering across the operator boundary.

Semantic Interpretation:
- Unit Reduction: "Decrease the value of this numeric identity by exactly one unit of its defined type."
- Temporal Retreat (Postfix): "Snapshot the current state for the immediate expression, then regress the state for all subsequent operations."
- Immediate Reduction (Prefix): "Reduce the state of this identity before utilizing it in the current transformation."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Prefix: Dispatched via the NUD (Precedence 85). Captures the following expression unit.
  - Postfix: Dispatched via the LED (Precedence 85). Attaches to the preceding expression unit.
  - Validation: The parser ensures the operand is not a constant or a void result.
- Pass 4 (Semantic Validation):
  - Verifies the operand is mutable.
  - Checks if the type supports decrementing (integers, pointers). For pointers, the decrement is scaled by sizeof(type).
- Pass 6 (Emitter):
  - For Prefix: Emits a load, sub, and store in sequence, yielding the sub result.
  - For Postfix: Emits a load (to a temporary register), sub, and store, yielding the initial load value.
- Atomic Note (Stage 0.1+): If the operand is defined as `@[ atomic ]@`, the emitter must use an atomicrmw sub instruction to satisfy the architectural contract.

### Arithmetic and Bit-Level Infix

| Operator | Precedence | Associativity | Meaning | Notes |
| --- | ---: | --- | --- | --- |
| `**` | 75 | right | Exponentiation | High-precedence math; typically lowers to an intrinsic or bootstrap math macro |
| `&` | 65 | left | Bitwise and | Performs a bit-by-bit intersection; preserves the bit-width of the numeric operands |
| `\|` | 65 | left | Bitwise or | Performs a bit-by-bit union; preserves the bit-width of the numeric operands |
| `^` | 65 | left | Bitwise exclusive-or | Performs a bitwise symmetric difference (XOR); frequently used for toggling or fast parity checks |
| `*` | 60 | left | Multiply | Basic arithmetic product; maps to the sign-agnostic machine instruction |
| `/` | 60 | left | Divide | Arithmetic quotient; Pass 4 selects `sdiv` or `udiv` based on the signedness metadata |
| `%` | 60 | left | Modulus | Arithmetic remainder; Pass 4 selects `srem` or `urem` based on the signedness metadata |
| `+` | 50 | left | Add | Arithmetic sum; also serves as the base for pointer-offset calculations |
| `-` | 50 | left | Subtract | Arithmetic difference; may require explicit whitespace in hyphen-glue contexts |
| `<<` | 45 | left | Shift left | Bitwise shift; fills vacated lower bits with zeros |
| `>>` | 45 | left | Shift right | Bitwise shift; Pass 4 selects `ashr` (arithmetic) or `lshr` (logical) based on signedness |
| `<<<` | 45 | left | Roll left | Circular bitwise rotation; bits shifted out of the high end reappear at the low end |
| `>>>` | 45 | left | Roll right | Circular bitwise rotation; bits shifted out of the low end reappear at the high end |

#### `**`

Definition:
  - The exponentiation operator (Precedence 75, Right-associative)
  - Performs the power operation, raising the LHS (base) to the power of the RHS (exponent).

Overload Resolution:
  - If either operand's type defines an overload for `**`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default exponentiation is performed.

Operator Contract:
- Surface:
  - LHS: Numeric expression (base).
  - RHS: Numeric expression (exponent).
  - Result: Exponential power result (`base ^ exponent`) in a numeric type determined by promotion/validation rules.
- Compiler:
  - LHS: ANY_VALUE_NODE (resolving to a numeric primitive)
  - RHS: ANY_VALUE_NODE (resolving to a numeric primitive)
  - Result: `NODE_MATH` (carrying the `pow` operation).

Architectural Contract:
  - Right-Associativity: Following the standard mathematical convention, `a ** b ** c` is parsed as `a ** (b ** c)`.
  - Type Promotion: If the base and exponent are different in bit-width, the result follows the standard Pass 4 promotion rules (typically promoting to the wider type).
  - Zero/Negative Exponents: Zero and negative exponent behavior is type-dependent.
    - Floating-point behavior follows IEEE 754.
    - Integer negative-exponent behavior is currently implementation-defined (TBD for standardization).

Semantic Interpretation:
  - Exponential Scaling: "Raise this base by the magnitude specified by the exponent."
  - Power Projection: "Project the base identity through an exponent transform."

Technical Behavior:
  - Pass 1 (Pratt/Parsing):
    - Dispatched via the LED (Left Denotation) handler (Precedence 75).
    - Right-Associative: Calls `parse_expression(p, 75)` to preserve `a ** b ** c` groups correctly as `a ** ( b ** c )`.
  - Pass 4 (Semantic Validation):
    - Verifies both operands are numeric.
    - Constant Folding: If both LHS and RHS are literals, the result is calculated at compile-time and replaced with a single `NODE_CONSTANT`.
  - Pass 6 (Emitter):
    - For floats: Maps to LLVM `llvm.pow.*` intrinsic.
    - For integers: Lowers to a bootstrap integer-power routine or backend-specific integer-power implementation.

#### `&`

Definition:
  - The Bitwise AND (`Intersection`) Operator (Precedence 65, Left-Associative).
  - Performs a bit-by-bit logical AND between two numeric operands.

Overload Resolution:
  - If either operand's type defines an overload for `&`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default bitwise AND is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Integral numeric expressions (integers or bit-fields).
  - Result: A numeric value where each bit is `1` only if the corresponding bits in both operands are `1`.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (must resolve to integral numeric/bit-field types).
  - Result: `NODE_BITWISE` (tagged as `AND`).

Architectural Contract:
  - Masking Role: Primarily used to isolate specific bits or mask out unwanted data.
  - Width Unification: Operands may differ in width; Pass 4 applies integer-promotion rules to a common width before emission.
  - Width Preservation: The emitted operation preserves the unified operand width.

Semantic Interpretation:
  - Filtering: "Extract the commonality between these two bit-patterns."
  - Intersection: "Yield the overlapping bits of the LHS and RHS."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Dispatched via the LED handler (Precedence 65).
  - Left-Associative: Calls `parse_expression(p, 66)` for RHS capture.
- Pass 4 (Semantic Validation):
  - Verifies both operands are integral/bit-field types.
  - Rejects non-bitwise-compatible types (e.g., float, pointer, boolean unless explicitly cast).
  - Applies width/signedness normalization per integer-promotion rules.
- Pass 6 (Emitter):
  - Lowers to the LLVM `and` instruction.
  - If both operands are compile-time constants, emits a folded constant instead of a runtime instruction.

#### `|`

Definition:
  - The Union Operator (Precedence 65, Left-Associative).
  - Performs a bit-by-bit logical OR; a bit is `1` if either corresponding operand bit is `1`.

Overload Resolution:
  - If either operand's type defines an overload for `|`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default bitwise OR is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Integral numeric expressions (integers or bit-fields).
  - Result: A numeric value where each bit is `1` if either corresponding operand bit is `1`.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (must resolve to integral numeric/bit-field types).
  - Result: `NODE_BITWISE` (tagged as `OR`).

Architectural Contract:
  - Inclusive Merge Role: Primarily used to combine feature flags, capability masks, or partial bit-sets into one aggregate state.
  - Width Unification: Operands may differ in width; Pass 4 applies integer-promotion rules to a common width before emission.
  - Width Preservation: The emitted operation preserves the unified operand width.

Semantic Interpretation:
  - Accumulation: "Combine these bit-patterns into a single inclusive state."
  - Union: "Yield all bits present in either the LHS or the RHS."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Dispatched via the LED handler (Precedence 65).
  - Left-Associative: Calls `parse_expression(p, 66)` for RHS capture.
- Pass 4 (Semantic Validation):
  - Verifies both operands are integral/bit-field types.
  - Rejects non-bitwise-compatible types (e.g., float, pointer, boolean unless explicitly cast).
  - Applies width/signedness normalization per integer-promotion rules.
- Pass 6 (Emitter):
  - Lowers to the LLVM `or` instruction.
  - If both operands are compile-time constants, emits a folded constant instead of a runtime instruction.

#### `^`
Definition:
  - The Symmetric Difference Operator (Precedence 65, Left-Associative).
  - Performs a bit-by-bit logical XOR; a bit is `1` if the corresponding bits differ.

Overload Resolution:
  - If either operand's type defines an overload for `^`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default bitwise XOR is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Integral numeric expressions (integers or bit-fields).
  - Result: A numeric value where each bit is `1` only when the corresponding operand bits differ.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (must resolve to integral numeric/bit-field types).
  - Result: `NODE_BITWISE` (tagged as `XOR`).

Architectural Contract:
  - Toggle Role: Primarily used for selective bit flipping, parity checks, and symmetric-difference masks.
  - Width Unification: Operands may differ in width; Pass 4 applies integer-promotion rules to a common width before emission.
  - Width Preservation: The emitted operation preserves the unified operand width.

Semantic Interpretation:
  - Toggle: "Flip the state of bits in the LHS based on the mask in the RHS."
  - Difference: "Identify where these two bit-patterns diverge."

Technical Behavior:
- Pass 1 (Pratt/Parsing):
  - Dispatched via the LED handler (Precedence 65).
  - Left-Associative: Calls `parse_expression(p, 66)` for RHS capture.
- Pass 4 (Semantic Validation):
  - Verifies both operands are integral/bit-field types.
  - Rejects non-bitwise-compatible types (e.g., float, pointer, boolean unless explicitly cast).
  - Applies width/signedness normalization per integer-promotion rules.
- Pass 6 (Emitter):
  - Lowers to the LLVM `xor` instruction.
  - If both operands are compile-time constants, emits a folded constant instead of a runtime instruction.

#### `*`

Definition:
  - The Arithmetic Product operator (Precedence 60, Left-Associative).
  - In a sign-agnostic machine (LLVM), this is the simplest of the trio.

Overload Resolution:
  - If either operand's type defines an overload for `*`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default arithmetic multiplication is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Numeric expressions.
  - Result: The mathematical product of the operands.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Numeric).
  - Result: `NODE_MATH` (tagged as `MUL`).

Architectural Contract:
  - Sign Agnosticism: In Two's Complement, the bit-pattern for multiplication is identical regardless of signedness.
  - Overflow Handling: By default, follows the "Wraparound" behavior of the target architecture.

Semantic Interpretation:
  - Product Composition: "Combine these magnitudes multiplicatively into a single result."
  - Scaling: "Scale the LHS by the factor represented by the RHS."

Technical Behavior:
  - Pass 1 (Pratt/Parse): LED at 60; calls `parse_expression(p, 61)`.
  - Pass 6 (Emitter):
    - Integer operands: emits LLVM `mul`.
    - Floating-point operands: emits LLVM `fmul`.

#### `/`

Definition:
   - The Arithmetic Quotient operator (Precedence 60, Left-Associative).
   - Performs the division of the LHS (dividend) by the RHS (divisor).

Overload Resolution:
  - If either operand's type defines an overload for `/`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default arithmetic division is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Numeric expressions.
  - Result: The quotient of the operation.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Numeric).
  - Result: `NODE_MATH` (tagged as `DIV`).

Architectural Contract:
  - Signedness Mandate: The compiler must resolve the signedness of the operands to select the correct machine instruction (sdiv vs udiv).
    - Floating-point operands bypass signedness selection and always emit fdiv.
  - Divide-by-Zero: In the Bootstrap Core, division by zero is undefined behavior at the hardware level; Pass 4 may attempt to catch literal-zero divisors at compile-time.

Semantic Interpretation:
  - Partitioning: "Distribute the LHS magnitude into the number of segments specified by the RHS."
  - Ratio Extraction: "Determine how many times the RHS fits wholly within the LHS."

Technical Behavior:
  - Pass 1 (Pratt): LED at 60; calls `parse_expression(p, 61)`.
  - Pass 4 (Semantic): Queries the Symbol Table for the signed metadata of the operand types.
  - Pass 6 (Emitter):
    - Constant Folding: If both operands are compile-time constants and the divisor is non-zero, folds to a single constant.
    - Integer (Signed): Emits LLVM `sdiv`.
    - Integer (Unsigned): Emits LLVM `udiv`.
    - Floating-point: Emits LLVM `fdiv`.

#### `%`

Definition:
  - The Arithmetic Remainder operator (Precedence 60, Left-Associative)
  - Performs the division of the LHS (dividend) by the RHS (divisor) and returns the remainder
  - For integers, this is the true machine-level remainder, not the mathematical modulo: the sign of the result matches the dividend (C/LLVM semantics).
  - For floating-point operands, computes the IEEE 754 floating-point remainder (not mathematical modulo).

Overload Resolution:
  - If either operand's type defines an overload for `%`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default remainder operation is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Numeric expressions.
  - Result: The remainder of the division.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Numeric).
  - Result: `NODE_MATH` (tagged as `MOD`).

Architectural Contract:
  - Signedness Mandate: The compiler must resolve the signedness of the operands to select the correct machine instruction (srem vs. urem).
    - Floating-point operands bypass signedness selection and always emit frem.
  - Divide-by-Zero: In the Bootstrap Core, division by zero is undefined behavior at the hardware level; Pass 4 may attempt to catch literal-zero divisors at compile-time.
  - Mathematical Note: The % operator in Turned-C (like C and LLVM) is a remainder, not a true modulo. For negative dividends, the result may be negative. If mathematical modulo is required (always non-negative), it must be implemented explicitly.

Semantic Interpretation:
  - Residue Extraction: "Calculate the remaining magnitude after a full partitioning of the LHS by the RHS."
  - Cyclic Mapping: "Map a linear value into a repeating range defined by the divisor."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): LED at 60; calls `parse_expression(p, 61)`.
  - Pass 4 (Semantic Validation):
    - Verifies both operands are numeric.
    - Queries the Symbol Table for the signed metadata of the operand types to determine the REM variant.
  - Pass 6 (Emitter):
    - Constant Folding: If both operands are compile-time constants and the divisor is non-zero, folds to a single constant.
    - Integer (Signed): Emits LLVM `srem`.
    - Integer (Unsigned): Emits LLVM `urem`.
    - Floating-point: Emits LLVM `frem`.

#### `+`

Definition:
  - The Arithmetic `sum` operator (Precedence 50, Left-Associative)
  - Performs the addition of the LHS and the RHS and returns the result.
  - Pointer Note: When one operand is a `pointer` and the other is an `integer`, the operator performs Pointer Arithmetic (Address Offsetting).

Overload Resolution:
  - If either operand's type defines an overload for `+`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default arithmetic or pointer addition is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Numeric expressions (Integers/Floats) OR one Pointer and one Integer.
  - Result: The arithmetic sum (Numeric) or a new Pointer to the offset address.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Numeric) | `NODE_REF` (Pointer).
  - Result: `NODE_MATH` (tagged as `ADD`).

Architectural Contract:
  - Overflow Policy:
    - Signed Integers: Overflow is Undefined Behavior (mapping to LLVM `nsw` in the reference implementation)
    - Unsigned Integers: Overflow follows Two's Complement Wraparound (mapping to LLVM `nuw` in the reference implementation)
    - Floats: Follows IEEE 754 addition rules.
  - Pointer Scaling: When adding to a pointer, the integer operand is automatically scaled by `sizeof(T)`, where `T` is the type the pointer points to.
  - Commutativity: Arithmetic addition is commutative `(a + b == b + a)`. Pointer-integer addition is also commutative in syntax, but the compiler must treat the pointer as the "Base" and the integer as the "Index."

Semantic Interpretation:
  - Aggregation: "Combine the magnitudes of two independent identities into a single unified result."
  - Positional Offset: "Advance the location of a memory reference by a specified discrete interval."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): LED at 50; calls `parse_expression(p, 51)`.
  - Pass 4 (Semantic Validation):
    - Verifies type compatibility (Numeric + Numeric, or Pointer + Integer).
    - Performs Width Unification: Promotes smaller integers to the width of the larger operand before emission.
  - Pass 6 (Emitter):
    - Constant Folding: If both operands are compile-time constants, folds to a single constant (`NODE_CONSTANT`).
    - Integer/Float: Emits LLVM `add` or `fadd`.
    - Pointer Arithmetic: Emits LLVM `getelementptr` (GEP) to perform the scaled offset calculation.

#### `-`

Definition:
  - The Arithmentic `subtraction` operator (Precedence 50, Left-Associative)
  - Performs the subtraction of the RHS from the LHS and returns the result.
  - Pointer Note: `-` serves as either a Linear Regression (Pointer - Integer) or a Distance Calculation (Pointer - Pointer)

Overload Resolution:
  - If either operand's type defines an overload for `-`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default arithmetic or pointer subtraction is performed.

Operator Contract:
- Surface:
  - LHS/RHS:
    - Numeric: (Integer - Integer) or (Float - Float)
    - Pointer Offset: (Pointer - Integer)
    - Pointer Difference: (Pointer - Pointer)
  - Result: 
    - Numeric: The arithmetic difference (subtraction)
    - Offset: A new Pointer at the offset address
    - Difference: The "number of elements" difference (scaled by `1/sizeof(T)`)
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Numeric) | `NODE_REF` (Pointer).
  - Result: `NODE_MATH` (tagged as `SUB`).

Architectural Contract:
  - The Air-Gap Requirement: Due to Hyphen-Glue lexing, the `-` operator must be separated by whitespace from adjacent `BAREWORD` identifiers to be parsed as an operator. Without whitespace, it is consumed as a part of a composite identifier.
  - Pointer Regression: Subtracting an integer from a pointer moves the address "backward" in memory, scaled by `sizeof(T)`, where `T` is the type the pointer points to.
  - Pointer Distance: Subtracting one pointer from another of the same type yields an integer representing the number of elements between them `(scaled by 1/sizeof(T))`. Subtraction between pointers of different types is a Pass 4 (Semantic) Error.
  - Overflow Policy: Matches the Addition (`+`) contract for nsw/nuw flags in LLVM.

Semantic Interpretation:
  - Diminution: "Reduce the magnitude of the LHS by the quantity specified in the RHS."
  - Positional Regression: "Regress the location of a memory reference by a specified discrete interval."
  - Interval Measurement: "Calculate the discrete distance between two established memory coordinates."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): LED at 50; calls `parse_expression(p, 51)`.
  - Pass 4 (Semantic Validation):
    - Verifies type compatibility: (Numeric - Numeric), (Pointer - Integer), or (Pointer - Pointer).
    - Type-Match Rule: For Pointer - Pointer, verifies both pointers share the same base type.
  - Pass 6 (Emitter):
    - Constant Folding: Resolves literal subtraction at compile-time.
    - Integer/Float: Emits LLVM `sub` or `fsub`.
    - Pointer - Integer: Emits LLVM `getelementptr` with a negated index.
    - Pointer - Pointer: Emits LLVM `ptrtoint` on both, `sub`, and then `sdiv` by the element size to find the count.

#### `<<`

Definition:
  - The "Bitwise Shift Left" operator (Precedence 45, Left-Associative)
  - Moves the bit-pattern of the LHS to the left by the number of positions specified by the RHS, filling vacated lower bits with `0`

Overload Resolution:
  - If the LHS type defines an overload for `<<`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default bitwise shift is performed.

Overload Resolution:
  - If the LHS type defines an overload for `<<`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default bitwise shift is performed.

Operator Contract:
- Surface:
  - LHS: The integer value or bit-field to be shifted.
  - RHS: An integer specifying the number of bit-positions to shift.
  - Result: The LHS value shifted left; bits shifted out of the high end are discarded.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Integer)
  - Result: `NODE_BITWISE` (tagged as `SHL`)

Architectural Contract:
  - Width Preservation: The operation is performed within the bit-width of the LHS. Shifting by a value equal to or greater than the LHS bit-width results in Undefined Behavior (mapping to LLVM's `shl` constraint in the reference implementation).
  - Arithmetic Equivalence: For non-overflowing values, `x << y` is mathematically equivalent to $x \times 2^{y}$.
  - Sign Agnosticism: Unlike right-shifts, the left-shift operation is identical for both signed and unsigned integers in two's complement.

Semantic Interpretation:
  - Magnitude Inflation: "Scale the bit-pattern upward by a power-of-two factor."
  - Bitwise Displacement: "Relocate the internal state of the LHS toward the higher-order bit-positions."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): LED at 45; calls `parse_expression(p, 46)`.
  - Pass 4 (Semantic Validation):
    - Verifies both operands are integral types.
    - Literal Check: If RHS is a constant, verifies it is within the bounds of the LHS bit-width (e.g., 0-63 for an i64).
  - Pass 6 (Emitter):
    - Constant Folding: Resolves literal shifts at compile-time.
    - Standard Emission: Emits the LLVM `shl` instruction or the equivalent for other backends.

#### `>>`

Definition:
  - The "Bitwise Shift Right` operator (Precedence 45, Left-Associative)
  - Moves the bit-pattern of the LHS to the right by the RHS positions; bits shifted out of the low end are discarded.
  - Signedness Rule: The "fill" for vacated upper bits depends on the type: Signed types preserve the sign bit (Arithmetic), while Unsigned types fill with zeros (Logical).

Overload Resolution:
  - If the LHS type defines an overload for `>>`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default bitwise shift is performed.

Overload Resolution:
  - If the LHS type defines an overload for `>>`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default bitwise shift is performed.

Operator Contract:
- Surface:
  - LHS: The integer value or bit-field to be shifted.
  - RHS: An integer specifying the number of bit-positions to shift.
  - Result: The LHS value shifted right, with upper-bit filling determined by the LHS type.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Integer)
  - Result: `NODE_BITWISE` (tagged as `SHR`)

Architectural Contract:
  - The Signedness Split:
    - Signed (i64): Performs Arithmetic Shift. The sign bit (MSB) is copied into all vacated positions. This preserves the "negativity" of the value (e.g., `-10 >> 1` remains negative).
    - Unsigned (u64): Performs Logical Shift. All vacated positions are filled with 0.
  - Width Constraint: Shifting by a value equal to or greater than the LHS bit-width results in Undefined Behavior.
  - Arithmetic Equivalence: For non-negative integers, $x \gg y$ is equivalent to $x/2^{y}$.

Semantic Interpretation:
  - Magnitude Deflation: "Scale the bit-pattern downward by a power-of-two factor while maintaining sign-integrity."
  - Logical Displacement: "Relocate the internal state toward the lower-order bit-positions."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): LED at 45; calls `parse_expression(p, 46)`.
  - Pass 4 (Semantic Validation):
    - Verifies both operands are integral types.
    - Literal Check: If the RHS is a constant, verifies it is within the range $[0, \text{bit\_width} - 1]$.
    - Metadata Lookup: Retrieves the signed flag from the LHS Typespec to determine the shift variant.
      - If the signedness metadata is missing (such as a type referencing the "Naked" for of LLVM::i64), the compiler should choose a safe default - the "Logical" shift (`lshr`) to prevent accidental sign-bit smearing.
  - Pass 6 (Emitter):
    - Constant Folding: Resolves literal shifts at compile-time.
    - Standard Emission:
      - For signed types emits the `ashr` LLVM primitive
      - For unsigned types emits the `lshr` LLVM primitive
      - For "naked" types (thost that lack signeness metadata) the `lshr` LLVM primitive shall be emitted. (See the note under the Pass 4 details for information on why)
  
#### `<<<` and `>>>`

Definition:
  - The "Circular Bitwise Rotation" operators (Precedence 45, Left-Associative)
  - `<<<`: Moves bits to the left; the MSB (`Most Significant Bit`) rolls around to the LSB (`Least Significant Bit`) position
  - `>>>`: Moves bits to the right; The LSB rolls around to the MSB position
  - Example: `0b1001 <<< 1` yields `0b0011` (for a 4-bit value)

Overload Resolution:
  - If the LHS type defines an overload for `<<<` or `>>>`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default rotation is performed.

Operator Contract:
- Surface:
  - LHS: The integer value to be rotated.
  - RHS: The number of bit-positions to rotate.
  - Result: The rotated bit-pattern; The bit-width of the result is strictly identical to the bit-width of the LHS.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Integer).
  - Result: `NODE_BITWISE` (tagged as `ROL` or `ROR`)

Architectural Contract:
  - Conservation of Bits: Unlike shifts, no bits are discarded and no "fill" (0 or sign) is required.
  - Modulo Rotation: Rotating a 64-bit integer by 64 positions results in the original value. The compiler should implicitly treat the RHS as `RHS % bit_width`. (e.g. `x <<< 64` on an `i64` is a no-op)
  - Sign Independence: The operation is bit-level and behaves identically for `signed` and `unsigned` types.

Semantic Interpretation:
  - Circular Permutation: "Relocate the bit-pattern within a closed loop."
  - Position Re-entry: "Ensure bits exiting one boundary of the identity re-enter at the opposite boundary."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): LED at 45; calls `parse_expression(p, 46)`.
  - Pass 4 (Semantic Validation):
    - Verifies both operands are integral.
  - Pass 6 (Emitter):
    - Constant Folding: Resolves literal rotations at compile-time.
    - Standard Emission: Maps to LLVM `fshl` (`Funnel Shift Left`) or `fshr` (`Funnel Shift Right`) intrinsics. To achieve a true rotation, the LHS value is supplied as both inputs to the funnel shift intrinsic.

### Comparison, Identity, and Boolean Infix

| Operator | Precedence | Associativity | Meaning | Notes |
| --- | ---: | --- | --- | --- |
| `===` | 40 | left | Identity check | Compares raw memory addresses or unique symbol IDs; ignores value-level equality. |
| `!==` | 40 | left | Inverted identity check | Inverted check; returns `true` if the two operands are physically distinct entities. |
| `==` | 40 | left | Equality | Value-level comparison; Pass 6 selects `icmp eq` or `fcmp oeq` based on type metadata. |
| `!=` | 40 | left | Inequality | Value-level disparity; returns `true` if the internal magnitudes of the operands differ. |
| `<` | 40 | left | Less than | Ordered comparison; Pass 6 selects `slt` (signed) or `ult` (unsigned) from the symbol metadata. |
| `<=` | 40 | left | Less than or equal | Ordered comparison; Pass 6 selects `sle` (signed) or `ule` (unsigned) variants. |
| `>` | 40 | left | Greater than | Ordered comparison; Pass 6 selects `sgt` (signed) or `ugt` (unsigned) variants. |
| `>=` | 40 | left | Greater than or equal | Ordered comparison; Pass 4 selects `sge` (signed) or `uge` (unsigned) variants |


#### `===`

Definition:
  - The "Formal Identity" operator (Precendence 40, Left-Associative)
  - Performs an "Identity-Level" comparison; returns `true` only if the LHS and RHS refer to the exact same memory location or unique symbolic instance.
  - Literal Note: Constant literals of identical value are treated as having a shared "Global Identity" in the Refinery (e.g., `100 === 100` is always `true`).

Operator Contract:
- Surface:
  - LHS/RHS: Any symbol, pointer, array or literal.
  - Result: A `boolean` value (`true` if the identities are indistinguishable, else `false`).
- Compiler:
  - LHS/RHS: ANY_NODE (identities must be addressable or comparable).
  - Result: `NODE_BOOL` (carrying the raw expression)
    - Note: it might be needed to add a `NODE_LOGICAL` to provide a clear distinction.

Architectural Contract:
  - Physicality Rule: For non-literals, identity is defined by the Base Address. Two different pointers to the same address are identical; two different variables with the same value are not identical.
  - Symbolic Integrity: In Pass 4, the compiler compares the Fat Pointer Metadata (`base` and `bounds`). If the provenance matches, the identity is confirmed.
  - Short-Circuit Identity: If the compiler can prove both operands refer to the same symbolic identity at compile-time, the operation is folded to `true` without a runtime check.
  - For arrays, structures, or compound types, identity is established only if both operands refer to the same storage instance (i.e., their base addresses and type metadata are identical). Value equivalence of contents does not constitute identity.

Semantic Interpretation:
  - Identity Discovery: "Determine if these two references are aliases for the same physical entity."
  - Singularity Check: "Prove that no transformation has occurred between the perception of the LHS and the RHS."

Technical Behavior:
  - Pass 1 (Pratt/Parse): LED at 40; calls `parse_expression(p, 41)`.
  - Pass 4 (Semantic Validation):
    - Verifies that both sides are "comparable" (not void).
    - Identity Folding: If both operands are the same `NODE_IDENTIFIER` or identical `NODE_CONSTANT`, replaces with `true`.
  - Pass 6 (Emitter):
    - For Pointers/References: Performs a raw bitwise comparison of the base addresses (LLVM `icmp eq`).
    - For Non-Primitive Structures: Implicitly performs an Address-of (`&`) comparison. Two structures are identical only if they occupy the same memory footprint; matching member-values do not constitute identity.
    - For Primitives (Runtime): If the operands are mutable variables (L-values), identity is defined by their memory address, not their current value.
    - For Primitives (Constants): If both operands are `NODE_CONSTANT`, the emitter relies on the Literal Note—matching constant values are treated as a single "Global Identity" and folded to `true` at compile-time.

#### `!==`

Definition:
  - The inverse of the "Formal Identity" operator (Precendence 40, Left-Associative)
  - Performs an "Identity-Level" comparison; returns `true` only if the LHS and RHS refer different memory locations or differing unique symbolic instances.
  - Literal Note: Constant literals of identical value are treated as having a shared "Global Identity" in the Refinery (e.g., `100 !== 100` is always `false`).

Operator Contract:
- Surface:
  - LHS/RHS: Any symbol, pointer, array or literal.
  - Result: A `boolean` value (`true` if the identities are distinguishable, else `false`).
- Compiler:
  - LHS/RHS: ANY_NODE (identities must be addressable or comparable).
  - Result: `NODE_BOOL` (carrying the raw expression)
    - Note: it might be needed to add a `NODE_LOGICAL` to provide a clear distinction.

Architectural Contract:
    - This operator is the "logical inverse" of `===` and the contract can be read as the physical opposite.
    - All physical and symbolic rules for `===` apply; the result is logically inverted.

Semantic Interpretation:
  - Disparity Discovery: "Prove that these two references do not share a common genesis or memory footprint."
  - Collision Check: "Confirm that a mutation to one entity cannot, by definition, affect the state of the other."

Technical Behavior:
  - Pass 1 (Pratt/Parse): LED at 40; calls `parse_expression(p, 41)`.
  - Pass 4 (Semantic Validation):
    - Verifies that both sides are "comparable" (not void).
    - Identity Folding: If both operands are the same `NODE_IDENTIFIER` or identical `NODE_CONSTANT`, replaces with `false`.
  - Pass 6 (Emitter):
    - The "Unfolded" Case: If Pass 4 cannot prove identity at compile-time, the emitter treats the result as a runtime `boolean` based on the address disparity.
    - The Inversion Logic: For all cases (Pointers, Structures and Constants), the emitter generates the `icmp ne` (`Not Equal`) instruction in LLVM.
    - For Pointers/References: Performs a raw bitwise comparison of the base addresses (LLVM `icmp ne`).
    - For Non-Primitive Structures: Implicitly performs an Address-of (`&`) comparison. Two structures are different if they occupy different addresses.
    - For Primitives (Runtime): If the operands are mutable variables (L-values), identity is defined by their memory address, not their current value.
    - For Primitives (Constants): If both operands are `NODE_CONSTANT`, the emitter relies on the Literal Note—matching constant values are treated as a single "Global Identity" and folded to `false` at compile-time.

#### `==`

Definition:
  - The `value equality` operator (Precendence 40, Left-Associative)
  - Performs a logical comparison of the magnitudes or contents of two operands.
  - Literal Note: If both LHS and RHS are constants, the compiler performs a Constant Fold, replacing the expression with a `true` or `false` literal during Pass 4.

Overload Resolution:
  - If the LHS type defines an intrinsic or overload for `==`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default value equality is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Numeric expressions, bit-fields, or types with an `intrinsic` equality operator.
  - Result: `true` if the values represent the same magnitude or state; else `false`.
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (must be type-compatible)
  - Result: `NODE_BOOL`

Architectural Contract:
  - Value Equivalence: Unlike `===`, this operator ignores the Genesis or memory address. It asks only: "Is the data held by these two identities identical in bit-pattern or logical meaning?"
  - Type Compatibility: Pass 4 enforces that operands are either the same type or can be promoted to a Common Denominator (e.g., `u32 == u64` promotes the `u32` to `u64` before checking).
  - Floating-Point Sensitivity: Follows IEEE 754 rules; notably, `NaN == NaN` is always `false`.


Semantic Interpretation:
  - Parity Assessment: "Determine if these two distinct identities represent the same logical quantity."
  - Content Validation: "Confirm that the transformation of the LHS results in a state indistinguishable from the RHS."

Technical Behavior:
  - Pass 1 (Pratt/Parse): LED at 40; calls `parse_expression(p, 41)`.
  - Pass 4 (Semantic Validation):
    - Verifies that both sides are "comparable" (not void)
    - Verifies that both sides have "compatible" types
    - Constant Folding: if both the LHS and RHS are `NODE_CONSTANT` with the same identity (e.g., `LHS === RHS`), replaces with `true`; otherwise replaces with `false`.
    - Overload Lookup: If the operands are structures, searches for an `intrinsic` operator `==` within the LHS type definition.
      - If no overload is found, this is a Semantic Error (`Operation Not Implemented`)
  - Pass 6 (Emitter):
    - Integer (All): Emits LLVM `icmp eq`. (Signedness does not affect equality bit-patterns).
    - Floating-Point: Emits LLVM `fcmp oeq` (Ordered and Equal).
    - Complex Types: Emits a `call` to the `intrinsic` function resolved in Pass 4.

#### `!=`

Definition:
  - The Value Inequality operator (Precedence 40, Left-Associative).
  - Performs a logical comparison; returns `true` if the magnitudes or states of the operands differ.
  - On a technical level this is, literally, the inverse of the `Value Equality` operator, `==`

Overload Resolution:
  - If the LHS type defines an intrinsic or overload for `!=`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default value inequality is performed.

Operator Contract:
- See `==` and invert the logic involved.

Architectural Contract:
  - As the inverse of `==` all the same rules and guarantees apply, just inverted.

Semantic Interpretation:
  - Disparity Check: "Confirm that these two identities represent distinct logical quantities."

Technical Behavior:
  - Pass 1 (Pratt/Parse): LED at 40; calls `parse_expression(p, 41)`.
  - Pass 4 (Semantic Validation):
    - Verifies that both sides are "comparable" (not void)
    - Verifies that both sides have "compatible" types
    - Constant Folding: if both the LHS and RHS are `NODE_CONSTANT` with differing identities (e.g., `LHS !== RHS`), replaces with `true`; otherwise replaces with `false`.
    - Overload Lookup: If the operands are structures, searches for an `intrinsic` operator `!=` within the LHS type definition.
      - If no `!=` overload is found but a `==` overload exists, the compiler may synthesize `!=` as the logical negation of `==`. Otherwise, it is a semantic error.
  - Pass 6 (Emitter):
    - Integer (All): Emits LLVM `icmp ne`. (Signedness does not affect equality bit-patterns).
    - Floating-Point: Emits LLVM `fcmp une` (Unordered and Not-Equal).
    - Complex Types: Emits a `call` to the `intrinsic` function resolved in Pass 4.

#### `<`

Definition:
  - An "Ordered Comparison" operator (Precedence 40, Left-Associative)
  - Performs a magnitude comparison; returns `true` if the LHS value is strictly lower than the RHS value.

Overload Resolution:
  - If the LHS type defines an overload for `<`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default less-than comparison is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Numeric expressions (Integers/Floats).
    - Operands must be of the same type, promotable to a common denominator, or implement an overload that defines the comparison.
  - Result: true if $LHS \lt RHS$
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Numeric).
    - Type compatibility or overload resolution is enforced.
  - Result: `NODE_BOOL`

Architectural Contract:
  - Signedness Mandate: The compiler must query the signed metadata of the operand types to select the correct machine-level ordering (`slt` for signed, `ult` for unsigned).
  - Float Variance: Follows IEEE 754 (Standard "Ordered" comparison). If either operand is `NaN`, the result is `false`.

Semantic Interpretation:
  - Magnitude Precedence: "Determine if the LHS identity precedes the RHS identity in a linear numeric sequence."
  - Boundary Verification: "Prove that the LHS has not reached the limit defined by the RHS."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): LED at 40; calls `parse_expression(p, 41)`.
  - Pass 4 (Semantic Validation):
    - Verifies numeric compatibility or resolves an overload.
    - Performs Width Unification for numeric types.
    - Retrieves the signed flag to determine the comparison variant.
    - Constant Folding: if both operands are literals, computes the result at compile time.
  - Pass 6 (Emitter):
    - Integer (Signed): Emits LLVM `icmp slt`.
    - Integer (Unsigned): Emits LLVM `icmp ult`.
    - Floating-point: Emits LLVM `fcmp olt` (`Ordered Less Than`).
    - Overload: Emits a call to the operator code.

#### `>`

Definition:
  - An "Ordered Comparison" operator (Precedence 40, Left-Associative)
  - Performs a magnitude comparison; returns `true` if the LHS value is strictly greater than the RHS value.

Overload Resolution:
  - If the LHS type defines an overload for `>`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default greater-than comparison is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Numeric expressions (Integers/Floats).
    - Operands must be of the same type, promotable to a common denominator, or implement an overload that defines the comparison.
  - Result: true if $LHS \gt RHS$
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Numeric).
    - Type compatibility or overload resolution is enforced.
  - Result: `NODE_BOOL`

Architectural Contract:
  - Signedness Mandate: The compiler must query the signed metadata of the operand types to select the correct machine-level ordering (`sgt` for signed, `ugt` for unsigned).
  - Float Variance: Follows IEEE 754 (Standard "Ordered" comparison). If either operand is `NaN`, the result is `false`.

Semantic Interpretation:
  - Magnitude Dominance: "Determine if the LHS identity succeeds the RHS identity in a linear numeric sequence."
  - Boundary Surpassing: "Prove that the LHS has exceeded the limit defined by the RHS."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): LED at 40; calls `parse_expression(p, 41)`.
  - Pass 4 (Semantic Validation):
    - Verifies numeric compatibility or resolves an overload.
    - Performs Width Unification for numeric types.
    - Retrieves the signed flag to determine the comparison variant.
    - Constant Folding: if both operands are literals, computes the result at compile time.
  - Pass 6 (Emitter):
    - Integer (Signed): Emits LLVM `icmp sgt`.
    - Integer (Unsigned): Emits LLVM `icmp ugt`.
    - Floating-point: Emits LLVM `fcmp ogt` (`Ordered Greater Than`).
    - Overload: Emits a call to the operator code.

#### `<=`

Definition:
  - An "Ordered Comparison" operator (Precedence 40, Left-Associative)
  - Performs a magnitude comparison; returns `true` if the LHS value is less than or equal to the RHS value.

Overload Resolution:
  - If the LHS type defines an overload for `<=`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default less-than-or-equal comparison is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Numeric expressions (Integers/Floats).
    - Operands must be of the same type, promotable to a common denominator, or implement an overload that defines the comparison.
  - Result: true if $LHS \leq RHS$
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Numeric).
    - Type compatibility or overload resolution is enforced.
  - Result: `NODE_BOOL`

Architectural Contract:
  - Signedness Mandate: The compiler must query the signed metadata of the operand types to select the correct machine-level ordering (`sle` for signed, `ule` for unsigned).
  - Float Variance: Follows IEEE 754 (Standard "Ordered" comparison). If either operand is `NaN`, the result is `false`.

Semantic Interpretation:
  - Inclusive Boundary Check: "Determine if the LHS magnitude is within or exactly equal to the limit defined by the RHS."
  - Non-Exceedance Proof: "Verify that the LHS identity has not surpassed the RHS in a linear numeric sequence."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): LED at 40; calls `parse_expression(p, 41)`.
  - Pass 4 (Semantic Validation):
    - Verifies numeric compatibility or resolves an overload.
    - Performs Width Unification for numeric types.
    - Retrieves the signed flag to determine the comparison variant.
    - Constant Folding: if both operands are literals, computes the result at compile time.
  - Pass 6 (Emitter):
    - Integer (Signed): Emits LLVM `icmp sle`.
    - Integer (Unsigned): Emits LLVM `icmp ule`.
    - Floating-point: Emits LLVM `fcmp ole` (`Ordered Less Than or Equal`).
    - Overload: Emits a call to the operator code.

#### `>=`

Definition:
  - An "Ordered Comparison" operator (Precedence 40, Left-Associative)
  - Performs a magnitude comparison; returns `true` if the LHS value is equal to or greater than the RHS value.

Overload Resolution:
  - If the LHS type defines an overload for `>=`, the operator overloading system dispatches to the appropriate implementation. Otherwise, the default greater-than-or-equal comparison is performed.

Operator Contract:
- Surface:
  - LHS/RHS: Numeric expressions (Integers/Floats).
    - Operands must be of the same type or satisfy Implicit Width Unification (see Pass 4 rules for promotion to a common denominator).
  - Result: true if $LHS \geq RHS$
- Compiler:
  - LHS/RHS: ANY_VALUE_NODE (Numeric).
    - Type compatibility or overload resolution is enforced.
  - Result: `NODE_BOOL`

Architectural Contract:
  - Signedness Mandate: The compiler must query the signed metadata (e.g., type flags or symbol table entry) of the operand types to select the correct machine-level ordering (`sge` for signed, `uge` for unsigned). 
  - Float Variance: Follows IEEE 754 (Standard "Ordered" comparison). If either operand is `NaN`, the result is `false`.

Semantic Interpretation:
  - Threshold Verification: "Determine if the LHS magnitude has met or exceeded the minimum requirement established by the RHS."
  - Lower-Bound Inclusion: "Prove that the LHS is not strictly less than the RHS."

Technical Behavior:
  - Pass 1 (Pratt/Parsing): LED at 40; calls `parse_expression(p, 41)`.
  - Pass 4 (Semantic Validation):
    - Verifies numeric compatibility or resolves an overload.
    - Performs Width Unification for numeric types.
    - Retrieves the signed flag to determine the comparison variant.
    - Constant Folding: if both operands are literals, computes the result at compile time.
  - Pass 6 (Emitter):
    - Integer (Signed): Emits LLVM `icmp sge`.
    - Integer (Unsigned): Emits LLVM `icmp uge`.
    - Floating-point: Emits LLVM `fcmp oge` (`Ordered Greater Than or Equal`).
    - Overload: Emits a call to the operator code.
