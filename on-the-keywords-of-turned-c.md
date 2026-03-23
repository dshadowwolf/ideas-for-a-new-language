# Turned-C Keywords

Turned-C is a minimalist language where most features are implemented as functions, but a core set of keywords defines the irreducible "physical laws" of a program. These keywords are grouped by their role in the language:

## 1. Annotations

Annotations are metadata that guide the compiler but do not occupy space in the final binary. They modify translation, code generation, or enforcement rules.

| Annotation   | Description |
|--------------|-------------|
| `atomic`     | Basic `seq_cst` guarantee; triggers atomic instructions. |
| `sync`       | Guarantees `acq_rel` memory ordering; context-dependent. |
| `relaxed`    | Like `memory_order_relaxed` in C11/C17. |
| `visibility` | Controls item visibility (`public`, `private`, etc.). |
| `scope`      | Defines the scope of visibility (e.g., `class`, `package`). |
| `foreign`    | Marks a symbol as defined in another language. |
| `link_name`  | The actual name of the foreign routine as it is stored. |
| `mangle`     | Which "name mangling" algorithm was used to generate the actual name for the foreign symbol |
| `target`     | What is the "backend target" of this definition ? |
| `requires`   | Specifies required environment captures. |

### 1.1 `atomic`, `sync` and `relaxed`

#### 1.1.1 Definition
  - These are three related annotations relating to memory-ordering and the atomicity (that is, execution as a single, uninteruptable unit of work) of operations.
  - `atomic` is both a `Definition` and a `Directive` annotation.
    - Definition Annotation: Declares that a variable or name must always be accessed atomically, preventing data races and ensuring indivisible operations.
    - Directive Annotation: Applied to an operation, enforces atomic execution with the strongest memory ordering: sequential consistency (equivalent to `C11`’s `memory_order_seq_cst`).
  - `sync`
    - Directive Annotation only: Specifies that the operation must use acquire-release memory ordering. For single-step operations, this means both acquire and release semantics; for multi-step operations, acquire and release are paired appropriately. This provides a balance between performance and safety. (this is equivalent to `C11`'s `memory_order_acq_rel` in single-step form and a properly paired use of `C11`'s `memory_order_acquire` and `memory_order_release` in multi-step form)
  - `relaxed`
    - Directive Annotation only: Indicates that the operation should be performed atomically, but with relaxed memory ordering (maps directly to `C11`’s `memory_order_relaxed`). This allows maximum performance but provides no ordering guarantees beyond atomicity.
  - The `Directive` forms (`atomic`, `sync`, `relaxed`) should only be applied to operations where at least one operand is already declared as `atomic` via a `Definition` annotation.

#### 1.1.2 Technical Details
  - Pass 4 (Semantic Validation):
    - The Dependency Check: Verifies that at least one operand in the annotated expression carries the `atomic` Definition Annotation. Applying these to non-atomic names is a Pass 4 Error.
    - Constraint Enforcement: Ensures these are only applied to Integral or Pointer types. Applying atomic to a complex `NODE_STRUCTURED` without a specific backend-supported alignment is a Type Incompatibility Error.
  - Pass 6 (Emitter):
    - The LLVM Mapping:
      - atomic: Emits instructions with `SequentiallyConsistent` ordering. This is the "Safe-Mode."
      - sync: Emits `AcquireRelease` (for RMW) or paired `Acquire` (loads) and `Release` (stores).
      - relaxed: Emits `Monotonic` ordering.
    - The Memory Fence: For multi-step operations (like a complex `NODE_FLOW`), the Emitter must inject the appropriate fence instructions before and after the primary memory transaction to satisfy the Directive contract.

### 1.2 `visibility` and `scope`

#### 1.2.1 Definition
  - These two annotations jointly determine access control for symbols: `visibility` specifies who can access a symbol, while `scope` defines the context or region where that restriction applies.
  - `visibility`
    The `Turned-C` equivalent of classic access modifiers (`public`, `private`, `protected`), but with a broader set of options:
    - `object`: The symbol is accessible only within the object where it is defined (similar to `private` in C++/Java).
    - `derived`: The symbol is accessible within the object and any objects derived from it (similar to `protected` in C++/Java).
    - `public`: The symbol is accessible from any context. This is the default.
    - `private`: (Considered shorthand for `module` scope; see below.)
    - `protected`: (Considered shorthand for `package` scope; see below.)
  - `scope`
    Specifies the region or boundary within which the `visibility` restriction is enforced:
    - `object`: Restricts access to the defining object.
    - `package`: Restricts access to the specified package.
    - `module`: Restricts access to the specific module within a package.
    - `world`: No restriction; the symbol is accessible everywhere.
  - Note: In `Turned-C`, `private` and `protected` may be treated as convenient aliases for `module` and `package` visibility, respectively.

#### 1.2.2 Technical Details
  - Pass 2 (Import/Linkage):
    - The Access Gate: When a module is imported, the Refinery filters the incoming symbol list. Any symbol tagged with a scope of module or object that does not match the current importing context is hidden from the Symbol Table.
    - Namespace Isolation: Symbols marked as `scope: object` are only injected into the Child Symbol Table of the parent structure. They are unreachable via global or package-level lookups.
  - Pass 4 (Semantic Validation):
    - The Provenance Check: For every symbol access (`node.member`), the validator compares the Origin Metadata of the symbol against the Current Context.
      - If `visibility: derived` and the current context is not a `NODE_DERIVED` of the origin, throw a Visibility Violation Error.
    - Alias Resolution: If the parser encounters `private`, it automatically promotes the metadata to `visibility: object` + `scope: module`. If it encounters `protected`, it promotes to `visibility: derived` + `scope: package`.
  - Pass 5 (Metadata Emitter):
    - TCH0 Filtering: When generating the `.tch` file, symbols with `scope: module` are marked with the `INTERNAL` bit. This instructs the Pass 2 logic of other packages to ignore these entries entirely, effectively "stripping" the private API from the public header.
  - Pass 6 (Emitter):
    - Symbol Mangling: Symbols with `scope: world` or `visibility: public` are emitted with their raw link_name. All other symbols undergo Identity Mangling (prefixed with the Package/Module hash) to prevent accidental collisions during the final OS-level link phase.

### 1.3 `foreign`, `link_name` and `mangle`

#### 1.3.1 Definition
  - These three annotations enable Foreign Function Interface (FFI) capabilities in Turned-C, allowing seamless integration with code and symbols defined outside the language.
  - `foreign`
    Indicates that the attached definition refers to a function or symbol implemented outside of Turned-C.  
    - Can be used as a standalone (monadic) annotation or as a key in a key-value pair, where the value specifies the calling convention to use when invoking the foreign function.
    - When used standalone (monadic) the value defaults to `system`
    - Common calling convention values include:
      - `system`: Uses the platform’s default calling convention (e.g., `stdcall` on Win32).
      - `cdecl`: Standard C calling convention.
      - `stdcall`: Standard call, often used in Windows APIs.
      - `fastcall`: Passes arguments via registers for speed.
      - `vectorcall`: Passes arguments (especially vector and floating-point types) in registers for improved performance on supported platforms; commonly used in modern x86 and x64 Windows environments.
      - (Other conventions may be supported as needed for interoperability.)
  - `link_name`
    Specifies the exact name of the foreign function or symbol as it appears in the shared library or DLL.  
    - This is necessary when the external name does not conform to Turned-C’s identifier rules (e.g., names with special characters or case sensitivity).
  - `mangle`
    Indicates the name mangling scheme used for the foreign symbol, enabling compatibility with languages like C++ where name mangling is part of the ABI.  
    - This allows Turned-C to correctly resolve and link to symbols whose names are transformed by the target language’s mangling rules.
    - When omitted the default of `mangle: none` is used.

#### 1.3.2 Technical Details
   - Pass 4 (Semantic Validation):
   - Type-Check Inversion: For foreign definitions, the validator relaxes the Concept #3 (Provably Correct) checks for the body (since no body exists) but enforces strict C-Compatibility on the signature. Parameters must be `pointer`, `array`, or primitive types that map directly to the target ABI.
   - Calling Convention Registry: Validates the `foreign: <convention>` value against the Backend Capability Table. If the requested convention (e.g., `vectorcall`) is not supported by the target architecture, throw a Backend Incompatibility Error.
   - Pass 5 (Metadata Emitter):
   - External Marker: Symbols marked as foreign are serialized into the `.tch` file with the `STUB` bit. This prevents the compiler from attempting to inline or optimize the logic, as the implementation is "Invisible" to the compiler.
   - Linker Directive: The `link_name` is stored as the primary External Identity for the symbol, overriding the `Turned-C` `#name` for all final linkage steps.
   - Pass 6 (Emitter):
   - LLVM Attribute Injection:
      - `foreign: cdecl` maps to the `cc 10` (`C Calling Convention`) attribute in LLVM IR.
      - `foreign: stdcall` maps to `cc 64`.
      - `foreign: system` is resolved to the host OS default (e.g., `x86_64_win64` or `linux_x86_64`).
   - Symbol Resolution: If `link_name` is present, the Emitter discards the `Turned-C` identifier and emits the raw string into the assembly/object file.
   - Mangling Engine:
      - If `mangle: none` (default for foreign), the symbol is emitted as-is.
      - If `mangle: cplusplus`, the Emitter invokes a Target-Specific Mangler (e.g., `Itanium` or `MSVC`) to transform the signature into the appropriate decorated string (e.g., `_Z3fooii` for `foo(int, int)`).

### 1.4 `target`

#### 1.4.1 Definition
  - The `target` annotation is a key mechanism that enables Turned-C to bootstrap itself without built-in arithmetic or comparison operators in the "Bootstrap Core." It allows primitives provided by a code-generation backend (such as LLVM) to be directly associated with a type or operator.
  - Used as the key in a key-value pair, the value designates the backend primitive to target, such as `LLVM::i64` or `LLVM::uge`.
  - This acts as a direct bridge between the parser’s core and the final compilation stage (`Pass 6` - The Emitter), enabling high-performance, backend-native operations from within Turned-C source code.

#### 1.4.2 Implementation Note
  - A symbol with a `target` annotation cannot have a `NODE_FUNCTION` (a `{ block }` body) because the logic is already provided by the backend. This prevents the "Double Implementation" conflict.

#### 1.4.3 Technical Details
  - Pass 1 (Pratt/Parsing):
    - Annotation Collapse: During parsing, the `target` annotation is merged into the metadata of the annotated symbol. This ensures that backend-specific information is available for later compilation stages without altering the symbol’s syntactic structure.
  - Pass 4 (Semantic Validation):
    - Protocol Match: The validator checks that the Turned-C definition’s signature matches the requirements of the backend primitive.
      - For example, `target: LLVM::sdiv` must be attached to an operator with exactly two numeric operands and a numeric return type.
      - If the signature is incompatible (e.g., `(u64) : void` for `LLVM::add`), a "Backend Protocol Mismatch" error is raised.
    - Intrinsic Verification: The compiler consults its internal Intrinsic Table (e.g., the `LLVM::` namespace). If the specified value (like `LLVM::nonexistent_op`) is not recognized, an "Unknown Backend Target" error is thrown.
  - Pass 6 (Emitter):
    - Zero-Instruction Swap: When the Emitter encounters a symbol with a `target` annotation, it emits the specified backend opcode (e.g., `sdiv`, `icmp`, `fadd`) directly, rather than generating a function call.
      - This enables Turned-C to define operators like `+` or `/` in source, yet have them execute as efficiently as native assembly.
      - If `target: LLVM::icmp` and the intrinsic `signed` is `true`, the Emitter knows to pick the `slt`/`sgt` variants. If `false`, it picks `ult`/`ugt`. It turns signed into a Sub-Opcode Directive.
    - Type Mapping: For example, `target: LLVM::i64` instructs the Emitter to lower the associated type to a 64-bit integer in LLVM IR.

### 1.5 `requires`

#### 1.5.1 Definition
  - This works similar to how a `lambda capture` works in C++, but is different in key details. Unlike a `lambda capture` if the original scope `dies` (that is, if it goes out of scope or is destroyed) the metadata -- in fact the entire original AST Node -- remains a valid, seekable part of the package.
  - The `key` of a key-value pair. The value is a bracketed array (`[...]`) of names to "capture" from the defining environment that may later be overridden by the final, "reparented" (i.e., moved to a new calling environment or been lowered into a "mock" object) calling environment.

#### 1.5.2 Technical Details
  - Pass 3 (Macro Expansion / Capture):
    - The Snapshot: When the parser encounters a `required: [...]` block, it performs a Lexical Lookup for each name in the list.
    - The Node Wrap: Each captured name is transformed into a `NODE_CAPTURE`. This node stores both the symbol's name and a Fat Pointer (e.g., a pointer with provenance and metadata) to its original definition in the current scope.
    - Environment Nesting: These captures are injected into the macro's internal symbol table, ensuring they are "recorded" before the macro is shipped off to a new call site.

  - Pass 4 (Semantic Validation):
    - The Shadow Check: If a required name is later "overridden" in the calling environment, Pass 4 treats the local version as a Shadow Variable.
    - Preference Logic: The Refinery defaults to the captured NODE_CAPTURE unless an explicit override is provided in the call site's context.

  - Pass 6 (Emitter):
    - The Closure Bridge: For deferred blocks (`[=]`), the Emitter uses the required list to generate the Closure Environment. It packs these captures into a serializable structure (the Thunk) so they can be transported across thread or module boundaries without losing their identity.

## 2. Types, Type-classes and Type modifiers

Types, type-classes, and type modifiers are the foundational building blocks for describing data and its behavior in Turned-C.  
  - **Types** define the concrete shape, storage, and interpretation of values.
  - **Type-classes** group related types under a common semantic or operational umbrella, enabling generic programming and operator compatibility.
  - **Type modifiers** extend or refine the meaning of base types, such as by adding addressability or bounds.

### The "Axiomatic" Promotion Logic

In Pass 4 (Semantic Validation), type promotion negotiation proceeds as follows:
1. **Identity Match:** If `LHS.type === RHS.type`, no promotion occurs.
2. **Intrinsic Override:** If either operand’s type definition provides an `intrinsic` promotion rule for the other, that rule is applied (e.g., `#u64` may specify: “If paired with `u32`, promote the `u32`”).
3. **Type-Class Fallback:** If both operands belong to the same type-class (e.g., Integer), the Refinery applies the Axiomatic Width Rule:
   - Promote the narrower bit-width to the wider bit-width.
   - If widths are equal but signedness differs, promote to the unsigned variant (or trigger a compile-time Precision Warning).

**Note**: the work on proper designation of types and the type-promotion rules and other features is incomplete at this time and ongoing, but the preceding text is final and will not change.


The following table summarizes the core types and type-classes available in the language, along with their primary roles:
| Type         | Description |
|--------------|-------------|
| `Integer`    | Type-class representing all registered integer types (`i64`, `u32`, ...) |
| `Float`      | Type-class representing all registered floating-point types (`f128`, `f64`, ...) |
| `Character`  | Type-class representing all registered "character' types (`byte`, `rune` and `grapheme`) |
| `Vector`     | Type-class representing discrete data storage for vector operations and the related types |
| `Numeric`    | Type-class representing all "numeric" data types and type-classes |
| `byte`       | Single Byte -- either raw storage, ASCII text or a similar low-level use |
| `code_point` | UCS-2 code point (rune). |
| `grapheme`   | Human-perceived character (may be multi-codepoint). |
| `void`       | Absence of value or erased type info. |
| `boolean`    | Classic true/false. |
| `string`     | A collected, contiguous run of `grapheme`s |
| `pointer`    | Address-bearing type modifier. |
| `structure`  | Heterogeneous data aggregate. |
| `array`      | Bounds-checked array modifier. |

### 2.1 Type-classes

#### 2.1.1 `Integer`

##### 2.1.1.1 Definition
  - A Semantic Category representing any discrete, whole-number bit-vector of fixed width.
  - Acts as the primary Membership Anchor for all bitwise and arithmetic displacement operations.

##### 2.1.1.2 Members
  - Includes all built-in signed and unsigned integer types (e.g., `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`).

##### 2.1.1.3 Required Operations
  - Supports arithmetic (`+`, `-`, `*`, `/`), bitwise (`&`, `|`, `^`, `~`), and comparison (`==`, `<`, `>`, etc.) operators.
 
##### 2.1.1.4 Extensibility
  - User-defined types may join the `Integer` type-class by providing compatible storage and implementing the required operations.

##### 2.1.1.5 Axiomatic Contract (Requirements)
  - To qualify for membership in the `Integer` type-class, a type must expose the following intrinsic metadata and operations:
  - Metadata Keys:
    - `` `bits``: A constant integer representing the physical bit-width.
    - `` `signed``: A boolean flag indicating Two's Complement (true) or Unsigned (false) interpretation.
  - Operator Hooks:
    - Must implement the Infix Arithmetic Tier (`+`, `-`, `*`, `/`, `%`, `**`).
    - Must implement the Bitwise Tier (`&`, `|`, `^`, `~`) and Shift Tier (`<<`, `>>`, `<<<`, `>>>`).
    - Must implement the Comparison Tier (`==`, `!=`, `<`, `>`, etc.).

##### 2.1.1.6 Type-Class Fallback (The Axiomatic Width Rule)
  - The Integer class provides the default promote logic for all its members:
  - Negotiation: When two Integer types interact, the Refinery selects the type with the greatest `` `bits`` value as the common denominator.
  - Symmetry Break: If bit-widths are equal but signedness (the `` `signed`` intrinsic) differs, the Unsigned variant is selected to prevent sign-bit corruption during bitwise operations.

##### 2.1.1.7 Extensibility
  - Manual Registration: Any type may join the class via the injection operator: `[ my-type ] -> Integer`.
  - Validation: Upon registration, Pass 4 (Semantic) verifies that the type provides the required intrinsic keys. Failure to provide these results in a "Class Contract Violation" error.

##### 2.1.1.8 Implementation Note
  - Typically represented as fixed-width, two’s-complement (signed) or unsigned binary values.
  - In the reference implementation, all Integer types are assumed to follow Two’s Complement representation for signed values to ensure predictable mapping to the LLVM iN backend.

##### 2.1.1.9 The Axiomatic Width Rule (`Integer Promotion`)

To ensure Provably Correct behavior during operations involving disparate Integer types, the Refinery applies a non-ambiguous negotiation protocol. This rule governs the selection of the Common Denominator Type before any arithmetic or bitwise machine instructions are emitted.

   1) Resolution Sequence:
      1. Identity Parity: If the LHS.type and RHS.type share a formal identity (`===`), the operation proceeds within that type’s native bit-width.
      2. Width Supremacy: If bit-widths differ, the type with the greatest bits value is selected as the target type. The narrower operand is promoted via Sign Extension (`sext`) if signed, or Zero Extension (`zext`) if unsigned, to match the target width.
      3. Signedness Symmetry-Break: If bit-widths are identical but the signed metadata differs (e.g., `i64` vs `u64`):
         - The Unsigned Bias: The Unsigned variant is selected as the target type to preserve the bit-pattern integrity for bitwise operations.
         - Safety Directive: The compiler must emit a Precision Warning at compile-time, alerting the developer that a signed-to-unsigned conversion may result in a change of interpreted magnitude (e.g., negative values becoming large positive values).
      4. Implicit Bit-Vector Fallback: If neither operand can prove its bit-width or signedness via intrinsic metadata, the operation is rejected as a Semantic Breach ("Indeterminate Numeric Contract").

   2) Implementation Note:
     - This rule ensures that the Pass 6 Emitter always receives "Normalized" operands of identical width.
     - By prioritizing the Unsigned variant during a tie, Turned-C prevents the "Sign-Bit Smearing" that occurs when an arithmetic shift is accidentally applied to a logical bit-mask.

#### 2.1.2 `Float`

##### 2.1.2.1 Definition
  - A Semantic Category representing a "floating point" value -- that is, a value with both an integer component and a fractional component.
  - Acts as the secondary Membership Anchor for all bitwise and arithmetic displacement operations.
  - For consistency and because of the well understood rules around them, the members of this type-class shall use the IEEE 754 encoding and storage in the various, documented bit-widths of said specification.

##### 2.1.2.2 Members
  - Includes all built-in floating-point types (e.g., `f32`, `f64`, and `f128`).

##### 2.1.2.3 Required Operations
  - Supports arithmetic (`+`, `-`, `*`, `/`), bitwise (`&`, `|`, `^`, `~`), and comparison (`==`, `<`, `>`, etc.) operators.
 
##### 2.1.2.4 Extensibility
  - User-defined types may join the `Float` type-class by providing compatible storage and implementing the required operations.
 
##### 2.1.2.5 Implementation Note
  - Required to be represented and stored as per the IEEE 754 specification.
  - Bit-Pattern Integrity: Members must adhere to the bit-level layouts defined by IEEE 754 (Sign, Exponent, Fraction).
  - Signage Note: While `` `signed`` is always true for numeric interpretation, the compiler recognizes that the sign bit is an internal part of the magnitude. Unlike integers, Pass 6 (Emitter) does not need a separate signed bit to select instructions; it simply selects the FPU variant (e.g., `fadd` vs `add`).

##### 2.1.2.6 Axiomatic Contract (Requirements)
  - To qualify for membership in the `Float` type-class, a type must expose the following intrinsic metadata and operations:
  - Metadata Keys:
    - `` `bits``: A constant integer representing the physical bit-width. (the `IEEE 754 Precision`)
    - `` `signed``: This metadata must be present, but shall always have the value of `true`.
  - Operator Hooks:
    - Must implement the Infix Arithmetic Tier (`+`, `-`, `*`, `/`, `%`, `**`).
    - Must implement the Comparison Tier (`==`, `!=`, `<`, `>`, etc.).

##### 2.1.2.7 Type-Class Fallback: (The Axiomatic Precision Rule)
  1) Precision Dominance: If bit-widths differ (e.g., `f32` vs `f64`), the type with the greatest bits count is selected as the target. The narrower operand is promoted via Floating-Point Extension (`fpext`).
  2) No Signedness Tie-Break: Since all `Float` members are inherently signed per the Axiomatic Contract, there is no "Unsigned Bias." Negotiation is strictly based on the bit-width (`Precision`).
  3) Cross-Class Promotion (Integer-to-Float): If an `Integer` and a `Float` interact (e.g., `i64` + `f32`), the `Float` class always wins the negotiation. The integer is promoted to a float of equivalent or greater bit-width (`sitofp` or `uitofp`) before the operation. This prevents the "Truncation Trap" of integer math.

##### 2.1.2.8 Extensibility
  - Manual Registration: Any type may join the class via the injection operator: `[ my-type ] -> Float`.
  - Validation: Upon registration, Pass 4 (Semantic) verifies that the type provides the required intrinsic keys. Failure to provide these results in a "Class Contract Violation" error.

##### 2.1.2.9 The Axiomatic Precision Rule (Floating-Point Promotion)
To ensure Provably Correct behavior during operations involving disparate Float types, or mixed-class numeric expressions, the Refinery applies a deterministic negotiation protocol. This rule governs the selection of the Common Precision Target before any floating-point machine instructions are emitted.

- Resolution Sequence:
   1. Identity Parity: If the `LHS.type` and `RHS.type` share a formal identity (`===`), the operation proceeds within that type’s native bit-width (e.g., `f64` + `f64`).
   2. Precision Supremacy: If bit-widths differ, the type with the greatest bits value is selected as the target type (e.g., `f32` vs `f64` promotes to `f64`). The narrower operand is promoted via Floating-Point Extension (`fpext`) to match the target precision.
   3. Cross-Class Dominance (The Numeric Bridge): When a member of the `Integer` type-class interacts with a member of the `Float` type-class:
      - The Float Bias: The Float type is always selected as the target, regardless of the relative bit-widths.
      - The Conversion: The integer operand is promoted to the target float type via Signed-to-Float (`sitofp`) or Unsigned-to-Float (`uitofp`) conversion, preserving the numeric magnitude within the limits of the target's precision.
   4. Bit-Pattern Integrity: Because all Float types are inherently signed per the `IEEE 754` specification, no signedness symmetry-break is required. The compiler relies on the internal bit-pattern for sign preservation during all operations.

- Implementation Note:
  - This rule prevents the "Truncation Trap" by ensuring that any interaction between a discrete integer and an approximate float always results in a float.
  - By prioritizing the widest precision, `Turned-C` minimizes the accumulation of rounding errors during complex arithmetic chains.

#### 2.1.3 `Character`

##### 2.1.3.1 Introduction
The `Character` type-class in Turned-C is intentionally anchored to Unicode standards (UTF-32/UCS-2 for `rune`, Unicode grapheme clusters for `grapheme`). This ensures maximal compatibility, expressiveness, and future-proofing for all human writing systems, as Unicode is the globally recognized and evolving standard for text representation. Extensions beyond this model are not anticipated or encouraged, as Unicode is designed to accommodate new scripts and symbols as they arise.

##### 2.1.3.2 Definition
  - A Semantic Category representing the various layers of textual and symbolic data.
  - Acts as the primary Membership Anchor for all text-processing, comparison, and display-oriented operations.

##### 2.1.3.3 Axiomatic Contract -- Members
  - `sigil`
    - Width: 8 bits (`LLVM::i8` in a specifically unsigned representation)
    - Domain: Binary/Legacy Encoding (e.g., ASCII, EBCDIC)
    - Description: A `sigil` represents the minimal irreducible storage unit of a textual encoding. While it serves as the physical carrier for ASCII or EBCDIC symbols, it remains semantically distinct from the abstract rune to prevent 'Magnitude-vs-Meaning' ambiguity.
  - `rune`
    - Width: 32 bits (`LLVM::i32` in a specifically unsigned representation)
      - I'm sticking with 32 bits for now, but have decide to "leave the lights on" for whenever the Unicode Consortium decides that human expression needs a 64-bit playground.
    - Domain: Unicode Scalar
    - Description: A single Unicode code point. Represents any symbol defined by the Unicode standard (`U+0000` to `U+10FFFF`).
  - `grapheme`
    - Width: Indeterminate (Fat Reference: pointer to runes + cluster count)
    - Domain: User-perception
    - Description: A "user-perceived character" or Extended Grapheme Cluster. May be composed of multiple runes (e.g., a base letter plus combining marks like accents or emojis).

##### 2.1.3.4 Axiomatic Contract -- Requirements
  - Metadata Keys:
    - `` `encoding``: A constant identifying the character encoding scheme (e.g., `ASCII`, `UTF-8`, `UCS-2`, etc...)
    - `` `is-atomic``: A `boolean` flag; `true` for fixed-width types/encodings, `false` for variable-width types/encodings (`grapheme`, `UTF-8`)
  - Operators:
    - All members of the `Character` type-class must implement the full set of comparison operators (`==`, `!=`, `<`, `>`, `<=`, `>=`) and identity operators (`===`, `!==`).
    - The semantics of these operators are defined on a per-member basis to provide maximum expressive power and flexibility.
      - This enables logic similar to C’s idiomatic range-checks, among other things.
    - Other comparison operators on `sigil` and `rune` operate on their raw values, while `grapheme` comparisons use Unicode normalization.

##### 2.1.3.5 Type-Class Fallback -- "The Textual Negotiation Rule"
  In Pass 4 (Semantic Validation), interactions between different `Character` members follow a "Completeness" logic:
    1) Promotion Path: A `sigil` is promoted to a `rune` via zero-extension; a `rune` is promoted to a `grapheme` as a single-element cluster.
    2) The Perception Bias: When a `rune` and a `grapheme` interact, the Grapheme class wins the negotiation. The result is treated as a grapheme-aware sequence to ensure no accidental splitting of user-perceived symbols. (e.g., performing an operation between the rune `´` and the grapheme `e` results in the grapheme cluster `é`).
    3) Strict Boundary Rule: Going from a `grapheme` to a `sigil` is like trying to fit a novel into a fortune cookie—you’re going to lose something. The compiler won't let you do it by accident; you have to sign for that loss with a `cast`.

##### 2.1.3.6 Implementation Notes
  - For Pass 6 (Emitter), `sigil` maps to `LLVM::i8` and `rune` maps to `LLVM::i32` or their same-width representations in other backends. `grapheme` is treated as a Fat Reference containing a pointer to a sequence of `rune`s and a length count.
  - Comparison Tier: `==` on graphemes performs Canonical Equivalence (i.e., Unicode normalization, comparing user-perceived characters rather than raw bits).

#### 2.1.4 `Vector`

##### 2.1.4.1 Rationale

The past decade has seen a fundamental shift in mainstream hardware: nearly all modern CPUs and many embedded processors now include vector processing (SIMD) execution units. This, combined with the rise of data-parallel computation in scientific, graphics, and machine learning workloads, makes first-class vector support a necessity for any modern language. Most existing languages treat vector operations as an afterthought, requiring awkward intrinsics or platform-specific libraries. Turned-C addresses this gap by defining a dedicated `Vector` type-class, enabling generic, type-safe, and portable vector operations as a core language feature.

##### 2.1.4.2 Description and Definition

###### 2.1.4.2.1 Definition
  - A Data-Parallel Category representing an ordered, fixed-length sequence of identical scalar "lanes" (elements).
  - The Parallel Atom: Unlike an array (a memory-resident container), a Vector is a register-resident value, designed to be loaded, transformed, and stored as a single atomic unit across multiple data points simultaneously.
  - Hardware Mapping: Maps directly to the LLVM SIMD Vector Type (`<N x T>`), where `N` is the lane count and `T` is a member of the Integer or Float type-classes.

###### 2.1.4.2.2 Axiomatic Contract (Requirements)
  - To qualify as a member of the Vector type-class, a type must expose the following intrinsic metadata:
    - `lanes`: A constant integer representing the number of parallel elements.
    - `base-type`: The scalar type of the individual elements (must be Integer or Float).
    - `width`: The total bit-width of the vector (e.g., `lanes * base-type.bits`).

###### 2.1.4.2.3 Required Operations
  - Elementwise arithmetic: `+`, `-`, `*`, `/`, `%` (if base-type supports).
  - Elementwise bitwise: `&`, `|`, `^`, `~` (if base-type supports).
  - Elementwise comparison: `==`, `!=`, `<`, `>`, `<=`, `>=`.
  - Lane-wise shift: `<<`, `>>` (if base-type supports).
  - Swizzle/shuffle: Swizzle operations (e.g., `v.xxyy`) are implemented as Intrinsic Property Accessors that map to the LLVM `shufflevector` instruction.
  - Scalar broadcast: Implicitly convert a compatible scalar to a vector of matching width for mixed operations.
  - Horizontal operations (e.g., `sum(v)`) are provided by the Standard Prelude as specialized intrinsic functions, as they break the Symmetry of Parallelism.

###### 2.1.4.2.4 Axiomatic Logic (The Broadcast Rule)
  - In Pass 4 (Semantic Validation), interactions involving Vector types follow the Symmetry of Parallelism:
    1. **Elementwise Dispatch:** Operations between two identical vectors (e.g., `v4f32` + `v4f32`) are performed lane-wise in parallel.
    2. **Scalar Broadcast:** If a vector interacts with a compatible scalar (e.g., `v4f32` * `f32`), the scalar is automatically broadcast to a temporary vector of the same width before the operation.
    3. **Strict Lane Matching:** Operations between vectors of differing lane counts or base-types are rejected as a structural mismatch unless an explicit shuffle or cast is provided.

###### 2.1.4.2.5 Technical Behavior
  - Pass 6 (Emitter):
    - Generates LLVM vector instructions (e.g., `fadd <4 x float>`).
    - Enforces target-specific alignment (e.g., 16-byte for SSE, 32-byte for AVX) to ensure hardware can perform aligned loads and stores.
    - **Register Width Note:** If the total vector width (`lanes * base-type.bits`) exceeds the maximum physical SIMD register size supported by the target CPU, the Emitter relies on the backend to automatically "scalarize" or "split" the vector operation into hardware-legal chunks. This ensures portability across architectures with varying SIMD capabilities.

#### 2.1.5 `Numeric`

##### 2.1.5.1 Introduction

A semantic supertype representing all values that participate in numeric operations and cross-primitive promotion. The `Numeric` type-class encompasses all members of the `Integer` and `Float` type-classes, and may include other types that can be promoted to numeric forms. This type-class provides a unified framework for reasoning about numeric computation, type negotiation, and operator compatibility.

- **NOTE**: The "other types" mentioned should fit with `Integer` and `Float` in representing a scalable magnitude or quantity, not merely be convertible to one. The `Numeric` type-class is for things that are quantities, not just things that can be coerced into looking like one.

##### 2.1.5.2 Definition

- The "Root Category" for all types representing a scalable magnitude or quantity.
- **Arithmetic Anchor:** Serves as the primary parent for the `Integer` and `Float` type-classes.
- **Unified Negotiation:** Defines the global rules for how disparate numeric representations (discrete vs. approximate) interact within Turned-C.
- **Extensibility:** Permits future numeric-like types (e.g., fixed-point, complex) to join, provided they satisfy the axiomatic contract.

##### 2.1.5.3 Axiomatic Contract (Requirements)

- **Membership:** All `Integer` and `Float` types are implicit members. Additional types may join as long as they meet the type-class contract.
- **Operator Hooks:** Must support the full set of arithmetic and comparison operators, either directly or via promotion to a canonical numeric type.
- **Promotion Guarantee:** Any member must define or inherit a deterministic and unambiguous promotion path to a canonical numeric type for all supported operations, ensuring no ambiguous or lossy conversions.

##### 2.1.5.4 Promotion Logic (The Numeric Bridge)

When operands of different numeric type-classes interact (e.g., `Integer` and `Float`), the `Numeric` promotion rules are applied:
1. **Class Dominance:** If one operand is a `Float` and the other is an `Integer`, the result is promoted to the `Float` type best capable of representing the entire Integer's full range, or the widest available Float in the absence of a perfect match.
2. **Precision Preservation:** Promotions always favor the type with the greatest precision and range, minimizing loss of information. If two types are incomparable in precision or range, the operation is ill-formed unless an explicit cast is provided.
3. **Literal Resolution:** Unbound numeric literals are resolved to the minimal type that can represent their value and satisfy the requirements of the operation, subject to context and the above rules.

##### 2.1.5.5 Technical Details

- **Pass 4 (Semantic Validation):**
  - Literal constants are born into the `Numeric` class as 'Unbound Magnitudes' and only resolve to a concrete type-class (`Integer` or `Float`) upon their first interaction with a typed identity or an explicit type-hint.
  - The validator enforces that all operands in a numeric operation are either of the same type or are promotable to a common `Numeric` type via the above rules.
  - If the compiler can't guarantee the safety of the promotion, it doesn't 'try its best'—it stops and demands an explicit cast. We'd rather offend the programmer's ego than lose their data.

##### 2.1.5.6 Implementation Notes

- Promotion from Integer to Float defaults to the Axiomatic Precision Rule, prioritizing the preservation of magnitude over bit-width symmetry.
- The `Numeric` type-class centralizes cross-type promotion logic, allowing `Integer` and `Float` to remain focused on intra-class rules.
- The Emitter (Pass 6) relies on the resolved `Numeric` type to select the appropriate backend instructions and conversions.
- While Vector types contain Numeric elements, they do not participate in the Numeric promotion bridge. Parallel-to-Scalar interactions are governed by the **Vector Broadcast Rule** (see Section 2.1.4).
- **Extensibility Note:** Any future numeric-like type (e.g., `fixed-point` or `complex`) must specify its promotion relationship to existing Numeric types to avoid ambiguity and ensure deterministic behavior.

### 2.2 `types`

#### 2.2.1 `boolean`

##### 2.2.1.1 Definition

- The fundamental Logic Type of the Bootstrap Core.
- Represents a binary state: `true` (Assertion) or `false` (Negation).
- The Decision Fuel: The only type capable of driving the `?` (Decision) and Predicate Lambda `[[ ... ]]` structural operators.

##### 2.2.1.2 Axiomatic Contract (Requirements)

- Only two valid values: `true` and `false`.
- May not be implicitly constructed from integers, pointers, or other types; explicit comparison is required.

##### 2.2.1.3 Technical Details

- **Pass 4 (Semantic Validation):** Enforces strict predicate logic. Integers or pointers cannot be "truthy"; they must be explicitly compared (e.g., `x != 0`) to produce a `boolean`.
- **Pass 6 (Emitter):** Maps to the LLVM `i1` type.

#### 2.2.2 `void`

##### 2.2.2.1 Definition

- Represents the Absence of Value or Erasure of Metadata.
- The Empty Pipe: Used to signal that a logic block (`{}`) or function produces no data for the next stage.
- Dual Semantics:
  - As a Return Type: Indicates a function is executed solely for its side effects and does not associate a resulting value with the caller’s scope.
  - As a Pointer Target (`pointer void`): Denotes a memory address where the "storage shape" is intentionally erased, allowing for the transport of raw data across type-safe boundaries.

##### 2.2.2.2 Axiomatic Contract (Requirements)

- May not be instantiated or assigned a value.
- May not be the source or target of a flow/bind operation except as explicitly permitted for erased pointers.

##### 2.2.2.3 Technical Details

- **Pass 4 (Semantic Validation):**
  - The Flow Gate: Prevents a `void` result from being bound (`<->`) or flowed (`<-`) into a non-void destination.
  - Size Zero: Assigns a `serialized_size` of `0`, ensuring it consumes no physical space.
- **Pass 6 (Emitter):** Maps to the LLVM `void` type and suppresses return-value generation.

#### 2.2.3 `string`

##### 2.2.3.1 Definition

- A composite type representing a contiguous, immutable sequence of Graphemes.
- The Perception Engine: Unlike a raw `sigil` array, a `string` is aware of the Character Type-class boundaries. It treats a "user-perceived character" (grapheme) as its atomic unit.

##### 2.2.3.2 Axiomatic Contract (Requirements)

- Immutable: Once constructed, the sequence of graphemes cannot be altered.
- All operations (indexing, slicing) must occur at grapheme boundaries; slicing within a multi-rune cluster is a semantic error.

##### 2.2.3.3 Technical Details

- **Pass 4 (Semantic Validation):**
  - The Grapheme Lock: Enforces that any operation on a string (indexing, slicing) occurs at grapheme boundaries.
  - Promotion Hub: Facilitates the Perception Bias. If a byte or rune is "flowed" (`<-`) into a string, it is automatically promoted to a grapheme cluster.
- **Pass 6 (Emitter):**
  - Represented as a Fat Managed Structure containing a pointer to the heap/data-segment and a Grapheme Count.
  - Intrinsic Library: Instead of a "binary blob," the Emitter sees a capability table of intrinsic functions (e.g., `split`, `upper`, `join`) that map to highly optimized, encoding-aware backend routines.
  - Note: Raw storage for a string constant should be Pascal-style (length-prefixed), not C-style null-terminated. The prefix is the number of runes making up the graphemes, followed by the collected runes.

#### 2.2.4 `Integer` type-class members

##### 2.2.4.1 Introduction

The following are the required, pre-defined "primitive" integer storage types that every Turned-C implementation must provide, along with their mapping to the LLVM backend types used in the reference implementation.

All signed integer types in Turned-C are defined to use two’s complement representation.

##### 2.2.4.2 The Types: Quick Overview

###### 2.2.4.2.1 `u8` / `i8`

**Definition:**
- The smallest integral types, each storing a single octet.
  - `u8` (unsigned): Range 0–255
  - `i8` (signed): Range -128–127

**Technical Details:**
- Both map to LLVM `i8`.
- Pass 6 (Emitter) uses type metadata to select signed/unsigned instruction variants.

###### 2.2.4.2.2 `u16` / `i16`

**Definition:**
- 16 bits wide.
  - `u16` (unsigned): Range 0–65535
  - `i16` (signed): Range -32768–32767

**Technical Details:**
- Both map to LLVM `i16`.
- Pass 6 (Emitter) uses type metadata to select signed/unsigned instruction variants.

###### 2.2.4.2.3 `u32` / `i32`

**Definition:**
- 32 bits wide.
  - `u32` (unsigned): Range 0 to $2^{32} - 1$
  - `i32` (signed): Range $-2^{31}$ to $2^{31} - 1$

**Technical Details:**
- Both map to LLVM `i32`.
- Pass 6 (Emitter) uses type metadata to select signed/unsigned instruction variants.

###### 2.2.4.2.4 `u64` / `i64`

**Definition:**
- 64 bits wide.
  - `u64` (unsigned): Range 0 to $2^{64} - 1$
  - `i64` (signed): Range $-2^{63}$ to $2^{63} - 1$

**Technical Details:**
- Both map to LLVM `i64`.
- Pass 6 (Emitter) enforces signedness guarantees.

###### 2.2.4.2.5 `u128` / `i128`

**Definition:**
- 128 bits wide.
  - `u128` (unsigned): Range 0 to $2^{128} - 1$
  - `i128` (signed): Range $-2^{127}$ to $2^{127} - 1$

**Technical Details:**
- Both map to LLVM `i128`.
- On hardware lacking native 128-bit registers, the Emitter relies on backend legalization to decompose operations.
- Pass 6 (Emitter) uses type metadata to select signed/unsigned instruction variants.

#### 2.2.5 `Float` type-class members

##### 2.2.5.1 Introduction

The following are the required, pre-defined floating-point types that every Turned-C implementation must provide. All members conform strictly to the IEEE 754 specification for binary floating-point arithmetic.

##### 2.2.5.2 The Types: Quick Overview

###### 2.2.5.2.1 `f16` (Half Precision)
  - **Definition:** 16-bit storage (5-bit exponent, 10-bit significand). Primarily intended for high-density storage and GPU interoperability.
  - **Technical Details:** Maps to LLVM `half`.

###### 2.2.5.2.2 `f32` (Single Precision)
  - **Definition:** 32-bit storage. The standard floating-point type for general-purpose computation, graphics, and modeling.
  - **Technical Details:** Maps to LLVM `float`.

###### 2.2.5.2.3 `f64` (Double Precision)
  - **Definition:** 64-bit storage. The default floating-point type for high-accuracy and scientific calculations.
  - **Technical Details:** Maps to LLVM `double`.

###### 2.2.5.2.4 `f128` (Quad Precision)
  - **Definition:** 128-bit storage. Used for extreme precision where rounding errors must be minimized.
  - **Technical Details:** Maps to LLVM `fp128`.

#### 2.2.6 `Vector` type-class members

##### 2.2.6.1 Introduction

The following are the required, pre-defined `Vector` types that a Turned-C implementation may provide, subject to the capabilities of the target platform’s hardware. The internal representation of these members is platform-defined, and interoperability of `Vector` types between different platforms is not guaranteed.

**NOTE**: This refers to binary compatibility and not source compatibility. A `vf64` saved to disk or otherwise serialized on the x86 platform cannot be safely read back and directly utilized by a Turned-C program on the ARM platform without the potentially risky use of a translation layer.

##### 2.2.6.2 The Types: Quick Overview

###### 2.2.6.2.1 `vf4-32`
  - **Definition:** A vector with 4 lanes and a base type of `f32`.
  - **Target Hardware:** SSE / NEON
  - **Intrinsic Metadata:**
    - `` `lanes``: 4
    - `` `base-type``: `f32`
    - `` `width``: 128 bits
  - **Technical Details:** Maps to LLVM `<4 x float>` (`<4 x f32>`).

###### 2.2.6.2.2 `vf8-32`
  - **Definition:** A vector with 8 lanes and a base type of `f32`.
  - **Target Hardware:** AVX
  - **Intrinsic Metadata:**
    - `` `lanes``: 8
    - `` `base-type``: `f32`
    - `` `width``: 256 bits
  - **Technical Details:** Maps to LLVM `<8 x float>` (`<8 x f32>`).

###### 2.2.6.2.3 `vf2-64`
  - **Definition:** A vector with 2 lanes and a base type of `f64`.
  - **Target Hardware:** SSE2
  - **Intrinsic Metadata:**
    - `` `lanes``: 2
    - `` `base-type``: `f64`
    - `` `width``: 128 bits
  - **Technical Details:** Maps to LLVM `<2 x double>` (`<2 x f64>`).

###### `vi4-32`
  - **Definition:** A vector with 4 lanes and a base type of `i32`.
  - **Target Hardware:** SSE2 / NEON
  - **Intrinsic Metadata:**
    - `` `lanes``: 4
    - `` `base-type``: `i32`
    - `` `width``: 128 bits
  - **Technical Details:** Maps to LLVM `<4 x i32>`.

###### 2.2.6.3 Additional Notes and Details
  1) The "Legalization" Clause: If a required Vector member is utilized on a platform lacking the corresponding SIMD execution units (e.g., v8f32 on a non-AVX target), the Pass 6 Emitter shall rely on backend legalization to decompose the operation into smaller hardware-native vectors or scalar sequences. Performance may be significantly degraded, but Semantic Correctness is preserved.
  2) The "Language Extension" Clause: Additional vector types may be added as the specification matures and as the full capabilities of LLVM and target hardware are explored during the development of the reference implementation.
  3) A note: the `width` metadata is important for `Pass 5 (Metadata)` as it informs the system of the storage-size to reserve for the item in the `tch` module/package file.

#### 2.2.7 `structure`

##### 2.2.7.1 Introduction

The `structure` type is, perhaps, the most important type in Turned-C. It allows the definition of a heterogeneous collection of functions and data as a single unit. They can be considered the Turned-C analogue of the `objects` and `classes` of other languages, but they offer a breadth and depth of semantic variation and meaning that is hard to fully define in classic programming terms.

##### 2.2.7.2 Definition

1. The primary Aggregate Type and Identity Anchor of Turned-C.
2. The Holistic Unit: A heterogeneous collection of data (`members`), logic (`functions`), and metadata (`annotations`/`overloads`) treated as a single, contiguous symbolic entity.
3. Semantic Versatility: Depending on its internal composition and Meta-Capabilities (like `intrinsic`), a structure can manifest as a raw C-style struct, a high-level Object, or a specialized Module Proxy.

##### 2.2.7.3 Axiomatic Contract

1. Member Isolation: Every named element within a structure must have a unique identifier within the structure’s local Symbol Table.
2. Layout Determinism: By default, data members are laid out in memory in the order of their declaration, subject to the compilers’s Alignment Rules.
   - A pair of annotations are being considered -- `packed` and `align` -- that could alter things, but have not, yet, been added to any version of the language specification.

##### 2.2.7.4 Technical Details

1. Pass 4 (Semantic Validation):
   - The Offset Map: Calculates the physical "Distance from Base" for every data member. This map is used by Pass 6 to generate LLVM GEP (GetElementPtr) instructions.
   - Method Binding: Resolves the self identity for nested functions, ensuring that logic defined inside the structure has "Intrinsic Access" to its private data.
2. Pass 6 (Emitter):
   - Maps to the LLVM struct type for data members.
   - Functions are emitted as standalone symbols with a mangled name reflecting their "Home" structure (e.g., #my-struct.my-func).

#### 2.2.8 `Character` Type-Class Members

##### 2.2.8.1 Introduction
This section provides a concise summary of the canonical members of the `Character` type-class. For comprehensive details, including technical and semantic contracts, refer to Section 2.1.3, especially 2.1.3.3 (“Axiomatic Contract — Members”).

##### 2.2.8.2 `sigil`

**Definition:**
- The minimal, irreducible storage unit of a textual encoding.
- Physically represents symbols in legacy encodings (e.g., ASCII, EBCDIC).
- Semantically distinct from a `rune` to avoid conflating raw storage with abstract character meaning.

##### 2.2.8.3 `rune`

**Definition:**
- A single Unicode code point (range: `U+0000` to `U+10FFFF`).
- Serves as the atomic unit of abstract character identity.

##### 2.2.8.4 `grapheme`

**Definition:**
- A user-perceived character (Extended Grapheme Cluster).
- May consist of multiple runes (e.g., a base letter plus combining marks or emoji modifiers).
- Corresponds to a single, indivisible character as perceived by users.

### 2.3 Type Modifiers

#### Introduction

Type modifiers are a distinct category within Turned-C: they are neither types nor type-classes, but instead alter or extend the semantics of types to which they are applied. Unlike types and type-classes—which may be defined and extended by the user—the set of type modifiers is fixed by the language specification. This immutability makes them unique, and both of the two defined type modifiers serve a specific, well-defined role in the language’s type system. The following section details their purpose, behavior, and interaction with other type constructs.

#### 2.3.1 `pointer`

##### 2.3.1.1 Definition

A `pointer` is, at its core, an address. In most languages, a pointer is simply the memory address of an item—internally represented as an integer large enough to hold the platform’s native address size.

In Turned-C, however, a `pointer` is a richer construct. It consists not only of the base address, but also a mutable “index” (distinct from, but conceptually related to, an array index) and the size of the referenced memory region. This design enables the compiler to perform compile-time bounds checking and allows the Turned-C runtime to enforce run-time bounds checking on memory accesses.

Thus, a type with the `pointer` modifier represents the address of a memory block sized precisely to the base type, augmented with both the region’s size and the current “co-address” (the current offset within the region) index. This composite is commonly known as a “fat pointer.”

##### 2.3.1.2 Technical Details

1. The Trinity (Data Layout):
  - `base`: The raw machine address (LLVM `ptr`).
  - `index`: A signed integer offset (relative to base) used for arithmetic.
    - Note: this may, at the implementations discretion, be implemented as a second pointer entirely.
  - `limit`: A constant or dynamic magnitude representing the maximum safe extent of the memory region.
    - Note: as with the `index` this may, at the implementations discretion, be implemented as a third pointer entirely.
  - **NOTE**: `At the implementations discretion` means, precisely, that any implementation of the Turned-C compiler+Runtime (i.e. "The Language") may choose how to best represent this value for their Host-OS and hardware platform.
2. **Intrinsic Mutability of Index:** The `index` field of a pointer is always mutable, regardless of whether the pointer itself is declared as `mutable` or not. This mutability is fundamental to pointer semantics: pointer arithmetic and the index operator (`[]`) are permitted operations, provided the resulting `index` remains within the bounds $0 \leq \text{index} \lt \text{limit}$. This does not affect the mutability of the value being pointed to, only the position within the referenced region.
3. Pass 4 (Semantic Validation):
  - The Bounds Check: When the parser sees `p + 1` or `p.member`, Pass 4 verifies that the resulting index remains within the `0` to `limit` window. If it can be proven to exceed these at compile-time, it throws a Semantic Violation.
4. Pass 6 (Emitter):
  - Maps to a Target-Specific Structure (e.g., a 192-bit or 256-bit block).
  - The Safety Guard: Injects a runtime comparison before any dereference. If `index >= limit`, the compiler triggers an Immediate Trap to prevent a memory-safety breach.
5. Runtime:
  - All `pointer` accesses—whether by dereferencing or member access—must be checked by the Turned-C runtime (however implemented) to ensure that the following invariant holds: $(0 \leq \text{index} < \text{limit})$. Any attempt to use a pointer with an `index` less than zero (negative indexing) or greater than or equal to `limit` must immediately trigger a runtime trap, aborting execution to preserve memory safety. No negative indexing is permitted unless explicitly allowed by a documented intrinsic. This strict enforcement prevents out-of-bounds and potentially hostile memory accesses.

#### 2.3.2 `array`

##### 2.3.2.1 Definition

1. A bounds-checked type modifier representing a contiguous, fixed-length sequence of identical elements in memory.
2. The Physical Anchor: Unlike a `pointer` (which references a region), an `array` is the region itself. It establishes the memory bounds that any derived `pointer` must respect.

##### 2.3.2.2 Technical Details

1. Pass 4 (Semantic Validation):
   - **Static Sizing and Immutability:** The `limit` of an `array` must be established at the moment of declaration and remains strictly immutable throughout the array's lifetime. Resizing an existing array is prohibited; any change in magnitude requires the creation of a new, distinct array identity via allocation or flow.
   - **Multi-Dimensional Integrity:** For multi-dimensional arrays (e.g., `array: array: integer`), the `limit` represents the count of immediate child elements. Each sub-array maintains its own discrete limit metadata, ensuring independent bounds for every dimension.
   - **Type Preservation:** When passed as a function parameter, an `array` retains its full type and bounds information as a first-class member of the type system. It does not decay to a pointer except when explicitly required for Foreign Function Interface (FFI) compatibility or via an explicit `intrinsic` conversion.
2. Pointer Interaction:
   - **Manual Conversion (Decay):** A `pointer` may be derived from an `array`, at which point the compiler synthesizes a fat pointer with:
     - `base`: Address of the first element.
     - `index`: 0 (initial position).
     - `limit`: The array’s element count.
   - **Multi-Dimensional Navigation:** Indexing into a multi-dimensional array yields a reference to the inner array (preserving its bounds) rather than a raw address, maintaining a provably correct chain of access.
3. Pass 6 (Emitter):
   - **Data Layout:** Arrays are mapped to a contiguous memory block. The emitter ensures that any padding required for element alignment is included in the total serialized size.
4. Runtime:
   - **Bounds Enforcement:** All array accesses—whether direct or via a decayed pointer—inherit the runtime invariant $(0 \leq \text{index} < \text{limit})$ for all indexing operations, ensuring memory safety and preventing out-of-bounds access.

## 3. Capabilities

Capabilities are qualifiers that enable or restrict actions on an identity.

> **Capability Scope Restriction:**  
> Capabilities are applied exclusively to the underlying base type and never to type modifiers or to other capabilities. This restriction is fundamental: it ensures that the semantic guarantees and invariants established by type modifiers (such as `pointer` or `array`) cannot be altered, circumvented, or overridden by any capability annotation. As a result, the integrity and safety properties provided by type modifiers remain provably intact, regardless of the capabilities present on the base type.

| Capability     | Description |
|----------------|-------------|
| `mutable`      | Permits state changes after initial association. |
| `static`       | Lifetime/storage marker; initialized before entry. |
| `constructable`| Defines a default constructor. |
| `destructable` | Defines a default destructor. |

### 3.1 `mutable`

#### 3.1.1 Definition
   - A capability modifier that permits an identity to undergo state transformation after its initial association.
   - **The Flow Gate:** By default, all names in Turned-C are "locked" at genesis (`#`). The `mutable` keyword explicitly unseals the name, enabling its value to be updated via the Flow (`<-`) or Bind (`<->`) operators.
   - **Type Invariance:** Mutation is strictly value-level; the physical type and memory address of the identity remain constant. Only the internal state-representation may be transformed.

#### 3.1.2 Technical Details
  1. **Pass 4 (Semantic Validation):**
    - **Permission Check:** When the parser encounters `x <- y`, the validator inspects the metadata of `x`. If the mutable bit is unset, the operation is **ill-formed** (“Illegal Mutation of a Locked Identity”).
    - **Type Compatibility:** Ensures the incoming value (`y`) matches the established width and class of the target (`x`). Mutation cannot be used for type-punning.
    - **Genesis Exception:** Initial assignment during the Genesis (`#`) block is permitted for all identities. The mutable check only applies to subsequent Flow (`<-`) or Bind (`<->`) operations.
  2. **Pass 6 (Emitter):**
    - **Store Directive:** For mutable identities, the emitter generates a standard LLVM store instruction.
    - **Optimization Hint:** For immutable identities, the emitter treats the value as a constant, enabling aggressive global value numbering (GVN) and constant propagation, as the value is provably unchanging.

### 3.2 `static`

#### 3.2.1 Definition
  - A lifetime and storage capability that designates an identity as a permanent fixture of the Refinery's global state.
  - **Pre-Entry Genesis:** Unlike stack-resident identities that are created and destroyed within a logic block, a static identity is initialized before the `ENTRY` block of the module is executed.
  - **Persistence:** Its state is preserved across all logic-flow transitions for the entire duration of the process.
  - **Function Scope:** When applied within a function, a `static` identity is initialized exactly once, at the moment control first enters the function during program execution. Its state persists across all subsequent invocations of that function, retaining any modifications made during prior calls. This behavior is directly analogous to function-scope `static` in C and C++.


#### 3.2.2 Technical Details
  1. **Pass 5 (Metadata Emitter):**
    - **Archive Allocation:** Flags the identity for inclusion in the persistent data segment of the module’s package file (.tch). If a literal value is provided at genesis, it is serialized into the package using the appropriate structured format.
    - **Zero-Initialization Default:** In the absence of a literal value at genesis, Turned-C defaults to zero-initialization in the persistent data segment.
  2. **Pass 6 (Emitter):**
    - **Global Mapping:** Maps to an LLVM global variable.
    - **Linkage Logic:** Unless paired with a visibility modifier, static identities default to internal linkage, preventing them from leaking into the global namespace of other modules.
    - **Deterministic Initialization:** The emitter utilizes LLVM Global Constructors (`llvm.global_ctors`) to ensure deterministic initialization. Dependencies between `static` identities across modules are resolved during Pass 2 (Import/Linkage).
    - **Thread-Safe Initialization Guard:**  
      For each function-scope `static` identity, the emitter injects a thread-safe guard mechanism to ensure that initialization occurs exactly once, even in the presence of concurrent threads. This guard is implemented as an atomic "once flag" or equivalent synchronization primitive. If multiple threads attempt to enter the initialization path simultaneously, the guard ensures that only one thread performs the initialization, while all others block or spin until completion. This prevents data races and guarantees that all threads observe a fully initialized, valid state before accessing the `static` identity.

  3. **NOTE** Turned-C enforces that all `static` identities are constructed before any `INITIALIZE` logic executes, and are not destroyed until after all `EXIT` logic completes.
  
#### 3.2.3 Initialization and Destruction Order
All module-, package-, or global-scope `static` identities are initialized prior to the execution of any user-defined `INITIALIZE` block within the module or program. Likewise, destruction (if applicable) of such `static` identities occurs only after the completion of any user-defined `EXIT` block. This guarantees that global `static` storage is fully available throughout the entire explicit initialization and finalization phases of the module or program.  
Function-scope `static` identities are excluded from this rule; their initialization and destruction are governed by the function-scope semantics described above.

#### 3.2.4 Implementation Notes
  - A static identity that is not also marked as mutable is treated as a thread-safe constant. If a static identity is mutable, the Refinery enforces (or warns for) the inclusion of atomic or synchronization annotations to prevent data-race leaks.

#### 3.2.5 C/C++ Parallel
- The `static` capability in Turned-C provides a lifetime and storage guarantee directly analogous to the `static` keyword in C and C++. A `static` identity is allocated once, persists for the entire duration of the program or module, and is initialized before any logic is executed. Its state is preserved across all function calls and logic-flow transitions, and it is never destroyed until program termination.

- In both module and function scope, `static` ensures a single allocation and persistent storage for the identity, regardless of how many times the enclosing logic is entered or exited.

### 3.3 `constructable`

#### 3.3.1 Introduction

In object-oriented languages, an “object” or “class” typically exposes a constructor—a special method responsible for building and initializing new instances. This is often achieved via a method named after the class itself or a reserved name such as `this`.

In Turned-C, any `structure` can serve as a type, and construction is accomplished via a method named `init`. However, the name `init` carries no intrinsic meaning until the `constructable` annotation is applied to the enclosing structure. Only then does `init` become the designated constructor, signaling to the compiler and runtime that it should be invoked to allocate and initialize new instances of the structure-as-type, with full control over member initialization and resource allocation.

#### 3.3.2 Definition

- A Lifecycle Capability that defines the axiomatic birth of an identity.
- The `init` Hook: Designates a specific logic block as the constructor, responsible for establishing the initial invariant state of a type or structure before it is released into the broader logic-flow.
- Contractual Integrity: Ensures that an identity cannot exist in an "uninitialized" or "garbage" state. Once a type joins the constructable class, the Refinery mandates its initialization.

#### 3.3.3 Technical Details

1. **Pass 4 (Semantic Validation):**
   - **Genesis Trigger:** On encountering a Genesis (`#`) event for a constructable type, the validator automatically injects a call to the `#init` logic.
   - **Signature Enforcement:** Verifies that the arguments provided at Genesis match the parameters defined in the `#init` block.
   - **The "Living" Check:** Ensures that no other operations (member access, flow) occur on the identity until the `#init` block has successfully completed execution.

2. **Pass 6 (Emitter):**
   - **Inline Optimization:** For simple types or small structures, the emitter attempts to inline the `init` logic directly into the allocation site (stack or static segment) to eliminate call overhead.

#### 3.3.4 Technical Implementation Notes

- **Identity Precedence**: In the Turned-C "Refinery", the Genesis event for a constructable type necessitates the prior allocation of a Managed Reference (comprising the pointer base and its associated metadata).
- **The Undefined Boundary**: This allocation is a low-level compiler requirement, not a user-definable logic block. It ensures that the `init` hook operates upon a valid, addressable identity from its first instruction.
- **Contractual Separation**: The programmer maintains absolute control over the Internal State Transformation within the `init` block, while the compiler maintains absolute control over the Memory Provenance (the base allocation).
- **Deterministic Existence**: This mechanism is functionally equivalent to a stack-frame reservation or register binding; it is an infrastructural invariant of the language that ensures no constructor ever operates on a "null" or "detached" identity.

### 3.4 `destructable`

#### 3.4.1 Introduction

In most classic object-oriented languages, a special method exists to “deallocate and shut down” an object or class instance. The naming and invocation of this method vary: in C++ it is the destructor (`~ClassName`), while in managed runtimes like Java and C# it is often implicit or handled by the garbage collector, introducing a layer of “magic” outside the programmer’s direct control.

Turned-C, by contrast, provides the `destructable` annotation. When applied to a structure, and paired with a method named `fini`, it designates an explicit destructor. As with `init`, the name `fini` is semantically inert unless the annotation is present. This mechanism gives the programmer full control over resource release, shutdown logic, and cleanup, with no hidden or implicit behaviors.

#### 3.4.2 Definition

- A Lifecycle Capability that defines the axiomatic “death” of an identity.
- The `fini` Hook: Designates a specific logic block as the destructor, responsible for releasing memory allocations, terminating asynchronous processes, and severing all remaining access to the identity.
- Scope-Bound Finality: Guarantees that an identity’s “Last Will and Testament” is executed automatically when its lexical or dynamic scope concludes.

#### 3.4.3 Technical Details

1. **Pass 4 (Semantic Validation):**
   - **Terminal Injection:** Upon scope closure (`}`) or an explicit “Kill” event, the validator automatically injects a call to the `fini` logic for all resident destructable identities.
   - **Inverse-Order Dissolution:** To prevent dangling dependencies, destructable identities are destroyed in strict reverse order of their Genesis. The most recently created is the first to be finalized.
   - **The Niladic Mandate:** The validator enforces that the fini hook is a niladic (zero-parameter) function. Because its invocation is triggered by scope-exit rather than an explicit call, it cannot accept external arguments.
   - 
2. **Pass 6 (Emitter):**
   - **Cleanup Shim:** Generates machine code to invoke `fini` immediately prior to stack-frame adjustment or heap release.
   - **Axiomatic Finality:** The `fini` hook is a void operation; it cannot fail or be canceled. The Refinery considers the identity’s existence to be ending, regardless of the destructor’s internal logic. Any attempt to 'escape' the destruction (e.g., by moving a `self` reference into a global static during `fini`) is a Semantic Breach. The identity’s metadata is marked as 'Invalid' the moment the `fini` block begins.

#### 3.4.4 Technical Implementation Notes

- **Managed Reference Release:** As with `init`, the final deallocation of the underlying Managed Reference (pointer and metadata) is performed internally by the language runtime. This ensures that, after `fini` completes, all resources associated with the identity are deterministically and safely reclaimed, with no user intervention or hidden side effects.

## 4. Meta-Types

Meta-types classify a symbol’s fundamental role within the language. Unlike capabilities or type modifiers, which alter or extend the semantics of a type or operation, meta-types describe the intrinsic nature or function of a symbol at the definition level. They provide the compiler and tooling with essential information about how a symbol should be interpreted, processed, or exposed to language internals.

| Meta-Type     | Description |
|---------------|-------------|
| `macro`       | A compile-time transformer, executed during parsing (Pass 3), that manipulates AST nodes. |
| `node`        | A raw AST node, exposed for macro system manipulation or analysis. |
| `symbol`      | A raw entry in the symbol table, not otherwise classified. |
| `annotation`  | A metadata-attaching construct that may provide limited, non-semantic features for analysis or tooling. |
| `type`        | A type definition, either user-defined or mapped to a backend primitive. |
| `type-class`  | An extensible group of closely related types sharing a common contract. |
| `operator`    | A symbol representing an operator, with defined precedence and behavior. |
| `function`    | A symbol associated with executable code (logic block or function). |
| `variable`    | A symbol representing a data-holding entity (not code). |

### 4.1 Metaprogramming

#### 4.1.1 Introduction

The following meta-types are foundational to metaprogramming in Turned-C. They enable advanced extensibility, allowing user code and tooling to analyze, transform, and generate language constructs at compile time.

| Meta-Type   | Description |
|-------------|-------------|
| `macro`     | A compile-time transformer, executed during parsing (Pass 3), that manipulates AST nodes. |
| `node`      | A raw AST node, exposed for macro system manipulation or analysis. |
| `symbol`    | A raw entry in the symbol table, not otherwise classified. |
| `annotation`| A metadata-attaching construct that may provide limited, non-semantic features for analysis or tooling. |

#### 4.1.2 Macro-Programming Keywords

**Note:** Both `node` and `symbol` meta-types are exclusively available within macro contexts and are not accessible as general-purpose types elsewhere in Turned-C.

##### 4.1.2.1 `macro`

###### 4.1.2.1.1 Introduction


Macros are a primary mechanism for extensibility in Turned-C. They allow user code to manipulate and transform the source at the Abstract Syntax Tree (AST) level before semantic validation, enabling the addition of new logic—including recursive constructs—without introducing new functions.

A macro in Turned-C captures exactly one complete, following AST node (such as a `function`, `structure`, or `expression` block). The macro receives this node as its input and may operate on its child nodes as a variadic sequence. This design enables macros to perform structural transformations, pattern matching, or code generation on arbitrarily complex language constructs, while preserving syntactic and semantic correctness.

The expected type of the captured node is not specified in the macro’s signature. This avoids locking implementations into rigid naming conventions and prevents global namespace pollution. As a result, it is the programmer’s responsibility to supply macros with the correct kind of parameter, analogous to the discipline required when calling a function.

###### 4.1.2.1.2 Definition
  - Definition: A compile-time logic block that consumes and produces node entities.
  - Role: Unlike functions, macros operate on the Structure of the code rather than the Values of the data. They are the primary engine of the Companion Macro Language.
  - Executed during Pass 3. A macro takes an un-validated AST fragment and returns a transformed AST fragment that is then passed to Pass 4.

###### 4.1.2.1.2 Technical Details

- **Macro Expansion Timing:**  
  Macros are expanded during Pass 3 (`Macro Expansion`), prior to semantic validation. This ensures that all macro-generated constructs are subject to the same validation and transformation rules as hand-written code.  
  This ensures that macro-generated code is indistinguishable from hand-written code in all subsequent compilation passes.

- **Metadata Query System:**  
  1. Before macro execution, the macro is provided with a read-only view of the metadata associated with the captured AST node. This includes:
     - All annotations (Definition and Directive) applied to the node.
     - The node’s meta-type (e.g., `function`, `structure`, `operator`).
     - Any capabilities, type modifiers, or visibility attributes present.
     - Symbol table context (e.g., parent scope, linkage).

  2. This metadata query system enables macros to:
     - Inspect and conditionally transform nodes based on their annotations or capabilities.
     - Enforce or suggest coding conventions.
     - Generate context-aware code expansions.

- **Immutability and Safety:** The metadata view is strictly read-only; macros cannot mutate or remove annotations, capabilities, or meta-types directly. Any transformation must occur by generating new AST nodes or by constructing a new structure for the captured node, not by mutating the node or its metadata in place.

- **Error Handling:** If a macro attempts to access metadata that does not exist, a well-defined ‘absent’ or ‘null’ value is returned. Macros must handle such cases explicitly to avoid generating ill-formed expansions.

- **Hygiene and Scoping:** Macro expansions are subject to lexical scoping rules. Any symbols introduced by a macro are automatically renamed or namespaced to prevent accidental capture or collision, unless explicitly marked as “unhygienic.”

##### 4.1.2.2 `node`

###### 4.1.2.2.1 Definition

The `node` meta-type represents a raw, unvalidated AST node. It is exclusively available within macro contexts, serving as both the input and output type for macro transformations.

###### 4.1.2.2.2 Role

Macros operate on `node` entities, enabling structural analysis, pattern matching, and code generation at the syntactic level. Outside of macro input or expansion, the `node` meta-type is not a first-class type and cannot be referenced or constructed directly by user code.

###### 4.1.2.2.3 Technical Note

The `node` meta-type exposes the full syntactic structure of the captured code, including child nodes, metadata, and source location. This enables advanced metaprogramming while preserving the integrity of the language’s semantic validation pipeline.

###### 4.1.2.2.5 Technical Details

- Pass 3 Utility: A `node` acts as a Fat Handle to a serialized fragment of the program. It provides macros with the ability to traverse children or reorder branches.
- Pass 4 Transition: Once Pass 3 concludes, every `node` must be successfully lowered into a Typed Identity. If a macro produces a `node` that cannot be semantically resolved (e.g., a malformed or incomplete structure), the compiler triggers a semantic error.

##### 4.1.2.3 `symbol`

###### 4.1.2.3.1 Definition

The `symbol` meta-type denotes a raw entry in the symbol table, not otherwise classified as a type, function, variable, or macro. It is intended for advanced macro use cases where direct inspection or manipulation of symbol table entries is required.

###### 4.1.2.3.2 Role

Within macro execution, a `symbol` may be referenced to query properties such as name, scope, linkage, or associated metadata. However, the utility of exposing raw symbols is limited, as most macro transformations operate on AST nodes rather than symbol table entries. The `symbol` meta-type is not a first-class type outside macro contexts.

###### 4.1.2.3.3 Editorial Note

The inclusion of the `symbol` meta-type is primarily for orthogonality and academic rigor. If practical use cases do not emerge, it may be omitted from the core specification to avoid unnecessary complexity.

###### 4.1.2.3.4 Technical Details

  - Identity Lookup: Identity Lookup: Allows macros to verify if a name already exists in the current scope or to generate hygienic names to avoid collisions.
  - Metadata Access: Provides a read-only view of a symbol’s meta-type and annotations (e.g., checking if a symbol is a type or a variable).

#### 4.1.3 `annotation`

##### 4.1.3.1 Clarification

In Turned-C there are two types of annotations: `Definition` and `Directive`. For safety purposes, only `Definition` annotations may be introduced by user-code during the compilation process, and then only in a very limited form that does not alter existing semantics. Any `Definition` annotation that requires a change of semantics, and all `Directive` annotations, must be added to the specification itself and implemented as a fixed part of the compiler, with `Directive` annotations required to be implemented as part of the "Bootstrap Core" itself.

##### 4.1.3.2 Definition

An `annotation` is a metadata construct that can be attached to any symbol--any `name` or `operator`--in Turned-C regardless of `type` (i.e., annotations are orthogonal to the type system). Annotations serve as the primary mechanism for associating auxiliary information with language entities, enabling advanced tooling, analysis, and controlled semantic extension. They are the second pillar of the metaprogramming system, complementing macros by providing a declarative means to influence compilation, code generation, or analysis without altering the core logic of the annotated entity.

Annotations are not first-class types and cannot be instantiated or manipulated as values. Instead, they exist solely as metadata, accessible to the compiler, macro system, and tooling during the appropriate compilation passes.

##### 4.1.3.3 Axiomatic Contract (Requirements)

- **Attachment:** An annotation may be attached to any symbol (`identifier` or `operator`), subject to the rules of the language and the annotation’s own definition.
- **Immutability:** Once attached, the annotation’s metadata is immutable for the duration of the compilation process.
- **Non-Intrusiveness:** Definition annotations must not alter the semantics of the annotated entity unless explicitly permitted by the specification. Any annotation that changes semantics is classified as a Directive and must be part of the Bootstrap Core.
- **Visibility:** Annotations are visible to macros and tooling via the metadata query system, but are not accessible as runtime values.

##### 4.1.3.4 Technical Details

- Annotations are captured during Pass 1 (for local source) or resolved during Pass 2 (for symbols imported from other modules/packages). They are then bound to their target identities before Pass 3 begins.

- **Pass 3 (Macro Expansion):** Macros may query the presence and content of annotations on captured nodes, enabling context-sensitive transformations or code generation. Macros may not mutate or remove annotations; they may only generate new nodes with additional annotations if permitted.

- **Pass 4 (Semantic Validation):** The presence and correct application of Directive annotations is enforced during semantic validation. Any violation of the annotation’s contract (e.g., attaching a `packed` annotation to an unsupported type) results in a semantic error.

- **Pass 5 (Metadata Emitter):** High-level annotations are serialized into the module/package "archive", allowing them to persist across module boundaries.

- **Extensibility:** Only Definition annotations that do not alter semantics may be introduced at run-time or by user code. All other annotations must be defined in the language specification and implemented in the compiler.

##### 4.1.3.5 Implementation Notes

- **Annotation Syntax:** Annotations are applied using a standardized syntax (e.g., `@[ name<:args> ]@`), and may accept parameters or arguments as defined by their specification. The argument list shown as `<:args>` in the example is optional and may be omitted for parameterless annotations.
- **Tooling Integration:** Annotations provide a stable interface for external tools, linters, or IDEs to extract metadata, enforce coding standards, or generate documentation.
- **Safety and Isolation:** The strict separation between Definition and Directive annotations ensures that user-extensible metadata cannot compromise the safety or determinism of the language.

### 4.2 Axiomatic Meta-Types

#### 4.2.1 Introduction

These meta-types represent the immutable building blocks of a program’s physical and logical existence. Once an entity passes through the Macro Expansion (Pass 3) phase and is assigned one of these meta-types, its fundamental nature and role within the system are fixed. These meta-types are mutually exclusive and collectively exhaustive for all non-macro, non-annotation entities in Turned-C.

| Name | Description |
| ---- | ---- |
| `type`        | A type definition, either user-defined or mapped to a backend primitive. |
| `type-class`  | An extensible group of closely related types sharing a common contract. |
| `operator`    | A symbol representing an operator, with defined precedence, fixity, and dispatch behavior. |
| `function`    | A symbol associated with executable code (logic block or function). |
| `variable`    | A symbol representing a state-bearing identity with a fixed type and storage location. |

#### 4.2.2 `type`

##### 4.2.2.1 Definition

- The Physical Blueprint Meta-Type.
- Designates a symbol as a formal specification of memory layout, storage width, alignment, and data interpretation.
- The Physical Anchor: A `type` is the primary entity that provides the Pass 6 Emitter with the information required to orchestrate memory allocation and codify data-access patterns.
- A `type` meta-type is the canonical source of truth for all memory layout and data interpretation decisions in the compilation pipeline.

##### 4.2.2.2 Axiomatic Contract:

- Layout Determinism: A `type` must provide a statically resolvable serialized size and alignment requirement, even in the presence of nested or composite members.
- Identity Integrity: Once defined, the physical shape of a `type` is immutable.

##### 4.2.2.3 Technical Details:
- Pass 4 (Semantic Validation): Resolves the recursive size and alignment of nested structures, and verifies that all member associations satisfy the invariants of the type’s associated Type-Class.
  - Cycle Detection: Ensures that no type contains a direct or indirect recursive instance of itself without an intervening pointer or array modifier. This prevents 'Infinite Density' errors where a type's size cannot be resolved.
- Pass 5 (Metadata Emitter): Serializes the layout into the module/package archive, allowing the type to be used as a "black box" by importing modules.
  - The "black box" includes the Alignment Invariants to ensure that an importing module does not accidentally misalign a structure, potentially causing a "performance leak" in SIMD/Vector operations.
- Pass 6 (Emitter): Utilizes the resolved layout to generate backend-specific allocation and access instructions.

#### 4.2.3 `type-class`

The type-class exists because 'magic' belongs in fantasy novels, not in a systems language. It provides the legal framework for how types interact, ensuring that when an `Integer` meets a `Float`, the result isn't a guess—it's a proven outcome.

##### 4.2.3.1 Definition
- The Contractual Meta-Type.
- Designates a symbol as a dynamic registry of types unified by a shared operational protocol.
- The Logic Hub: Centralizes Axiomatic Rules (such as the `Numeric` promotion bridge) and manages dispatch for sealed operators.

##### 4.2.3.2 Axiomatic Contract
- **Membership Authority:** A type-class defines the criteria for membership (intrinsic metadata keys and required operator hooks).
- **Extensibility:** Allows new types to register as members via the Inject (`->`) operator.
- **Infallible Adherence:** When a type is injected (`->`) into a class, the compiler must verify that the target type possesses all required `intrinsic` hooks and metadata keys. Failure to satisfy the contract results in a Registry Rejection (Pass 2) or a Semantic Breach (Pass 4).

##### 4.2.3.3 Promotion Semantics
**Promotion** refers to the process by which a value of one type is implicitly or explicitly converted to another type within the same type-class, in order to satisfy the requirements of an operation or to resolve type compatibility in mixed-type expressions.

- **Promotion Logic:**  
  Each type-class that supports promotion (e.g., `Numeric`, `Integer`, `Float`) must define a deterministic and unambiguous “promote” function or rule set. This logic specifies:
  - The valid source and target types for promotion within the type-class.
  - The precedence or dominance rules when multiple types are involved (e.g., promoting `Integer` to `Float`).
  - The conditions under which promotion is allowed, required, or forbidden (e.g., no lossy or ambiguous promotions without an explicit cast).
  - The precedence or dominance rules when members of different sub-classes interact (e.g., the Numeric Bridge promoting `Integer` to `Float`).

- **Pass 4 (Semantic Validation):**  
  During Pass 4, the compiler:
  - Identifies all operations involving type-class members.
  - Applies the type-class’s “promote” logic to resolve operand types, ensuring that the result type is well-formed and that no ambiguous or lossy conversion occurs.
  - Emits a semantic error if promotion cannot be resolved deterministically, or if the operation would result in information loss without an explicit cast.

- **Promotion Contract:**  
  - All type-class members must either implement or inherit the “promote” logic as defined by the type-class.
  - The promotion path must be statically analyzable and must not depend on runtime values.
  - Promotion must preserve the semantic invariants of the type-class (e.g., numeric precision, ordering).

##### 4.2.3.4 Technical Details
- Pass 2 (Registry): Assigns a unique Category ID to the symbol, which is used for O(1) membership checks during later passes.
- Pass 4 (Semantic Validation): Acts as the primary negotiator for Implicit Promotion.

### 4.2.4 `operator`

#### 4.2.4.1 Introduction

In most languages, the set of operators is fixed and implemented as hard-coded logic. In Turned-C, operators are defined as an extensible registry.

An operator is a distinct class of token processed by the parser using a dynamic Dispatch Table. This mechanism transforms a punctuation sequence into a structured AST Node according to explicit, analyzable rules.

A minimal set of “Atomic Operators” is built-in to support the language’s core self-descriptive features. All other operators are defined in Turned-C code, each providing a Structural Contract: explicit Intrinsic Metadata (such as precedence and fixity) and a unique Symbolic Identity, which types reference to implement operator logic.

#### 4.2.4.2 Definition

An `operator` is a symbolic “call to action”—a punctuation run that acts as a universal hook. It does not possess its own logic; instead, it serves as a standardized dispatch signal that types and type-classes respond to.

##### 4.2.4.3 Axiomatic Contract: Registration and Binding

- **Binding:** An `operator` may bind itself to a set of types or type-classes to show those deemed "eligible" to implement its semantic behavior. This is truly important for a `sealed` operator, but provides room for `unsealed` ones to have non-semantic actions (such as using `+` for string concatenation) added. This replaces hard-coded behavior with a Registry of Intent. The operator declares the semantic action (e.g., summation or concatenation), while participating types define the manifestation.
- **Intrinsic Definition:** All operator properties—such as `precedence`, `fixity`, and dispatch protocol—must be defined as explicit, intrinsic fields. This eliminates "magic" annotations, ensuring operator behavior is entirely contract-driven and analyzable.

##### 4.2.4.4 Axiomatic Contract: Structural Properties

- **Precedence and Fixity:** Every `operator` must declare its precedence level and fixity (associativity) as intrinsic fields. These properties are statically enforced during parsing and semantic validation to determine evaluation order and prevent ambiguous chaining (e.g., `none` associativity).
- **Extensibility and Sealing:** Operators utilize an explicit `sealed` intrinsic field to control their extensibility. Sealing acts as a safety lock to prevent "operator hijacking," restricting overloads to a specific, immutable set of types.

##### 4.2.4.5 Axiomatic Contract: Dispatch and Resolution

- **Dispatch Mechanism:** The `operator` is a pure routing mechanism. It contains no logic; it serves only to route operations to the implementation defined by the participating types. This ensures semantics remain explicit and extensible.
- **Overload Resolution:** The compiler selects the implementation matching the operand types according to the registry. If no valid implementation is found, a Registry Breach (Pass 4) is issued, as the "call to action" has no qualified responders.

##### 4.2.4.6 Example

- **NOTE**: This uses the currently proposed, non-normative syntax. It will be updated as "Idiomatic Turned-C" settles and matures.

```
// Addition operator: left-associative, Numeric type-class, open for extension
define # + : operator <-> [
  `precedence    : integer intrinsic <- 10 ;
  `fixity        : enum intrinsic    <- `left ;
  `type-class    : reference intrinsic <- `Numeric ;
  `sealed        : boolean intrinsic <- false ;
] ;

// Equality operator: infix, non-associative, supports Numeric and Character, open for extension
define # == : operator <-> [
  `precedence    : integer intrinsic <- 5 ;
  `fixity        : enum intrinsic    <- `infix ;
  `associativity : enum intrinsic    <- `none ; // No "a == b == c" allowed!
  `type-class    : reference intrinsic <- [ `Numeric, `Character ] ;
  `sealed        : boolean intrinsic <- false ;
] ;
```

### 4.2.5 `function`

#### 4.2.5.1 Introduction

In Turned-C, a `function` is a first-class, context-driven distillery. While other languages treat functions as fixed, named routines, Turned-C defines them as independent, scoped blocks of logic. A function’s name is merely metadata—a handle for invocation—rather than an intrinsic part of its operational identity.

The core mission of a function is to act as a filter: it accepts an injected execution context, operates upon that data, and produces a well-formed result.

#### 4.2.5.2 Definition

A `function` is a scoped entity containing a sequence of definitions and expressions. In its unbound state, it is a pure Lambda. When associated with a Symbolic Identity (a name), it becomes an invokable entity within the Registry of Intent.

- **Contextual Execution:** Unlike traditional models, a function does not possess a static parameter list. Instead, it relies on Context Augmentation. During invocation, typed names are assigned values and "flowed" directly into the function’s scope. This extends the execution context dynamically, allowing the function to operate on the provided data without "magic" global state or rigid procedural boundaries.

#### 4.2.5.3 Technical Details

- **Context Visibility:** The context in which a function is defined determines a portion of the data it can access at run-time. However, due to the "reparenting" process enabled by the "Testability" aspect of the Provably Correct Theorem, this is not guaranteed. The `requires` annotation is a core mechanism for specifying and enforcing required context, making such guarantees explicit.
- **Augmentation Structure:** As described above, a function does not have a traditional, fixed parameter list. Instead, it defines required and optional augmentations to its execution context, which are injected directly into the function’s scope at invocation.

#### 4.2.5.4 Implementation Notes

- **First-Class Status:** Functions are first-class entities and may be passed, returned, or stored like any other value.
- **Binding and Rebinding:** Functions may be associated with different symbolic identities (names) or invoked anonymously.
- **Testability:** The design supports advanced testability and reusability by allowing context to be injected or reparented as needed.
- **`requires`:** Failure to satisfy a requires contract results in a Contextual Breach (Pass 4), preventing the 'Refinery' from starting with missing parts.

##### 4.2.5.5 Testability and Context Reparenting

The `Provably Correct Theorem` states that any project should be built from progressively larger and larger pieces, each layer being testable and "provably correct" to produce a "Provably Correct" and hopefully bug-free finished product. As it is one of the five pillars of Turned-C, its reach can be felt in most of the language’s design, but specifically in the `function`, the "transformer" and "filter" of data in the pipes of the refinery.

- The `requires` keyword is deceptively simple: it works by saying "these are names in my defining environment that I require to perform my task" for the function. But it is complex in practice: there are rules about how the "new home" of the function may override those names, for instance, providing a means to test individual functions in a `structure` standalone by use of a `mock`.
  - Put more simply: The requires keyword is the formal 'ID check' at the function's door. It ensures that whether you're running in production or under a test harness, the function has exactly what it needs to be Provably Correct.

- **Context Reparenting:**  
  In Turned-C, a function’s execution context is not statically bound to its original definition environment. Instead, when a function is invoked or tested, its required names can be “reparented”—that is, supplied or overridden by the caller or test harness. This allows the function to operate in a new context, with dependencies injected or mocked as needed.  
  - This mechanism enables isolated, deterministic testing: a function that depends on external state or collaborators can be exercised with controlled, test-specific values, without modifying the function’s code or global environment.
  - The language enforces that all required names (as declared by `requires`) must be satisfied in the new context, either by inheritance from the original environment or by explicit injection at the call site.
  - If a required name is not provided, or if the reparenting would violate the function’s contract, a compile-time or run-time error is issued, ensuring testability and correctness are preserved.

##### 4.2.5.5.1 Syntax and Resolution
**NOTE:** This sub-section is an attempt to restate, for ease of reference, the mechanics of the `requires` annotation in a distilled form. If you have any questions, please verify that they have not already been answered in Section 1.5 of this document before bothering the language implementor(s).

- Notation: The `requires` keyword accepts either a single symbolic name (e.g., `@[ requires: logger ]@`) or a comma-separated list enclosed in square brackets (e.g., `@[ requires: [db, config, auth] ]@`).
- Resolution: Each name provided to requires must resolve to an existing entry in the Symbol Table at the time of the function's definition and have direct visibility in the functions defining scope.
- Type Inheritance: The compiler extracts the Type, Type-Class and Metadata from these entries to establish the function's "Required Context."
- Registry Check: During Pass 4 (Semantic Validation), any invocation or reparenting operation is checked against this contract. If the provided context does not satisfy the requirements of the original symbol-table entry, a Contextual Breach is issued.

### 4.2.6 `variable`

#### 4.2.6.1 Definition

If a `function` is the machinery of a refinery, a `variable` is the oil that machinery works on.

In Turned-C, this meta-type signifies that the Symbolic Identity it is bound to is designed to encapsulate a single, atomic unit of data. It ascribes no extrinsic meaning—neither `magnitude` nor `value`—representing only a single, indivisible unit of information.

#### 4.2.6.2 Technical Details
- A variable must always reference a single, concrete storage location.
- The type and mutability of a variable are determined by its associated type and capabilities, not by the `variable` meta-type itself.
- Variables cannot--_must_ _not_--be reinterpreted as code or logic blocks.
- The `variable` meta-type is never inferred by the compiler. Its presence is a deliberate Symbolic Binding, while its absence serves as the linguistic signal for 'untyped memory' or FFI-compliant pointers.

#### 4.2.6.3 Implementation Details (provisional wording)

- **Pass 4 (Semantic Validation):**  
  The compiler verifies that every variable is bound to a concrete storage location and that its type and mutability are consistent with its declared capabilities. Any attempt to invoke a variable as logic results in a Category Breach (Pass 4), as the symbol lacks the required `operator` or `function` meta-types.
- **Pass 6 (Emitter):**  
  During code generation, variables are resolved to their physical storage addresses or offsets. The emitter ensures that all variable references are mapped to valid, concrete memory locations, and that no variable is left uninitialized or abstract.

#### 4.2.6.4 Implementation Note

- For strict rigor and explicit testability, there should be a well-defined "empty" or "uninitialized" meaning for identities with the `variable` meta-type. For a raw `variable` without additional type information, this is currently undefined; the interpretation of a "default value" is left to each type, if it differs from the default state of the allocated storage.
- It is the opinion of the author of this document—the "creator," if you will, of Turned-C—that an explicit value or representation for a raw, untyped `variable` is desirable. However, the precise form this should take remains an open question.

## 5. Other Keywords

These are hard-coded primitives essential for language grammar and control flow.

| Keyword   | Description |
|-----------|-------------|
| `define`  | Defines a name in the symbol table. |
| `DEFAULT` | Universal predicate lambda (always true). |
| `true`    | Boolean true. |
| `false`   | Boolean false. |
| `intrinsic` | Attaches a built-in, compiler-recognized behavior or property to a symbol (type, operator, etc.); enables special handling or optimization not available to user-defined logic. |

### 5.1 Introduction

These five "misfits" of the refinery also for a chunk of "framework" that holds the "Refinery" of Turned-C together and they deserve a better name than "Other Keywords" or even "Semantic Keywords", but such has eluded me since the birth of this document as a list of keywords with simple, one or two sentence definitions attached.

### 5.2 `define`

#### 5.2.1 Definition

- The "Genesis Predicate": This keyword instructs the Syntax Tree Walker to prepare for a potentially complex definition sequence. It acts as the "Dispatch Office" for the Refinery, signaling that the following Genesis-Marked (`#`) identity is about to be populated with its meta-type, attributes, and operational logic.
- A `define` "operation" is permanent and immutable within its scope; once a `name` is defined--that is, a "Symbolic Identity" is created--it cannot be "re-defined" or shadowed without a Registry Breach.

#### 5.2.2 Axiomatic Contract

- **Uniqueness:** Each `define` statement creates a unique Symbolic Identity within its scope. Any attempt to redefine or shadow an existing name results in a Registry Breach (semantic error).
- **Immutability:** Once defined, a Symbolic Identity’s association (name, meta-type, and core attributes) is fixed for the lifetime of its scope.
- **Scope Boundaries:** The visibility of a `define`d identity is strictly bound to its Lexical Scope. In accordance with the "Single Meaning Philosophy", names cannot leak or be implicitly shadowed; they exist as absolute truths within their defined boundaries.
- **Foundation:** All higher-level constructs—types, functions, variables, operators—are ultimately rooted in a `define` operation.


### 5.3 `DEFAULT`

#### 5.3.1 Introduction

It's time for a little bit of frank honesty: `DEFAULT` exists to meet the requirements of the "Single Meaning Philosophy".

You could easily replace it with `[[ true ]]` in action: this is valid Turned-C and will work. But it feels like a "magical hack", like one of the old "POKE" commands on an Apple ][ or a C64. It also presents a bit of a "mixed meaning"--after all, `true` is a _value_, not an expression.

So `DEFAULT` exists--a way to signal intent and provide a clear, well-defined single meaning that is impossible to misunderstand.

#### 5.3.2 Definition

This is the symbolic name for the `Universal Predicate Lambda`—the ultimate "match-anything" construct. While technically not a predicate lambda itself, it represents a default block of comparison logic: one that, by definition, matches any value, always yielding `true`.

#### 5.3.3 Implementation Details

- `DEFAULT` may be implemented as an alias for the boolean-value keyword `true`.
- **The "Catch-All" Rule:** As defined, `[[ DEFAULT ]]` matches any input, regardless of context or value.
- **The "Terminal" Constraint:** Because of its catch-all nature, `DEFAULT` must always be placed as the terminal branch of any selection construct (e.g., `match`, `switch`). Placing it earlier will short-circuit the logic, causing all subsequent branches to become unreachable ("dead code").

#### 5.3.4 Technical Details

- **Pass 4 (Semantic Validation):**  
  The compiler enforces that `DEFAULT` appears only as the final branch in any selection construct. If `DEFAULT` is found in a non-terminal position, an Unreachable Branch Breach (Pass 4) is issued, as the universal predicate renders all subsequent logic inert.

### 5.4 `true` and `false`

#### 5.4.1 Definition

- **Axiomatic Constants:** These keywords represent the immutable, primitive states of boolean logic.
- **Binary Sovereignty:** In Turned-C, `true` and `false` are the only valid results for any Predicate Lambda or comparison operation.

#### 5.4.2 Technical Details

- **Pass 4 (Semantic Validation):** The compiler ensures these constants are strictly bound to the boolean Type-Class.
- **Pass 6 (Emitter):** Maps `true` to the backend’s canonical "set" bit (LLVM::i1 1) and `false` to the "clear" bit (LLVM::i1 0).

### 5.5 `intrinsic`

#### 5.5.1 Definition

- **Contractual Identifier:** This keyword designates a property, behavior, or metadata field as a built-in requirement necessary for a symbol to satisfy its Type-Class contract.
- **Logic Anchor:** It transforms an ordinary definition into a functional primitive recognized by the compiler’s internal logic. This enables the symbol to participate in core language mechanisms such as the Numeric Bridge, Dispatch Table, and other foundational systems.

#### 5.5.2 Axiomatic Contract

- **Membership Verification:** `intrinsic` is the primary mechanism for marking members of a type that fulfill the requirements of a Type-Class. During Pass 4 (Semantic Validation), the compiler verifies these fields to ensure the type satisfies its legal framework.
- **Immutable Association:** An `intrinsic` property cannot be shadowed, overridden, or reinterpreted; its meaning is fixed to the specific behavioral protocol it supports.
- **Optimization and Handoff:** While often used for Type-Class adherence, `intrinsic` also enables the Pass 6 Emitter to map specific symbols directly to optimized backend primitives (e.g., `LLVM::sdiv`) when flagged as core operational hooks.

#### 5.5.3 Technical Details

- **Pass 4 (Semantic Validation):** The compiler checks that all required `intrinsic` fields are present and correctly typed for Type-Class membership. Any missing or misapplied `intrinsic` results in a Registry Breach or Semantic Breach.
- **Pass 6 (Emitter):** Intrinsic-marked symbols may be mapped directly to backend instructions or runtime primitives, bypassing generic dispatch for maximum efficiency.
