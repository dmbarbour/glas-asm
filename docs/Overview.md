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

I propose to build upon a pure, untyped lambda calculus with lazy evaluation, extended with annotations and module-level gensym. I do not believe untyped lambda calculus or lazy evaluation require introduction. We model data, effects, and OO-style inheritance upon this foundation:

- Data is logically [Church or Scott encoded](https://en.wikipedia.org/wiki/Church_encoding). 
- Monadic effects, via [Free-er Monads, More Extensible Effects](https://okmij.org/ftp/Haskell/extensible/more.pdf).
- Inheritance, adapting [Prototypes: Object-Orientation, Functionally](http://fare.tunes.org/files/cs/poof.pdf).

Users never touch the raw lambdas. Instead, we Scott encode a tagged union to distinguish numbers, lists, symbols, dictionaries, functions, objects, etc.. This supports ad hoc polymorphism similar to dynamic types.

The extensions are more structural than semantic:

Annotations are essentially structured comments to support tooling. Use cases include logging, profiling, visualization, type checking, testing, tracing, acceleration, and memoization. But annotations must not be observable *within* the assembly.

Symbols are abstract data with equality checks. Unique symbols are useful for data abstraction, access control, and conflict avoidance. However, pure functions cannot locally construct globally-unique values. To mitigate this, the module system serves as a source of uniqueness: during import, modules may generate unique symbols.

## Performance

Performance of lambda calculus is mediocre by default, and bignum arithmetic certainly won't help. But performance can be significantly enhanced with guidance from annotations. This becomes relevant in context of intensive testing or metaprogramming. The most relevant patterns:

* *Acceleration*: An annotation requests that a user-defined function is substituted by a specified built-in. The assembler performs ad hoc verification then replaces the function or emits a warning. Although we can accelerate individual functions such as matrix multiplication, the more general solution is to develop memory-safe DSLs that easily compile to CPU or GPGPU, then accelerate interpreters for those DSLs.

* *Parallelism*: An annotation indicates that a lazy thunk will be needed later. The assembler adds the thunk to a queue to be processed by a background worker thread. Depending on configuration, this can be extended to remote workers.

* *Caching*: An annotation suggests specific functions should be memoized to avoid rework. The annotation may provide further guidance on mechanism: persistent or ephemeral, cache size and replacement policy, coarse-grained lookup table versus fine-grained decision-tree traces. Depending on configuration, and leveraging PKI signatures and certificates, it is feasible to share remote cache with a trusted community.

Aside from these patterns, it is feasible to support annotation-guided just-in-time compilation of functions. Although, assembler functions are only used at assembly-time, this could offer significant performance benefits for testing. But probably not worth pursuing before bootstrapping the assembler.

## Data

The basic data types are numbers, lists, symbols, and dicts. Data is immutable, i.e. to 'update' a dictionary returns a new dictionary with the update applied. This can be efficient due to structure sharing and clever encodings under the hood.

- Numbers include bignum integers and rationals, without implicit overflows or loss of precision. Exact arithmetic becomes intractable within a loop, and users may need to round numbers. (For high-performance number crunching, we'll rely on *acceleration*.)
- Lists are used for all sequential structures. Large lists are represented by finger-tree ropes under the hood to efficiently support most operations. Binaries are lists of small integers (0..255) and optimized very heavily.
- Symbols are abstract data that support only equality comparisons. Symbols can be constructed in two ways: modules may declare guaranteed-unique symbols when imported, and any composition of basic data may be abstracted as a symbol.
- Dictionaries are finite key-value lookups where the keys are transitively composed of basic data. Singleton dictionaries are useful for modeling tagged unions. Dictionaries do not support iteration over symbolic keys, providing a simple basis for data abstraction.

User-defined data types will mostly be modeled as tagged unions with declared unique symbols. By hiding the symbol, this effectively serves an abstract data type, enabling the module to control construction and observation.

## Objects

Pure functions can model stateless objects in terms of open recursion via latent fixpoint. A basic object model with mixin composition is `Dict -> Dict -> Dict` in roles `Base -> Self -> Instance`. Here, 'Base' represents the mixin target or host environment, and 'Self' the future fixpoint.

        mix child parent = λbase. λself.
            (child (parent base self) self)
        new spec env = fix (spec env)
        # fix is lazy fixpoint

Most observations on Base or Self prior to instantiation will either diverge on fixpoint or compromise extensibility. Although fixpoint divergence is easy to detect and debug, an opportunity cost to extensibility is invisible and awkward to explain. It's best to provide a syntax that avoids potential pitfalls.

Inheritance and override is a useful mechanism for extensibility. For example, we can model a grammar as an object where the methods represent parser combinators, then extend the integer parser. In context of binary specification, available overrides will likely be less structured, more organic, but still useful.

Note that implementation inheritance is the focus, not subtyping or substitutability. Extensibility is producing useful variations of a system without invasive edits. But useful variation doesn't imply monotonic updates. For example, it might be useful to disable a parse rule to restrict a language.

*Aside:* I'll call 'specification' or 'spec' what most OO languages call 'class'. I feel specification has cleaner connotations, notably avoiding connotations of subtyping.

### Explicit Override

To resist accidents, it's useful to syntactically distinguish between introducing a name and overriding a name. Doesn't need to be much, e.g. `=` vs. `:=` is probably sufficient. Will figure this out when I start detailing syntax.

### Multiple Inheritance

Multiple inheritance is convenient when composing systems that build upon common foundations or frameworks. Given `C:A,B` and `A:F`, `B:F`, where F is the shared framework and C is the composition, we want a final mixin order `C,A,B,F`. Relevantly, F is not duplicated and appears *after* both A and B, ensuring consistent order, though A's view of F is now influenced by B. 

Multiple inheritance is implemented by reifying dependencies then applying a linearization algorithm such as C3. Of course, `Dict -> Dict -> Dict` functions are incomparable. To support linearization, we pair these functions with tags as proxy identifiers. 

By default, we'll implicitly declare a globally unique symbol to tag every syntactic object. This probably covers 99.9% of use cases. We can let users manually provide the tag to cover rare exceptions such as metaprogramming of objects.

Tags don't need to be globally unique but must not be reused in scope of linearization. To express and verify this local-uniqueness assumptions, we can introduce annotations for asserting incomparable values are equivalent. Although the assembler cannot prove equivalence of functions in general, it can easily warn when not obviously equivalent (i.e. referentially or structurally), or raise an error if obviously not equivalent.

### Symbolic Names

In context of mixins and multiple inheritance, using strings as names, especially generic and contextual names like "map" or "draw", easily leads to ambiguities and collisions. There are also other weaknesses: no feedback on deprecating a name, no clean approach to private names.

A more robust solution is to declare unique symbols for distinct meanings. Access to the symbol is routed through the module system, subject to qualification or aliasing. If we delete or deprecate the symbol, we can easily chase errors. Private names are easily modeled by controlling exports.

OTOH, a relevant concern with symbolic names is risk of namespace clutter. Also, we cannot use unique symbols for names at the module scope. Making symbolic names pleasant to use will inevitably require careful attention to syntax.

### Testing

We might express assumptions about objects as a collection of named assertions. These assertions may be disabled or overridden. We could automatically assert the tests when instantiating the object.

## Effects

Even without a runtime, effects are convenient for implicit dataflow, backtracking and error handling, flexible composition, extensible behavior, etc.. 

        type Eff rq a =
            | Yield (rq x) (x -> Eff rq a)
            | Return a

We can model a free-er effects monads as either *yielding* a `(request, continuation)` pair or *returning* a final answer. In case of yield, the continuation expects the response type from the request. But in context of untyped lambda calculus, we won't be rigorously enforcing these type annotations, just borrowing the structure. 

We can easily introduce some syntactic sugar:

        (sugar)         (monadic ops)
        a <- op1        op1 >>= \ a ->
        op2             op2 >>
        op3 a           op3 a

Haskell has a *RecursiveDo* extension, enabling the user to capture outputs from a monad's future as inputs to prior operations. In context of assembly, we'll probably want this feature by default to support jumping or branching to forward labels. I don't anticipate any trouble encoding this, but actually using it requires caution to avoid divergence.

We can specialize the monadic operators for our only monad. Our untyped lambda calculus doesn't offer a direct solution to type-indexed behavior, such as typeclasses, so this is convenient:

        (Yield rq k1) >>= k2 = (Yield rq (k1 >>> k2))
        (Return a) >>= k = k a
        k1 >>> k2 = (>>= k2) . k1

Effectively, '>>=' captures the continuation into 'Yield'. Unfortunately, it's left-associative, i.e. `((((k1 >>> k2) >>> k3) >>> k4) >>> k5)`. If implemented directly, this will repeatedly rebuild four compositions every time 'k1' yields. Right-associative `(k1 >>> (k2 >>> (k3 >>> (k4 >>> k5))))` performance is vastly superior. To resolve this, it is feasible to explicitly model a queue of continuations, or to *accelerate* '>>>' (or '.' in general). In context, I favor acceleration. 

Effects logic is embodied in the runner or handler. In many cases, we can usefully build 'stacks' of handlers, passing unhandled requests down the stack.

        runReaderT env (Yield Ask k) = runReaderT env (k env)
        runReaderT env (Yield rq k) = Yield rq (runReaderT env . k)
        runReaderT env (Return result) = Return result

        runStateT st (Yield Get k) = runStateT st (k st)
        runStateT _ (Yield (Put st') k) = runStateT st' (k ())
        runStateT st (Yield rq k) = Yield rq (runStateT st . k)
        runStateT st (Return result) = Return (result, st)

        -- threads or coroutines; round-robin, no preemption
        runThreadT (Yield (Spawn t) k):ks = runThreadT (k ()):t:ts
        runThreadT (Yield Pause k):ts = runThreadT (ts ++ [k ()])
        runThreadT (Yield rq k):ts = Yield rq (runThreadT . (:ts) . k)
        runThreadT (Return _):ts = runThreadT ts -- drop results
        runThreadT [] = Return ()

We can also use runners that scope effects. 

        runPure (Return result) = result
        runPure (Yield _ _) = error "unhandled request in pure scope"

        runReader env = runPure . runReaderT env
        runState s = runPure . runStateT s
        -- pure 'runThread' is useless

        -- runChoiceT omitted for dubious semantics
        runChoice (Yield (Choose xs) k) = List.flatMap (runChoice . k) xs
        runChoice (Yield _ _) = error "cannot backtrack effects!"
        runChoice (Return result) = List.singleton result

        runScopeT (Yield (Outer rq) k) = (Yield rq (runScopeT . k))
        runScopeT (Yield _ _) = error "unhandled request in scope"
        runScopeT r@(Return _) = r

We'll need to develop a library of useful, reusable handlers. Maybe a little metaprogramming. But it probably isn't something that everyone needs to mess with.

*Note:* Although we can leverage effects for general-purpose programming, doing so dilutes design. Many of my design decisions about reasoning and performance are dubious in context of runtime effects. This project shall focus exclusively on assembly-time programming.

### Commutative Effects

Monads sequence effects. Unfortunately, monads *overspecify* order in many cases. Many effects can be partially reordered without influencing outcome, yet with a significant impact on performance. Unfortunately, with free-er monads and separate runners, it's infeasible to optimize implicitly. What we can do is explicitly 'fork' subtasks then heuristically merge effects based on pending requests.

        [(Yield BranchingQuery k1), (Yield TightConstraint k2), ...]
        -- constraint runner: let's add TightConstraint before branching!

Essentially, we can context switch heuristically based on pending requests. We don't need fully commutative effects: a partial order is sufficient. In the stack transform context, to preserve structure, we might distinguish a 'main' thread that is allowed to pass requests up the stack.

## Modularity

A module is represented by a file. Modules may reference other files within the same folder or subfolders, or content-addressed remote folders. We forbid Parent-relative ("../") and absolute filepaths. These constraints ensure folders are location-independent and temporally stable, yet editable by conventional means.

A content-addressed remote folder is uniquely identified and authenticated by secure hash of content or DVCS revision history. However, they are not *located* by secure hash. Instead, a remote import may provide a list of locations (URLs), and configurations may suggest a few more. It is useful to include a DVCS tag or branch name for shallow cloning and to notify users of updates, but it cannot replace the secure hash.

Modules are modeled as basic objects with limited effects, roughly `Dict -> Dict ->  {load, gensym} Dict` (in roles `Base -> Self -> Instance`). We aren't bothering with multiple inheritance. The Base argument receives a host environment (dict 'env') for adaptability, while Self supports overrides for extensibility. Here 'load' brings another module into scope, and 'gensym' is our source of unique identifiers. We can enforce filepath constraints on load, and also raise an error for dependency cycles.

Aside from adaptability and extensibility, a motivating use case for parametric modules is to isolate remote references into very a few files, simplifying maintenance. Aside from one imports file per project, desired definitions should be available through 'env.\*'. We assume lazy loading and caching to support very large environments.

We provide two mechanisms to integrate modules:

* *include* - bind included module's Base to host's current Base and share Self. Essentially, included module applies as mixin. 
* *import* - bind imported module's Base to `{ "env": env }` in host, instantiate, then assign local name to returned dict.

These cover two distinct use cases: include for extension, import for lazy loading. Instead of extending an import, we override the dictionary, perhaps even replace it by another import. Due to laziness, prior definitions are never loaded.

Regarding 'private' definitions, they aren't difficult to model via gensym, but they hurt extensibility at this layer. It seems wiser to use plain-text names and simple naming convention like Python's `_name` at the module layer. We'll still want explicit override to resist accidents, and we'll heavily use symbolic names within object specifications.

*Note:* Aliasing definitions from an imported dictionary is a separate declaration. This separation is slightly inconvenient, but the intention is clearer that we're binding to the dictionary (which is subject to override), and not to the import. 

### Configuration

The assembler implicitly loads a module based on the `GLAM_CONF` environment variable or an OS-specific default, i.e. `"~/.config/glam/conf.g"` in Linux or `"%AppData%\glam\conf.g"` in Windows. This module specifies a configuration. A small, local user configuration typically extends a large community or company configuration, imported from DVCS.

A configuration specifies an initial environment for the assembly module. We'll simply forward whatever the configuration defines as 'env', logically importing the assembly into the configuration. The assembly is free to ignore the user environment and substitute its own, but this provides an opportunity for adaptation or even script-like assemblies.

Excepting 'env', configuration shall not influence an assembly's binary output. It may influence logging, testing, caching, GPGPU resources for acceleration, distributed computing and work sharing, etc..

### Assembly

Any module that specifies a binary, or comes close enough for the command line, effectively serves as an assembly. 

A selected module is parameterized by the configured environment. Some assemblies may be script-like, mostly composing resources defined in this environment. Users may also extract binaries from the configured environment without reference to a separate assembly module. 

The assembler has no built-in knowledge of CPU architectures or assembly mnemonics. It is feasible to 'assemble' documents, ray-traced images, or JSON configuration files.

## Reasoning

What can we feasibly implement to support developers in reasoning about the assembly process and product?

* *Testing*: Modules or objects may define test methods that are implicitly evaluated upon instantiation. Instead of simple pass/fail, I propose to model tests monadically with non-deterministic choice. Use of non-deterministic choice simplifies work sharing, parallelization, fuzz testing, simulation of race conditions, etc.. Assuming sufficiently-advanced acceleration, we could emulate the machine code to test the product.

* *Visualization*: We can easily introduce annotations for logging and profiling to obtain feedback. Plain text is very limiting. The assembler can provide a local web server or GUI, and the 'log messages' might be expressed as renderable objects. Although immutable, interaction is still useful for progressive disclosure, filtered views, rotating graphs, queries, etc.. (Of course, messages should *also* support a plain text rendering.)

* *Reflection*: A peek under the hood might let us render computations in small steps to understand a problem. Access to the continuation might support testing and visualization with 'what if' scenarios. I hesitate to introduce reflection in general because it compromises type and abstraction-based reasoning. But we can safely provide a reflection API in context of assertion or logging annotations.

* *Tracing*: The assembler can maintain metadata to trace outcomes (errors, data, etc.) back to relevant sources - a reverse lookup index on the assembly process. Tracing will be heuristic and lossy for performance reasons, but we can use annotations to guide heuristics and potentially recompute for precision.

* *Types*: Developers may annotate terms or definitions with partial types descriptors (i.e. optional, gradual typing). The assembler makes a best effort to prove or disprove types, i.e. error if disproven, warning if neither proven nor disproven. In some cases, this best-effort may be 'dynamic', examining actual arguments and return values during assembly. Aside from annotations, we can model abstract, nominative data types by controlling export of unique symbols.

* *Abstract Interpretation*: Leveraging monadic effects, a library of assembly mnemonics could simultaneously 'write' machine code, maintain abstract machine states with Hoare logic, and propagate state changes through a constraint system. Users then express assumptions as additional constraints. Effectively, users are free to reinvent type systems within a limited scope.

Note how tests and types are bound to definitions. It would not be difficult to annotate types and tests deep within functions. However, doing so hurts extensibility because we cannot update deeply embedded annotations in the same way we update definitions. In practice, even types on individual definitions may prove troublesome.

I hope to leverage abstract interpretation as the main tool for sophisticated reasoning about assembly output. I find it easier to trust code that I control directly (as opposed to an assembler's "best effort" at types), and there is more opportunity extend or tune the reasoning. A constraint system can be very expressive, effective for phantom types, substructural types, dependent types. We can feasibly leverage explicit reasoning outside of annotations for program search.

### Expressing Constraints

To support abstract interpretation, we can introduce a constraint monad. Essentially, this is a more sophisticated choice monad that remembers prior choices, binding them to variables. 

This should support declaring variables like 'x'. then 'ask' about conditions like `(x < 5)`. If 'x' hasn't been constrained, this generates a true/false choice. If we're already on a choice path where we know `(x < 5)` (e.g. if we know `x = 3`), then ask returns true only. We might also 'insist' on conditions, equivalent to 'ask' followed by 'fail' on the false branch. Use of 'insist' pushes constraints.

This assumes a DSL for expressing questions that the constraint monad can interpret, ideally something that can target SMTLIB2 with a little translation. We can use an accelerator like Z3 to test the constraints. We cannot observe Z3's discovered solution (Z3 is version-specific and non-deterministic), but sat/unsat should be deterministic. We use this to prune branches.

Aside from analyzing for sat/unsat, it may prove useful to analyze 'blame', e.g. based on [Cornell's SHErrLoc project](https://www.cs.cornell.edu/projects/SHErrLoc/). This requires keeping some extra metadata in the constraint system. After we isolate blame to some fragment of the output, we can trace it further to assembly sources.

*Aside:* A constraint monad is an obvious candidate for *Commutative Effects* described earlier. The order in which constraints are pushed, branches are pruned, does not influence outcome, but can hugely impact performance.

## User-Defined Syntax

When loading a module, we may specify a front-end compiler. If no compiler is specified, one will be chosen based on file extension. In case of the ".g" file extension, the assembler provides a built-in compiler. Otherwise, we'll ask the provided environment, e.g. `env.lang.select(FileExt)` should return an object that implements a 'compile' method (details TBD).

A front-end compiler will be expressed as a monadic function: a parser combinator with access to gensym and loading more modules, that simultaneously assigns definitions. Several desiderata:

- parsing provides some feedback on what input is expected (keywords, name or number, etc.) for error reporting
- can easily split files or sections into subsections that are parsed independently (isolating errors per section)
- can output definitions from different parts independently (i.e. so we can work with modules that have syntax errors)
- supports implicit source mapping for tracing or debug views

Users may extract the binary to end-of-file, but they'd lose a lot of system integration.

User-defined syntax is a low priority for now (we'll just focus on the built-in syntax) but eventually it will support flexible external DSLs and high-level languages. 

*Note:* A file does not need to exist to be loaded. If it does not exist, it is treated as an empty file. This is potentially useful for projectional editing. However, any other error (permissions issues, unreachable remote, etc.) are reported.


# OLD CONTENT


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








