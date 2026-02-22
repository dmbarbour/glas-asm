# Project Overview

The goal for this project is to lift assembly-language into something that I'd feel comfortable with using as a primary language.

Like any language, assembly benefits from expressive metaprogramming, non-invasive extensions, support for composition and adapter layers, flexible annotations, and a scalable module system. Such compile-time features are focus of this language.

This is a language without a 'runtime'. The only thing we do with assembly code, other than develop and maintain it, is extract reproducible, deterministic binaries. The focus is clearly executable or linkable binaries, but extracting a GIF or similar is also okay. 

## Underlying Semantics

We'll build upon a pure, untyped lambda calculus with lazy evaluation, annotations, and unique identifiers. I don't believe lambda calculus or lazy evaluation need further introduction.

Annotations are structured comments for tooling, applied to lambda terms. Annotations do not influence formal output from a valid assembly. However, they may raise errors (e.g. types, assertions), influence performance (e.g. memoization or acceleration), and generate auxilliary outputs (e.g. logs, cache).

Unique identifiers are useful for memory allocations, multiple inheritance, and abstract data types. We borrow identity from the filesystem: a module may declare unique symbols at its toplevel. In case of metaprogramming, we thread and compose identities and use annotations to check assumptions about unique association.

## Data

The assembler and assembly syntax has built-in support for common data types: numbers, lists, dicts, and symbols. Binaries are encoded as lists of small numbers (integers 0..255), but heavily optimized (e.g. finger-tree ropes). 

Logically, all such data is Church or Scott-encoded, tagged so we can discriminate types (e.g. list vs. dict), and sealed via annotations. But we'll use efficient representations under-the-hood.

I won't expound on the conventional lists and dicts, just on what needs more attention:

### Numbers and Arithmetic

We'll support bignum integers and rationals with exact arithmetic by default. It's best to be explicit about lossy operations, e.g. due to overflow or rounding.

### Symbols

Symbols are abstract identifiers. The only operation on symbols is to compare two symbols for equality. There are two ways to construct symbols:

- Unique symbols may be declared at module toplevel. This is structurally limited to a small, finite number of symbols.
- Data composed of basic lists, dicts, numbers, and symbols can be converted to a symbol. Symbols constructed from the same data are equivalent.

In context of metaprogramming, users will often combine a unique symbol with generated data to create as many symbols as needed.

With annotations, we can express that a symbol is *uniquely associated* with a term, at least within the current assembly. This is useful when symbols are proxy for identity. If the same symbol is associated with two distinct terms, the assembler can report error.

### Abstract Data

Annotations can express 'seal' and 'unseal' of terms with a symbol. Except unsealing with the same symbol, any attempt to observe or interact with sealed data will raise an error. By using a declared symbol and controlling exports, modules can robustly enforce ad hoc data abstraction.

Although our untyped lambda calculus is untyped, data abstraction effectively implements a lightweight type system. Obvious errors may even be detected ahead of evaluation.

## Regarding Types

Earlier, I said "untyped" lambda calculus, then promptly mentioned types as one use case for annotations. 

I'm envisioning gradual types, partial types, using types to help detect or isolate errors. In some contexts, failure to prove a type might be a warning instead of an error, and only an active disproof is an error. 

Non-trivial type checking may be separated from the assembler, run as an independent operation with its own output and debugger.

## Modeling Objects

Inheritance and override is useful when constructing large assemblies. We can express that a new assembly is almost the same as an old one with just a few overrides. Of course, effective use requires anticipating likely points of change.

A simple object model supporting mixins is `Dict -> Dict -> Dict` in roles `Base -> Self -> Instance`. Here 'Self' represents an open fixpoint, supporting open recursion and overrides. 'Base' represents a mixin target or host system.

        mix child parent = λbase.λself.
            (child (parent base self) self)
        new spec base = fixpoint (spec base)

Unfortunately, this model does not support *multiple inheritance* (MI). Users may manually apply multiple mixins, but it's easy to accidentally duplicate or reorder operations in inconsistent ways (e.g. for mutexes or serialization).

To implement MI involves constructing an intermediate dependency graph then applying a *linearization* algorithm that drops duplicates and preserves critical constraints on method resolution order (cf. [C3 or C4](http://fare.tunes.org/files/cs/poof/ltuo.html#(part._.C4))). After linearization, we fold the earlier 'mix' function.

Linearization assumes an ability to compare nodes for equality. We cannot compare functions, but we can pair each function with a uniquely associated identifier as proxy. But it can be awkward to provide unique identifiers in every case.

I propose to use both models: 'spec' definitions in the module toplevel with multiple inheritance, and the basic model for anonymous or short-lived objects like *Parameter Objects*. 

*Note:* A potential concern with MI is collisions on 'meanings' of names. To mitigate this, we can syntactically distinguish 'introducing' a name versus overriding one. This at least lets us detect accidental collisions. Ideally, we can also resolve through aliasing.

## Parameter Objects

I propose one big parameter object, i.e. `ParameterObject -> Result`, as the default calling convention for user-defined functions. This offers multiple benefits:

- defaults, parameter object is expressed as a mixin that 'overrides' defaults, can easily extend with new methods
- refactoring, can build parameter objects over multiple steps, or as a composition of mixins, or other abstractions
- var-args, generally provide positional parameters as 'args' list
- implicit 'env', implicit parameters by default

In the syntactic integration, we can reserve 'args' for positional parameters, forward 'env' by default to further calls, and treat other parameters as keyword arguments.

## 


## Syntax Ideas 

### User Data

I hope to develop a lightweight syntax for users to define data constructors, leveraging unique symbols for pattern matching and sealing. We can leverage smart constructors, active patterns, and type annotations.

### Texts

The syntax will support '"'-quoted strings of printable ASCII (0x20-0x7E minus 0x22) as a limited embedding for binary data. There are no escape characters, but users may apply a functions to process the string, e.g. to support tabs and newlines, hexadecimal, or base64.

For large texts (and binaries), users may simply load a file from the module system as a binary. This may eventually support DSLs or higher-level general purpose languages.

*TBD:* I'll want a lightweight syntax for building lists. A syntactic sugar for a writer monad (or free monad used as writer) seems viable.

## Annotations

Annotations are structured comments that support instrumentation, verification, performance, and other tools. Annotations must have no impact on primary outputs except to halt compilation with errors. But they may influence *auxilliary* outputs, such as:

- message logs: debug trace, warning
- compile-time performance, profiling
- caching for incremental compilation
- source maps for runtime debugging

To resist silent degradation, we'll report warnings for unrecognized or unsupported annotations.

## Acceleration

With guidance from annotations, compilers may substitute user-defined functions with built-ins. This pattern is called *acceleration*. The aforementioned *Data Extensions* are essentially standard accelerators.

An especially useful pattern is to accelerate an *interpreter* for a virtual CPU or GPGPU, such that we can leverage low-level hardware for number crunching at compile-time. 

## Modularity

Each module is represented by a file. Modules may reference other files within the same folder or subfolder, or in remote DVCS repositories. Parent-relative ("../") and absolute filepaths are forbidden. Dependencies between modules must form a directed, acyclic graph.

Under these constraints, *folders* are easily copied and shared between users or projects within a network. 

To support whole-system versioning and curation, a configuration may translate DVCS refs, e.g. to redirect an unstable branch to a stable tag, or replace a distrusted repo with a privately-controlled fork.

We aren't restricted to code files. Modules may load file binaries to support embedding or metaprogramming.

*Note:* Module dependency graphs may grow very large. We'll rely on lazy loading, caching, and incremental compilation for performance.

## Configuration

A configuration is expressed as a separate module that defines a 'conf' spec. A user configuration can inherit and extend a community or company configuration from DVCS.

Configurations may influence compiler output, e.g. by specifying an `Object->Object` assembly adapter or redirecting DVCS references. However, most configuration options should be non-semantic, e.g. guidance for logging, caching, JIT, GC, or hooking up resources for acceleration like GPGPUs.

## Identity and Uniqueness

Although the lambda calculus has no notion of extrinsic identity, the module system does (via the filesystem). With a little compiler support, modules may declare a bounded, static set of abstract, unique identifiers.

Further identifiers can be computed via composition, e.g. a pair of identifiers is also a useful identifier.

However, 



Use cases include: 
- abstract data types
- multiple inheritance 
- memory allocation

In context of metaprogramming, we can work around limit of bounded static declarations by constructing relational identifiers through composition.

###

## Specifications

A specification or 'spec' is a toplevel module definition that supports inheritance and override of other specs. This is the main OO-inspired design pattern.

Assemblies, configurations, etc. are expressed as specs. This makes it easy to express a new assembly as a slight modification to an existing assembly, or let a user configuration override a community configuration. 

Multiple inheritance is convenient when composing features that build upon shared foundations, especially shared state.

## Modeling Objects

## Content-Addressed Memory

When expressing the assembly, users will directly jump to code or load data addresses with reference to the code or data, not the address. When rocessing the assembly, we'll aggregate the referenced resources and allocate required memory.

In case of immutable data ('.text' or '.rodata'), pure content addressing is sufficient. For mutable locations ('.data' or '.bss'), we may include a unique identifier as metadata so we can distinguish hundreds of 8-byte fields. Other metadata per field may include alignment requirements and layout hints.

This design significantly simplifies metaprogramming: we can allocate resources and computed subroutines as needed without predicting them in advance. To recover control over memory layout, an assembly may define interfaces to guide layout based on metadata.

## Assembly

At some point, we'll be writing out a sequence of ordered, parameterized assembly mnemonics. In some cases, we'll refer to external, content-addressed definitions. In others, we might refer to local labels for branch or jump instruction. And we may freely 'inline' assembly code. 

Ideally, as we write things out, we're also accumulating details such as which content-addressed structures the assembly references, or local labels for use as branch or jump targets. We may need a fixpoint or a separate pass to fill in final details.

This basic structure suggests something like a monadic structure for operating on the sequence of heuristics. But it is feasible to simply build a list of operators for ad hoc processing.

### Local Labels

In context of jump or branch operations, we'll often reference a local label within the assembly code. The notion of content-addressed locations does not apply in this case. But we cannot assume the label target is defined before it is used, i.e. we can still jump to something later within the assembly program.

### Structured Stacks

Although we can directly manipulate the stack pointer, it would be convenient if we can automatically allocate stack locations based on usage, similar to how we allocate '.bss' sections. This seems feasible if we automatically accumulate sizes for referenced stack data elements.

*TBD:* Generalize so we can leverage yet again for associative heap allocations.


## Tags and Adapters

We can model a tag as a namespace-layer function that, given an environment of adapters, picks one then applies it. For example, a "bar"-tagged Term, applied to `{ "foo":Op1, "bar":Op2 }`, returns `(Op2 Term)`. There are also many other ways to model tags, e.g. as singleton dictionaries. Perhaps abstract the tag encoding. 

I propose to basically tag everything with a little structural info. The compiler may introduce initial tags like "list", "dict", "int", "spec", "obj", etc.. Users may introduce new tags as needed. 

Pervasive use of tags effectively serves as a lightweight, dynamic type system. But it's all modeled within the untyped lambda calculus.

## Sketch of Syntax

TBD.

I'd like to avoid pervasive use of indentation. But we have specs, definitions within specs, etc. Perhaps we can simply use a visually distinctive section separators.

A literate programming style might also be nice.

## Design Concerns

### Reproducibility

If there is no error, primary assembler output is a deterministic binary on stdout. This binary must depend only on locally reproducible inputs: e.g. source code and a clear subset of command-line arguments (e.g. after '--'). *Note:* Auxilliary outputs (logs, warnings, cache, heat generated, etc.) are not restricted.

### Update Path

We can extend assemblers with new features, but we must respect reproducibility. Thus, attempting to use the features with the prior version of the assembler must generate an error. Conversely, we must also be able to deprecate old features, so long as we raise an error when an old assembly depends on the deprecated feature.

A viable approach is to forbid reflection (ifdef or iteration) on some dictionaries. This allows us to provide assembler features without letting binary output observably depend on assembler version.


...



If there is no error, primary assembler output shall be a deterministic binary, depending only on source code, command-line arguments, and optionally a binary input. If there is an error, a partial binary may be produced before halting with error.

Auxilliary outputs (logs, cache, heat generated, etc.) are less restricted and may depend on assembler version. Assemblers may introduce new 'features' but only if it results in errors.

independent of assembler version. Of course, it does depend on source updates and command-line parameters and source updates, of course.

This usually represents an executable binary or library, but nothing prevents leveraging assembly language as a simple calculator. 


That is, there are no implicit optimizers.

Though, it may depend on command-line parameters passed to the assembly. 

 modulo updates to source code and a few command-line arguments passed to the assembly. There is no hidden optimization step. 

The 'compiler' in this case is responsible for loading modules and lazily evaluating pure functions, ultimately extracting a deterministic binary from the module system.








