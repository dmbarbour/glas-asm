# Project Overview

Core idea: Use a lambda calculus extended with reified environments to compose assembly-level language fragments at compile-time. Leverage FP and OO-inspired design patterns. Fully deterministic output.

Anticipated benefits: assembly programming that is far more extensible, composable, scalable, and securable than existing assembly-level programming systems, while still offering fine-grained control.

## Base Namespace Calculus

Start with an untyped lambda calculus:

- functions, `λx.Term`, essentially templated terms
- applications, `(Term Term)`, applies a function
- substitution, `x`, replaced by a term in scope

Introduce first-class environments:

- reification, capture visible definitions into dictionary
- expansion, load reified environment into scope (at prefix)
- translation, rewrite or hide names in scope (via prefix)

Prefix-oriented expansions and translations are designed to simplify lazy computing and efficient rewrites at some cost to expressiveness. But they're adequate for the purpose.

## Data Extensions

Logically, we're working with [Church or Scott encodings of data](https://en.wikipedia.org/wiki/Church_encoding). But, for reasons of performance and syntactic convenience, we'll abstract the encoding for some basic data types - numbers, lists, binaries, dicts - then integrate them as built-ins.

## Annotations

Annotations are structured comments that support instrumentation, verification, performance, and other tools. An important constraint is that annotations have no impact on the generated binary. However, they may halt compilation with errors or influence *auxilliary* outputs, such as:

- message logs, warnings
- compile-time profiling
- cache for incremental compilation
- source maps for runtime debugging

Annotations are always applied to a term. To resist silent degradation, we may report warnings for unrecognized annotations.

## Acceleration

With guidance from annotations, compilers may substitute user-defined functions with built-ins. This pattern is called *acceleration*. The aforementioned *Data Extensions* are essentially built-in accelerators.

It is feasible to accelerate an *interpreter* for a virtual CPU or GPGPU such that we can leverage actual hardware at compile-time. This may require careful design to ensure memory safety and deterministic output, but it provides a future direction in case we need high-performance computing at compile-time.

## Modularity

Modules do not directly represent assemblies. Modules contain definitions, a subset of which may define assemblies. 

A module is represented by a file and may refer to other files within the same folder or subfolders. Parent-relative ("../") and absolute filepaths are forbidden, thus folders serve as location-independent packages. Remote dependencies are supported via remote DVCS. 

Limited module-system virtualization is supported: we can specify functions to rewrite filenames and DVCS repos. Use cases include whole-system versioning and curation, e.g. redirect an unstable repo branch to a stable hash or privately controlled fork.

Not all files represent modules. Files may instead be loaded as binary data for embedding or metaprogramming.

*Note:* Aside from compiling assemblies, consider support for extracting computed binaries.

## Content-Addressed Locations

An assembly often needs a data pointer or a jump target. Instead of declaring these locations separately, we describe content and metadata (such as '.rodata' section). Actual locations and layout is generated later. If users must precisely control location and layout, it can be guided by metadata and user-defined layout functions.

To support mutable allocations, the metadata supports an extra slot for content-addressing purposes. For example, the '.bss' section may need dozens of 8-byte variables, all initialized to zero. To prevent conflicts, see *Extrinsic Identity* below.

Content addressing significantly simplifies metaprogramming. We can allocate global static state by referencing it, or install a macro-generated subroutine by calling it. And conversely, we can erase anything that isn't transitively referenced.

This does have a cost: users lose direct control over memory layout. But indirect control remains feasible: extend metadata with layout hints, let assemblies override an interface to guide layout.

## Extrinsic Identity

Robust identity is useful for allocating mutable memory in context of content-addressed locations.

The module system serves as a convenient source of identity. With a little support from the compiler, a module may allocate however many unique, abstract identifiers as it wants.

However, this is easiest for 'shallow' definitions, i.e. where we can simply generate identity based on location within a source file. In context of metaprogramming, users must manually thread identity or generate new identities via composition with data or other identities.

## Assembly

At some point, we'll be writing out a sequence of ordered, parameterized assembly mnemonics. In some cases, we'll refer to external, content-addressed definitions. In others, we might refer to local labels for branch or jump instruction. And we may freely 'inline' assembly code. 

Ideally, as we write things out, we're also accumulating details such as which content-addressed structures the assembly references, or local labels for use as branch or jump targets. We may need a fixpoint or a separate pass to fill in final details.

This basic structure suggests something like a monadic structure for operating on the sequence of heuristics. But it is feasible to simply build a list of operators for ad hoc processing.

### Local Labels

In context of jump or branch operations, we'll often reference a local label within the assembly code. The notion of content-addressed locations does not apply in this case. But we cannot assume the label target is defined before it is used, i.e. we can still jump to something later within the assembly program.

### Structured Stacks

Although we can directly manipulate the stack pointer, it would be convenient if we can automatically allocate stack locations based on usage, similar to how we allocate '.bss' sections. This seems feasible if we automatically accumulate sizes for referenced stack data elements.

## Object Model

A basic model for mixins is `Env -> Env -> Env` in the roles `Base -> Self -> Instance`. We can compose mixins, overriding methods accessed through 'Self'. Or we can instantiate objects, linking to a 'Base' (often empty) then closing the fixpoint 'Self'.

        mix c p = λb.λs.(c (p b s) s)
        new o b = fix (o b)

Effective use of mixins requires deliberate design, presenting some methods as tunable parameters in anticipation of overrides.

We can further wrap this model to support multiple inheritance (MI). I am assured MI has many use cases, though it isn't clear to me what they are. 

MI requires consistent linearization, and linearization requires comparing mixins for equivalence. Unfortunately, comparing lambdas is difficult. As a proxy, we could leverage extrinsic identity if we can guarantee it isn't reused.

The compiler will support MI for second-class 'specs' in the module toplevel. The details are abstracted, though we'll use the C3 linearization algorithm. Specs evaluate to basic mixins.

### Assembly Objects

We'll model assemblies as objects to support overrides and extensions, or at least provide the opportunity. The object may override auxilliary methods, e.g. to guide memory layout. The CLI could feasibly 'inject' some command-line parameters or environment variables as a final mixin.

### Parameter Objects

For "low-level" functions, we may favor curried arguments, i.e. `A -> B -> C`. But such structure is unfriendly to extension, deprecation, defaults, varargs, refactoring call prep, etc..

The preferred solution is to shove parameters into a mixin object. When receiving a call, we apply the mixin to a defaults object such that parameters *override* defaults. Conventions:

* 'args' - list of anonymous, positional parameters
* 'env' - implicit context; caller may restrict and redirect
* other - keyword arguments

The syntax may make parameter objects the default, though curried forms may be supported indirectly. 

### Tags and Adapters

We can model a tag as a namespace-layer function that, given an environment of adapters, picks one then applies it. For example, a "bar"-tagged Term, applied to `{ "foo":Op1, "bar":Op2 }`, returns `(Op2 Term)`.

I propose to basically tag everything. The compiler may introduce initial tags like "list", "dict", "int", "spec", "obj", "env". Users may introduce new tags as needed. Pervasive use of tags effectively serves as a lightweight, dynamic type system, but does not require any new semantics.

*Note:* Alternatively, we can model tags as singleton dictionaries. But I favor the design above.

## Sketch of Syntax





