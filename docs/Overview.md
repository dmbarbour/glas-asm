# Project Overview

My goal for this project is to lift assembly-level programming into something I feel comfortable directly using as a primary language, and also a suitable medium for directly distributing software.

Rough approach:

- design language for specifying binaries in general
- libraries for CPU targets, executable formats, etc.
- syntactic sugars similar to conventional assembly

This document is both detailed overview and brainstorming.

## Desiderata

Regarding binary specification, relevant system-level goals include:

- reproducibility: easily share binary by sharing code
- extensibility: easy variants without invasive edits
- adaptability: outcome depends on a system definition
- modularity: share definitions, incremental computing

These goals entangle: System definition must also be reproducible, extensible, and modular. A system definition can provide modules to the binary specification for adaptability. Transitive dependencies must be reproducible. Incremental computing should share work (local verification, partial evaluation, etc.) across similar but non-identical system definitions.

Regarding the programmer experience:

- looks and feels like well-documented assembly code
- automated verification of assumptions and reasoning
- easily visualize assembly, interactive debug views
- easy live coding, continuous feedback during change 
- flexible metaprogramming with macros and DSLs

These present significant design challenges. I'll generally prioritize system-level features over experiential properties, but some difficult design decisions are required.

## Why Another Language?

Some existing languages align with some of my desiderata. For example, F\* or Vale support reasoning, and Unison supports reproducible modularity. But none attend the whole range, much less offer the programmer experience I want. Historically, assembly languages haven't received a lot of love from programming language designers.

## Semantics

I propose to build upon a pure, untyped lambda calculus with lazy evaluation, extended with annotations. I do not believe untyped lambda calculus or lazy evaluation require introduction

Annotations are simply structured comments for tooling, flavored identity functions. Annotations may generate auxilliary outputs (e.g. logs, cache), influence compile-time performance (e.g. memoization, acceleration, parallelism), or detect and diverge early on errors (e.g. assertions, types). However, annotations are not observable *within* the lambda calculus.

We can model data, effects, and OO-style inheritance upon this foundation:

- Data includes numbers, lists, symbols, and dicts. Data is tagged to support ad hoc polymorphism and flexible pattern matching similar to dynamically-typed languages. Data is logically [Church or Scott encoded](https://en.wikipedia.org/wiki/Church_encoding). 
- Even within a pure computation, 'effects' such as writer, state, exceptions, etc. are convenient for composing behavior. I propose to model effects monadically, adapting [Oleg Kiselyov's Free-er Monads](https://okmij.org/ftp/Haskell/extensible/more.pdf).
- Implementation inheritance is useful for extensibility, both for large specifications and modeling extensible keyword parameters with defaults. For this, I adapt Fare's [Prototypes: Object-Orientation, Functionally](http://fare.tunes.org/files/cs/poof.pdf).

Users do not receive direct access to lambdas. Instead, user-defined functions are modeled as tagged function objects with keyword parameters and a monadic 'result'.

## Performance

There are at least a few use cases for intensive number crunching at assembly time: We might include a physics simulation for ray tracing or CGI. We could introduce a little cryptography for homomorphic encryption or other reasons. Perhaps we'd like to integrate LLMs and prompts to guarantee reproducibility.

In this context, the most useful performance pattern is *acceleration*: Develop a memory-safe DSL for an abstract CPU or GPGPU, and implement an interpreter. Annotate the interpreter to be replaced by an assembler built-in. The assembler built-in compiles the DSL to evaluate on actual hardware.

Aside from acceleration, annotations may guide parallelization of pure computations. A simple annotation could ask the assembler to perform some computations in background threads. With a suitable configuration, we can utilize remote processors or GPGPUs.

Annotations may also guide memoization and caching. This won't help when an assembly is first computed, but can reduce rework across similar assemblies. Depending on configuration, we can potentially *share* the cache (and work) between multiple users, perhaps supported by PKI signatures and certificates.

*Note:* Modules are implicitly lazily loaded and subject to parallelism, memoization, and caching. Annotations extend this to the granularity of individual expressions.

## Modularity

A module is represented by a file. Modules may reference other files within the same folder or subfolders, or content-addressed remote folders. We forbid Parent-relative ("../") and absolute filepaths. These constraints ensure folders are location-independent and temporally stable, yet remain editable by conventional means.

A content-addressed remote folder is uniquely identified and authenticated by a secure hash of content or DVCS revision history. However, they are not necessarily *located* by secure hash. Instead, the reference includes a list of likely locations, e.g. URLs, perhaps paired with branch or tag names to support shallow cloning. I intend to initially support git, perhaps mercurial.

Modules are parameterized, an argument provided at 'import'. By default, the same argument propagates transitively across imports, serving as an implicit environment. A motivating use case is aggregating references to remote folder into just a few locations to simplify maintenance. This also supports adaptability, providing a configurable system definition.

The module system supports lazy loading and caching. This allows users to specify a large namespace from which only a small fraction is used.

### Configuration

The assembler implicitly loads a module based on the `GLAM_CONF` environment variable or an OS-specific default, i.e. `"~/.config/glam/conf.g"` in Linux or `"%AppData%\glam\conf.g"` in Windows. This module specifies configuration 'conf'. A small, local user configuration typically extends a large community or company configuration, imported from DVCS.

The configuration specifies the implicit environment argument passed to an assembly module. Configurations may also influence logging, testing, caching, cloud computing, quotas, etc..

### Assembly

Any module that exports a binary specification can serve as an assembly. The assembler CLI shall support command-line arguments to flexibly select file and binary, and even provide a subset command-line arguments to the binary spec.

The assembly module is parameterized by a configured environment. Some assemblies may be script-like, mostly composing resources defined in this environment. Others may stand alone, ignoring your reality and substituting their own, explicitly controlling import arguments.

Note that assemblies specify binaries. The assembler has no built-in knowledge of CPU architectures or assembly mnemonics. We could 'assemble' images, configuration files, or documents. But the assembler shall provide syntactic sugar suitable for coding CPUs directly.

### Sharing

Declared symbols are unique per import. Thus, if you import the same file at five locations, you'll have five distinct sets of unique symbols. This naturally extends to abstract data types and multiple inheritance. There are some use cases for this, but in most cases it's preferable to import only once then share definitions through the module parameter or object model.

Perhaps as a linter rule, we might warn developers if a file or DVCS repo is loaded more than once within an assembly or configuration and encourage sharing instead.

## Reasoning

The untyped lambda calculus does not require type annotations. However, type annotations are still useful to express and verify assumptions or isolate blame. I propose to support gradual typing with an ad hoc mix of static and dynamic checks. However, it is awkward to encode sophisticated properties into a type system.

Annotations also express assertions, thus serve a role in testing. Instead of pure boolean expressions, I propose to model assertions as effectful computations, especially including *non-deterministic choice*. This extends assertions to fuzz testing, property testing,constraint satisfaction. Other effects may provide heuristic guidance for search or feedback for visualization.

Instead of relying on fine-grained type annotations, I imagine assembly programs will simultaneously write out a binary (or staged precursor) and a representation of abstract meaning as a constraint system, perhaps borrowing from [Cornell SHErrLoc project](https://research.cs.cornell.edu/SHErrLoc/). We then 'assert' constraints at suitable boundaries.

Ideally, we verify all functions terminate based on static analysis of the assembly. Annotations may provide hints to guide analysis. However, can be difficult to prove and doesn't imply *timely* termination even when proven. Ultimately, we'll likely rely on configurable quotas to robustly reason about termination.

## Data

The basic data types are numbers, lists, symbols, and dicts. Data is immutable, i.e. to 'update' a dictionary returns a new dictionary with the update applied. This can be efficient due to structure sharing and clever encodings under the hood.

- Numbers are bignum integers and rationals. Arithmetic is exact, but maintaining full precision across many steps becomes intractable. High-performance number crunching is feasible via acceleration of an abstract CPU or GPGPU.
- Lists are used for all sequential structures. Large lists are represented by finger-tree ropes under the hood to efficiently support most operations. Binaries are lists of small integers (0..255) and optimized very heavily.
- Symbols are abstract data that support only equality comparisons. Symbols are constructed in two ways: modules declare guaranteed-unique symbols when imported, and any composition of basic data can be abstracted as a symbol.
- Dictionaries are finite key-value lookups where the keys are symbols. Singleton dictionaries are useful for modeling tagged unions. Dictionaries do not support iteration, providing a simple basis for data abstraction. 

User-defined data types will mostly be modeled as tagged unions. Tagged unions naturally support abstract data types: modules may choose to not export the symbol, providing access indirectly through smart constructors and restricted observers. They are also a source of extensibility: we can extend pattern matching with new symbols.

## Effects

It isn't difficult to model effects with pure functions. We essentially need two ways of returning from a function: final return value, or yield an effectful request paired with a continuation that receives the response. 

In context of an untyped lambda calculus, request and response are any data. Modeling the continuation needs some attention for performance. Roughly, the solution is to explicitly model a list of 'atomic' lambda continuations instead of one big lambda. Then implement a queue that rewrites left-associative monadic compositions to right-associative. This is discussed in sections 2.6 and 3 of Oleg's paper.

I intend to make effects the default, a standard feature of function objects and expressions, then explicitly restrict effects by controlling handlers in a given scope. This also enables the assembler to abstract the continuation, perform flexible inlining and partial evaluation, etc..

## Objects and Mixins

Pure functions can model stateless objects with mixins in terms of open recursion through a deferred fixpoint. A basic object model is `Dict -> Dict -> Dict` in roles `Base -> Self -> Inst`. Here, 'Self' supports open recursion via deferred fixpoint. 'Base' represents the mixin target or host environment.

        mix child parent = λbase. λself.
            (child (parent base self) self)
        new spec env = fix (spec env)
        # fix is lazy fixpoint

Most observations on Base or Self will diverge on fixpoint or compromise extension. A useful exception is 'ifdef' on Base, i.e. mixins may express simple conditional behavior.

Inheritance and override are useful tools for extensibility. For example:

- We can model functions as objects where we override keyword parameters then read a result. We can easily introduce new parameters without breaking existing callers.
- We can model grammars as objects where methods represent parser combinators. This makes it easy to extend the grammar with a new rule for parsing integers, for example.
- We can model logic systems as objects. We can easily edit tables, tweak logic rules, or perhaps integrate multiple logic systems via mixins and a few more rules.

Note that implementation inheritance is the focus here, not subtyping or substitutability. Extensibility is about producing useful variants on a system without invasive edits. In general, extensions may express non-monotonic updates, e.g. filtering a table or disabling a parse rule, which may conflict with substitutability.

*Aside:* I propose to call 'specification' or 'spec' what most OO languages call 'class'. In part to avoid connotations of subtyping.

### Multiple Inheritance

Multiple inheritance becomes useful when composing systems that build upon common foundations or frameworks. In these cases, we often want to share state through the framework without initializing things twice.

Multiple inheritance requires reifying the dependency graph, recognizing common ancestors, and applying a linearization algorithm (such as C3) that eliminates redundant components and preserves a consistent override order.

Although we cannot compare functions, a simple workaround:

- pair basic objects with proxy identifiers, e.g. symbols
- introduce annotations to assert values are equivalent
- linearization compares proxies and asserts equivalence

The assertion may be best effort, reporting an error when two functions are obviously distinct, or a warning if they aren't obviously equivalent (i.e. referentially or structurally). In practice, it is easy to construct proxy identifiers that will be unambiguous within the dependency graph outside of contrived circumstances.

*Note:* If we 'import' a framework three times, then inherit from all three, those would generally be considered three unique frameworks. 

### Symbolic Names

In context of mixins and multiple inheritance, using strings as names leads to ambiguities and collisions. The big culprits are generic, contextual names like "map" or "draw". There are some other weaknesses, too: no feedback upon deprecating a name, no convenient approach to 'private' names local to a mixin.

A robust solution is to declare unique symbols for distinct meanings or interfaces. Access to the symbol is routed unambiguously through the module system, subject to qualification or aliasing. If we delete or deprecate a symbol, we can easily chase the errors. Private names are easily modeled as symbols not exported.

A relevant concern with symbolic names is risk of namespace clutter. I hope to mitigate this syntactically, e.g. with 'using' directives. Also, we might stick to strings for keyword arguments to function objects.

### Type-Indexed Functions

*Aside:* Modeling typeclasses, multimethods, etc. as module-system 'globals' is very awkward due to potential conflicts. Better to model them with clear scope, e.g. within a specification.


### Checking Assumptions

In context of defining a mixin, most observations on 'Base' or 'Self' either compromise extensibility or diverge on fixpoint. A useful exception is observing which names are defined in 'Base': we can support 'ifdef' or check assumptions about whether a mixin introduces or overrides a name.

We still have several options: Holistic observations after a mixin or specification is defined. Define test methods for most things. Assert within method definitions.

## Syntax and File Extensions

Users may explicitly specify a front-end compiler when importing a file. In the trivial case, this could directly return a file binary. 

A front-end compiler can be expressed as a parser combinator: a monadic function with limited effects for backtracking, declaring symbols, defining specifications, and so on. Ideally, we carefully design this monad to uniformly support syntax highlighting, error isolation and reporting, edit suggestions, editable projections, source mapping for blame or debug views, etc.. This is a significant design challenge that I'll expand on later.

If the user does not specify a front-end compiler, the assembler will default to a built-in for ".g" files or query the import argument for other file extensions. (This might be expressed as a general 'compile' function that takes a cleaned up file extension as an extra argument. Details TBD.)

Usefully, we'll apply the same rule at the assembler CLI, allowing assemblies to be specified in any front-end syntax supported by the configured environment.

*Note:* Namespace macros shall essentially be front-end compilers *hold the file*, i.e. importing with just the front-end compiler expression instead of the file context or input, preserving relative paths to the current file for further imports.


## Assembly Ideas

### Content-Addressed Memory

Instead of manually managing layout, it is feasible to 'jump' to code or 'load' data and have these converted later to memory addresses based on content and metadata. For mutable targets ('.bss' or '.data') we can include a symbol in metadata to distinguish multiple instances.

This design significantly simplifies metaprogramming: we can allocate resources and computed subroutines as needed without predicting them in advance. To restore control over memory layout, an assembly may define interfaces to guide layout based on provided metadata.

If writing out assembly mnemonics is modeled effectfully, we can feasibly capture the data and integrate the final location after we've collected all the data.

### Local Labels

Content-addressed memory doesn't work for local labels used in branches and jumps. But we still cannot assume the label target is defined before it is used, i.e. we can jump to something later within the assembly program.

Labels may need specialized syntax within an assembly definition.

### Structured Stacks

Although we can directly manipulate the stack pointer, it would be convenient if we can automatically allocate stack locations based on usage, similar to how we allocate '.bss' sections. This seems feasible if we automatically accumulate sizes for referenced stack data elements.

*TBD:* Generalize so we can leverage yet again for associative heap allocations.

## Verification

Verification is driven by annotations. Though, it may also be relaxed or disabled by configuration.

There are two problems here: verifying the assembly process, and verifying the generated binary. In general, it's a lot easier to verify the process, so it's convenient to align verification of the generated binary with verification of the process.

But it is also feasible to support something like Hoare logic at the level of registers, and build up precise descriptions of the behavior for every CPU instruction. We might need to pass resulting constraint systems to an SMT solver, perhaps as an annotation, a specialized assertion of sorts.

TBD: review SMT solvers, or perhaps [SHErrLoc's](https://www.cs.cornell.edu/projects/SHErrLoc/) intermediate constraint language.

### Assertions

We can easily have assertions at many layers: modules, specs or objects (as a whole), and functions (local per call). Assertions modeled as annotations cannot influence output, but we coould feasibly support effectful assertions that raise an error. Probably want to distinguish the two.

### Types

Pervasive use of tagged data supports flexible pattern matching and ad hoc polymorphism. Effectively, we have a dynamically typed language. Though, in context of compile-time metaprogramming, this 'dynamic' refers to a late stage after modules are loaded.

However, it is feasible to perform static analysis to detect some errors earlier. We can introduce type annotations to guide this analysis.

Usefully, type annotations can potentially express 

Most types cannot say much about the assembly code, only about the metaprogramming framework. 

only about the framewor

Ultimately, we'll pro also want the opportunity to analyze assembly

These 'types' apply to the compile-time metaprogram. It may be feasible to support some assertions that are contingent on assembly, but it certainly is not trivial.

## Tests

To detect errors earlier, we can support tests per assembly module. 

A simple way to express tests is a simple `Integer -> Boolean` function.  

To mitigate, we can express some tests per module, e.g. as assertions. 

Of course, we can expres tests for modules, perhaps even per specification.  tests for modules



Ideally, we can detect many errors earlier, before modules are shared. 

We can perform some type inference and check for consistency issues. But 


Although it is better to detect errors later than never, it is best to detect errors early. One relevant boundary is to detect errors within a module before it is shared and used in an assembly.

We can develop static analysis leveraging dataflow, abstract interpretation, etc. to detect some errors ahead of time. But, in context of higher-order metaprogramming, inference is limited. To support analysis, we can introduce annotations 


for assembly modules that search for problems. We can introduce type annotations to precisely describe programmer assumptions.






given assembly languages have no runtime, and all this computation is

this 'dynamic' must be understood in the context of assembly-time stages. 

The 

 to dynamically typed languages. 

though 'dynamic' in this context refers to the 'evaluation' stage of assembly, which is still 

We may benefit from analysis to  prior to evaluati

Although all computation is static, we can still distinguish stages such as parse and evaluation.

 Although f




We build upon the untyped lambda calculus. However, t



Static dataflow analysis can potentially detect some errors before evaluation.







 may detect obvious errors, e.g. providing a number where a function is expected.

The 


I'm envisioning gradual types, partial types, using types to help detect or isolate errors. In some contexts, failure to prove a type might be a warning instead of an error, and only an active disproof is an error. 

Non-trivial type checking may be separated from the assembler, run as an independent operation with its own output and debugger.

## Instrumentation

logging, Profiling, 

## Assembler CLI

This project should develop an initial assembler executable

 is parameterized by an assembly file and specification name.

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

## 


###

## Modeling Objects

## Content-Addressed Memory


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








