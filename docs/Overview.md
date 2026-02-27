# Project Overview

The goal for this project is to lift assembly-level programming into something I'll feel comfortable directly using as a primary language. A very high-level language for very low-level programming.

Like any low-level language, assembly benefits from effective composition and extension, a scalable module system, and expressive metaprogramming. As stretch goals, I'm also interested in automated verification and proof-carrying code.

The main operation on assembly code is to extract binaries, usually executable or linkable. Proposed approach: 

- design language for specifying binaries in general
- modules support CPU targets, formats like ELF, etc. 
- optimize standard syntax and structure for assembly

Ideally, binary outputs are reproducible, fully specified within the module system, independent of assembler executable. The assembler competes on performance, support for validation, debugging experience, etc..

## Why Another Language?

Although any popular programming language can generate binaries, most can be disregarded due to pervasive side effects that hinder reproducibility, caching, or parallelism. I'm also wary of module systems that express location-dependent code or depend on separate package managers and lockfiles.

I have other desiderata for user experience: pervasive extensibility by default, expressive metaprogramming and user-defined syntax, compose assembly projects into a larger system, type annotations as guidelines instead of trauma.

If it were just the reproducibility goals, I believe Unison, F*, Dhall, CUE, and a number of other languages could be adapted. But I'm not aware of any that fit my vision for the user experience.

## Modularity 

A module is represented by a file. Modules may reference other files within the same folder or subfolder, or content-addressed remotes. We forbid Parent-relative ("../") and absolute filepaths. The constraints ensure folders are location-independent and stable, a suitable unit for packaging and distribution.

Content-addressed remotes may provide several URLs for reliability, and a configuration may supply alternative locations. Regardless of origin, we rely on a secure hash of content or DVCS revision history to identify and verify. I propose to initially support git, mercurial, and zipfiles over HTTP.

Maintaining remote refs scattered across sources is painfully awkward. To mitigate, modules are parameterized. By default, we transitively propagate the parameter across imports, serving as a tacit environment. This enables users to separate and aggregate references into a few locations. But, instead of passing remotes as parameters, it is usually preferable to import immediately then pass a directly usable environment.

The assembly may also load files from the module system as binaries for further processing. Aside from embedding binaries, this is useful for metaprogramming with external DSLs. See *User-Defined Syntax*.

*Note:* Files or subfolders whose names start with "." are also hidden from the module system, reserved for tooling. For example, we could maintain signed manifests and certificates in ".pki/" or provide a cache proof hints ".proofs/". 

## Configuration

The assembler loads a module indicated by `GLAM_CONF` or an OS-specific default, i.e. `"~/.config/glam/conf.g"` in Linux or `"%AppData%\glam\conf.g"` in Windows. A configuration is expressed as a 'conf' specification within this module. A small, local user configuration typically extends a large community or company configuration, imported from DVCS. 

The configuration is parameterized by `GLAM_CONF_OPTIONS` (via override).

A primary purpose of a configuration is to define an assembly environment, the parameter passed to assembly modules. An assembly is free to ignore your environment and substitute its own, but this allows developers to also express script-like assemblies that borrow heavily from an assumed environment.

The configuration further specifies ad hoc options for logging, testing, caching, etc.. These may be runtime specific, subject to de facto standardization.

## Assembly

An assembly is typically specified in a separate module from the configuration. The module and specification are indicated by command-line arguments, perhaps defaulting to definition of 'asm'. An assembly spec may be further parameterized (via override) by a list of command-line arguments, e.g. after a '--' separator. The assembly specification should define a binary. 

This pushes most assembly-specific logic to libraries and effective syntax. 

## Lambda Calculus

I propose to build upon a pure, untyped lambda calculus with lazy evaluation and annotations. I don't believe lambda calculus or lazy evaluation need further introduction.

Annotations are structured comments for tooling. Annotations may generate auxilliary outputs (e.g. logs, cache), influence compile-time performance (e.g. memoization, acceleration), or help detect errors (e.g. assertions, types). However, other than optionally halting an erroneous computation early, annotations do not affect the assembled binary.

We model data, objects, and other useful design patterns upon this simple foundation.

## Data

We support a few basic data types: numbers, lists, symbols, and dicts. Data is immutable, leveraging persistent data structure for efficient update. Logically, all data is Church or Scott-encoded, but the details are abstracted. The assembler favors efficient representations under-the-hood.

### Numbers and Arithmetic

The assembly supports exact rational numbers and bignum arithmetic. We'll provide operators to explicitly convert to and from common binary formats. Any loss of precision is on the programmer.

If high-performance number crunching is ever required for compile-time metaprogramming (cryptography, AI, physics simulation) we'll rely on annotation-guided *Acceleration* (discussed later).

### Lists and Binaries

We use lists for basic sequential structures at all scales. Annotations may guide compile-time representations. To support most use cases (indexing, concatenation, slicing, deques), large lists are often represented under the hood as finger-tree ropes. Lists may also be lazy streams. 

Binaries are simply lists of integers in range 0 to 255. However, they are recognized and heavily optimized, supporting an efficient 'is binary' test and compact encoding in memory. Ultimately, binaries are the only input and output for an assembly.

### Symbols

Symbols are abstract data that support exactly one observation: an equality check. Symbols may be constructed by two means:

- declare unique symbols at the module toplevel
- wrap any valid dict key as an abstract symbol

Although pure functions cannot generate unique symbols, we can borrow from the module system. Declared symbols are unique *per import*, i.e. the same file imported twice will declare distinct symbols. This can be useful, but usually you'll often want to share through the environment.

### Dictionaries and Tagged Unions

Dictionaries represent finite key-value associations. Keys are compositions of basic data - numbers, lists, symbols, dicts - forbidding incomparable data types such as functions. Tagged unions are simply modeled as singleton dictionaries. Values may have any type. 

Dictionaries do not support iteration over symbolic keys. It is possible to implement iteration over the subset of keys that do not contain symbols, with a reproducible key order:

- numbers before lists before dicts
- numbers in rational less-than order
- lists in lexicographical order
- dicts as lists of key-value pairs

But for iteration in general, consider defining a 'keys' method or field that provides a list of relevant keys.

### Data Abstraction

Annotations can express sealing and unsealing of data with a key, such that it's an error to observe sealed data. Tagged unions serve a similar role (due to dictionaries blocking iteration): the value cannot be accessed without the key. Users should choose between annotations and tagged unions based on whether they want to support pattern matching and equality checks. 

Either way, robust access control involves a unique symbol in the key.

### Pattern Matching

Basic data is implicitly tagged, thus we can distinguish numbers, lists, symbols, dicts, objects, functions, etc.. for pattern-matching purposes. Users can effectively extend pattern matching via tagged unions.

*Note:* Ideally, we'll also like to support something analogous to F#'s active patterns or Haskell's view patterns.

## Objects

A stateless object model supporting mixin composition: `Dict -> Dict -> Dict` in roles `Base -> Self -> Inst`. Here, 'Self' supports open recursion via deferred fixpoint. 'Base' represents the mixin target or host environment.

        mix child parent = λbase. λself.
            (child (parent base self) self)
        new spec env = fix (spec env)
        (fix is lazy fixpoint)

Inheritance and override are useful tools for extensibility. We'll leverage objects for extensibility at multiple scales: functions up to multi-component assemblies.

*Note:* We can support 'ifdef'-like conditional behavior referring to names in 'Base'. 

### Function Objects

Instead of applying curried arguments of form `X -> Y -> Z -> Result`, it's often better to model a function as an object where the caller overrides 'x', 'y', and 'z', then reads 'result'. The latter supports keyword arguments with flexible ordering, defaults, and flexible extensions. For example, we can introduce arg 'w' with a default. 

The 'Base' argument to the function object may link to the caller's environment. This provides a basis for context-dependent behavior like algebraic effects. The caller can control this environment per call site.

I intend to make some variation of function objects the default, likely coupled with monadic effects. 

*Note:* In theory, we can also extend with 'result2'. But this doesn't generalize nicely to monadic result, nor to use of functions in an expression syntax.

### Multiple Inheritance and Specifications

Multiple inheritance (MI) supports merging of features that build upon a shared foundation or framework. This is modeled as a linearization algorithm, such as C3. Linearization eliminates redundant mixins and ensures shared mixins are applied in a consistent order across all components. When there are no shared elements, this reduces simple mixin composition. 

Linearization assumes ability to distinguish shared components. Although we cannot compare functions, it is feasible to use symbols as a proxy. I propose to support specifications, or specs, that are allocated a unique symbol when declared in the module toplevel.

*Note:* A 'spec' is essentially what most OO languages call a 'class'. But I feel the connotations are better, especially with focus being implementation inheritance and not subtyping.

### Symbolic Names

In context of specification mixins or multiple inheritance, using strings as names leads to ambiguities and collisions. The big culprits are generic, contextual names like "map" or "draw". There are some other weaknesses, too: no feedback upon deprecating a name, no convenient approach to 'private' names local to a mixin.

A robust solution is to declare unique symbols for each meaning or interface. Access to a symbol is routed through the module system, subject to qualification or aliasing. If we delete or deprecate a symbol the errors are easily detected and reported. Private names are easily modeled via symbols not exported.

A relevant concern with symbolic names is risk of namespace clutter. I hope to mitigate this syntactically, e.g. with 'using' directives. Also, we might stick to strings for keyword arguments to function objects.

### Checking Assumptions

In context of defining a mixin, most observations on 'Base' or 'Self' either compromise extensibility or diverge on fixpoint. A useful exception is observing which names are defined in 'Base': we can support 'ifdef' or check assumptions about whether a mixin introduces or overrides a name.

We still have several options: Holistic observations after a mixin or specification is defined. Define test methods for most things. Assert within method definitions.

## Effects

It is not difficult to provide a syntactic sugar for a lightweight effects system, perhaps coupled with function objects. As for modeling effects, I believe [Oleg Kiselyov's Free-er Monads](https://okmij.org/ftp/Haskell/extensible/more.pdf) is a suitable foundation and works well with extensible tagged unions.

Effects are useful for threading state between operations, modeling backtracking parsers, error handling, writing large binaries in small steps. Even multi-threading can be modeled, breaking a large batch task into small interactive tasks. 

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








