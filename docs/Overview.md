# Project Overview

The goal for this project is to lift assembly-level programming into something I'll feel comfortable directly using as a primary language. A very high-level language for very low-level programming.

Like any low-level language, assembly benefits from effective composition and extension, a scalable module system, and expressive metaprogramming. As stretch goals, I'm also interested in automated verification or proof-carrying code.

Unlike most low-level languages, assembly is without optimizer or runtime except insofar as they are explicitly modeled by the programmer. The main operation on assembly code is to extract binaries - usually executable or linkable. Reproducibility and determinism are primary design concerns.

## Underlying Semantics

We build upon a pure, untyped lambda calculus with lazy evaluation and annotations. I don't believe lambda calculus or lazy evaluation need further introduction.

Annotations are structured comments for tooling. Annotations may help detect errors (e.g. assertions, types), generate auxilliary outputs (e.g. logs, cache), or influence compile-time performance (e.g. memoization or acceleration). However, annotations do not modify the represented binary.

We'll build data, objects, even constraint systems as design patterns upon the lambda calculus as a foundation, relying on annotations.

## Data

We'll support a few basic data types: numbers, lists, symbols, and dicts. 

Data is immutable, leveraging persistent data structure for efficient update. Logically, data is Church or Scott-encoded, but the details are abstracted. The assembler favors efficient representations under-the-hood.

### Numbers and Arithmetic

The assembly supports exact rational numbers and bignum arithmetic. We'll provide operators to explicitly convert to and from common binary formats. Any loss of precision is on the programmer.

If high-performance number crunching is ever required for compile-time metaprogramming (cryptography, AI, physics simulation) we'll rely on annotation-guided *Acceleration* (discussed later).

### Lists and Binaries

We use lists for basic sequential structures at all scales, including pairs. Annotations may guide compile-time representations. To support most use cases (indexing, concatenation, slicing, deques), large lists are represented under the hood as finger-tree ropes.

Binaries are simply lists of integers in range 0 to 255. However, they are recognized and heavily optimized, supporting an efficient 'is binary' test and compact encoding in memory. Ultimately, binaries are the only input and output for an assembly.

### Symbols

Symbols are abstract data that support exactly one observation: an equality check. Symbols may be constructed by two means:

- declare unique symbols at the module toplevel
- wrap any valid dict key as an abstract symbol

Although pure functions cannot generate unique symbols, we can abstractly borrow uniqueness from the module system. Some module syntax, e.g. defining a specification, implicitly introduces unique symbols.

### Dictionaries and Tagged Unions

Dictionaries represent finite key-value associations. Keys are compositions of basic data (numbers, lists, symbols, dicts), forbidding incomparable types such as functions. Values may have any type.

An unusual restriction is that dictionaries do not support iteration over keys. This is mostly because it is difficult to define a reproducible iteration order in context of declared symbols. However, we also leverage this property for *Data Abstraction*.

Tagged unions are modeled as singleton dictionaries, where the only key represents the tag. Typically, the tag is a declared symbol.

*Note:* The assembler implicitly tags all the things in another layer - numbers, lists, dicts, symbols, functions, specifications, etc.. 

### Data Abstraction

At the lower level, we introduce annotations to logically 'seal' and 'unseal' data with a key. Any operation on sealed data, except unsealing it with the correct key, raises an error. However, user-sealed data blocks use with pattern matching or dict keys. 

At the higher level, we favor tagged data for abstraction. The inability to iterate dict keys serves effectively as a seal. But we can also express pattern matching across several candidate tags, or use tagged data as an abstract dict key.

For robust abstraction, a module should declare a few symbols for use as tags or keys without exporting them. This enables the module to force construction and observation through a provided API.

## Objects

We'll use objects for at least two use cases:

- Parameter objects: We can express parameters as mixins that override defaults. We can refactor expression of parameter lists.
- implementation inheritance: We can express a large assembly in terms of ineriting and tweaking an existing assembly without pervasive edits.

### Base Object Model

An anonymous, stateless object model that supports mixin composition: `Dict -> Dict -> Dict` in roles `Base -> Self -> Instance`. Here, 'Self' represents an open fixpoint, supporting recursive definitions and overrides. 'Base' may represent a mixin target or host.

        mix child parent = λbase. λself.
            (child (parent base self) self)
        new spec base = fix (spec base)
        fix is lazy fixpoint

### Namespace

At least for the large scale, we'll favor symbols as names. Symbols can be uniquely bound to meanings, aliased for local imports. In contrast, string names like "map" or "draw" lead to conflicts when integrating contexts. We can also delete a symbol upon deprecation, resulting in clear errors. 

We'll distinguish intentions to override vs. introduce a name. This allows us to raise suitable errors if the name is or is not already present. We can also support an 'intro if not defined' case. 

### Parameter Objects

Instead of curried arguments `A -> B -> C -> ...`, we'll favor one big parameter object for most method calls. This is applied as a mixin to override default arguments, enhancing extensibility. There are also potential benefits for refactoring of repetitive argument structures.

The user syntax should implicitly build a parameter object based on parameter lists, keyword arguments, and a tacit environment. We can easily have two standard parameters then add symbols:

- 'args' - list of parameters
- 'env' - tacit environment
- symbolic keyword args

### Multiple Inheritance

Multiple inheritance (MI) is convenient when merging multiple systems that derive from a shared framework. We apply a linearization algorithm (e.g. [C4](http://fare.tunes.org/files/cs/poof/ltuo.html#(part._.C4))) to ensure each component is integrated, initialized, etc. only once, and order is consistent across all merged systems.

To support linearization, we can leverage unique symbols as a proxy for comparing mixin functions.

### Specifications



## Modularity





## System Verification


### Unique Associations

*Aside:* In context of metaprogramming, it is feasible to enforce unique associations via annotations.

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

## 

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








