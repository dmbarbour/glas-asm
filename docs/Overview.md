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

* *Parallelism*: An annotation indicates that a lazy thunk will be needed later. The assembler adds the thunk to a queue to be processed by a background worker thread. Depending on configuration, this can be extended to remote workers. (This pattern is called 'sparks' in Haskell.)

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

Even without a runtime, effects are convenient for implicit dataflow, backtracking and error handling, flexible composition, extensible behavior, etc.. I'll use Haskell(-ish) syntax to describe these, but it should be easy to translate.

        type Eff rq a =
            | Yield (rq x) (x -> Eff rq a)
            | Return a

We can model a free-er effects monads as either *yielding* a `(request, continuation)` pair or *returning* a final answer. In case of yield, the continuation expects the response type from the request. But in context of untyped lambda calculus, we won't be rigorously enforcing these type annotations, just borrowing the structure. 

We can easily introduce some syntactic sugar:

        (sugar)         (monadic ops)
        a <- op1        op1 >>= \ a ->
        op2             op2 >>
        op3 a           op3 a

Haskell has a *RecursiveDo* extension, enabling the user to capture outputs from a monad's future as inputs to prior operations. In context of assembly, we'll likely want this feature by default. For example, with assembly it's convenient if we can reference forward labels that haven't been defined yet. To keep it simple, we'll generally forbid name shadowing.

We can specialize the monadic operators for our only monad. Our untyped lambda calculus doesn't offer a direct solution to type-indexed behavior, such as typeclasses, so this is convenient:

        (Yield rq k1) >>= k2 = (Yield rq (k1 >>> k2))
        (Return a) >>= k = k a
        k1 >>> k2 = (>>= k2) . k1

Effectively, '>>=' captures the continuation into 'Yield'. Unfortunately, it's left-associative, i.e. `((((k1 >>> k2) >>> k3) >>> k4) >>> k5)`. If implemented directly, this will repeatedly rebuild four compositions every time 'k1' yields. Right-associative `(k1 >>> (k2 >>> (k3 >>> (k4 >>> k5))))` performance is vastly superior. To resolve this, it is feasible to explicitly model a queue of continuations, or to *accelerate* '.' (function composition). I favor acceleration. 

Behavior is embodied in the runners (aka handlers). It is often convenient to express 'stacks' of partial handlers for local subtasks that forward unrecognized requests. Basically, any effect can be encoded except race conditions (outcome is deterministic). Examples:

        -- generalize Reader to pure queries; forwards unhandled queries
        runEnvT env (Yield (Env q) k) | Just r <- env q = runEnvT env (k r)
        runEnvT env (Yield rq k) = Yield rq (runEnvT . k)
        runEnvT _ r@(Return _) = r

        -- generalize State to indexed, scoped Memory
        --   memory is extensible via unique symbols
        runMemT m (Yield (Mem idx op) k) = match idx with
            Outer idx' -> Yield (Mem idx' op) (runMemT m . k)
            _ -> match op with
                Get -> runMemT m (k (m.[idx])) 
                Put v -> runMemT (m with { [idx]: v }) (k ())
                Del -> runMemT (m without idx) (k ())
        runMemT m (Return r) = (Return (r, m))

        -- cooperative threads (round robin, no preemption, no sync, hierarchical)
        runThreadT (Yield (Thread op) k):ts = match op with
            (Spawn t) -> runThreadT (k ()):t:ts
            Pause -> runThreadT (ts ++ [k()])
            (Outer rq) -> Yield (Thread rq) (runThreadT . (:ts) . k)
        runThreadT (Yield rq k):ts = Yield rq (runThreadT . (:ts) . k)
        runThreadT (Return ()):ts = runThreadT ts
        runThreadT [] = Return ()

        -- delimited continuations (hierarchical)
        runContT (Yield (Cont (Reset op)) k) = runContT op >>= runContT . k
        runContT (Yield (Cont (Shift fn)) k) = runContT (fn k)
        runContT (Yield (Cont (Outer rq)) k) = Yield (Cont rq) (runContT . k) 
        runContT (Yield rq k) = Yield rq (runContT . k)
        runContT r@(Return _) = r

        -- effectful choice, halts on return but captures search state
        --   initial bt: Return None
        runChoiceT bt (Yield (Choose x:xs) k) =
            let bt' = runChoiceT bt (Yield (Choose xs) k)
            runChoiceT bt' (k x)
        runChoiceT bt (Yield (Choose []) _) = bt
        runChoiceT bt (Yield rq k) = Yield rq (runChoiceT bt . k)
        runChoiceT bt (Return r) = Return (Just (r, bt))

We can also model runners that scope effects:

        -- forbid effects from escaping
        runPure (Return r) = r
        runPure (Yield _ _) = error "unhandled effect in runPure"
 
        runEnv env = runPure . runEnvT env
        runMem m = runPure . runMemT m
        runCont = runPure . runContT
        -- pure 'runThread' is useless

        -- pure choice can be heavily optimized with laziness
        runChoice (Yield (Choose xs) k) = List.flatMap (runChoice . k) xs
        runChoice (Yield _ _) = error "unhandled request in runChoice"
        runChoice (Return result) = List.singleton result

We'll want a library of useful, reusable handlers. However, I hope we deliberately design handlers with flexibility and extensibility in mind. Regular users should rarely feel the need to write custom handlers.

*Note:* Although we can leverage effects for general-purpose programming, doing so would dilute design. Eschewing the runtime eliminates or ameliorates many complicating factors for reasoning and performance. So, we'll focus exclusively on binary assembly.

### Commutative Effects

Monads easily overspecify order. Many effects can be at partially reordered without influencing outcome, yet with a significant impact on performance. To mitigate this, it can be useful to model asynchronous and threaded effects.

Asynchronous effects enable the runner to aggregate several requests from a single thread, enabling a reordering before we compute observations. Threaded effects enable the runner to examine pending requests from multiple threads before making any decisions.

        [(Yield BranchingQuery k1), (Yield TightConstraint k2), ...]
        -- constraint runner: let's apply TightConstraint next!

These techniques work very well together, e.g. we can model asynchronous interactions between threads. They also work nicely with spark-based parallelism: a runner heuristically sparks evaluation for a past request when the data becomes available, or a runner can batch-process several threaded effects then parallelize evaluation of each thread's next step.
 
## Modularity

A module is represented by a file. Modules may reference other files within the same folder or subfolders, or content-addressed remote folders. We forbid Parent-relative ("../") and absolute filepaths. These constraints ensure folders are location-independent and temporally stable, yet editable by conventional means.

Modules are modeled as basic objects with limited effects during construction, conceptually `Dict -> Dict -> {load, gensym} Dict` (in roles `Base -> Self -> Instance`), but this is abstracted by an effects API for *User-Defined Syntax*. There are two mechanisms to integrate more modules:

* *include* - bind included module's Base to host's 'current' Base; share Self. Effectively applies a module as a mixin, also treating prior definitions as mixins. Eager evaluation.
* *import* - binds imported module's Base to a host-provided environment (e.g. `{ "env": Self.env }` by default). Defines local name to instance dictionary. Lazy loading. To 'override' an import, simply replace the definition.

Dependencies between files must form a directed acyclic graph. However, each import or include is independent for gensym and Base, and it's awkward to maintain scattered references to content-addressed remotes. In most use cases, we'll share definitions through 'env.\*' instead of loading a module twice.

To ensure extensibility, there are no 'private' definitions or export controls at the module layer except via naming convention, such as Python's `"_name"`. The normal motives for private definitions are mitigated by content-addressing of remote dependencies.

### Configuration

The assembler implicitly loads a configuration module based on the `GLAM_CONF` environment variable or an OS-specific default, i.e. `"~/.config/glam/conf.g"` in Linux or `"%AppData%\glam\conf.g"` in Windows. A small, local user configuration typically extends a large, remote community or company configuration.

The configuration serves several roles:

- Initial environment: we pass `{ "env": Config.env }` to the assembly, as if importing the assembly into the configuration. This environment may be arbitrarily large by relying on lazy loading.
- Command-line macros: the configuration may define 'cli' as a function of type `List of String -> List of String`. This rewrite is applied to assembler arguments if and only if the first argument does not start with '-'. 
- Resource management: specify which GPGPUs are available for acceleration, where to cache things and how much to retain, integrate with shared proxy cache for compilation and testing, compute additional search locations for content-addressed remotes, tune assembler JIT or GC heuristics, disable expensive tests and checks. 

The configuration does not control assembly output. Relevantly, the assembly is free to reject your configured environment and substitute its own, even bootstrapping its own front-end compiler. Command-line macros influence outcomes, but users may bypass them. Resources may influence performance, error detection, debugging, but should not affect valid outcomes.

To support project-specific overrides or let users share a local system configuration, `GLAM_CONF` is not limited to one file. Users may list multiple files using the same OS-specific separator as the `PATH` variable. We apply these as mixins, each file overriding those listed later.

### Assembly

The assembler receives command-line arguments that express an assembly module as a list of mixins. The relevant arguments:

- `(-f|--file) FileName` - list a file to include; first file is included last, overriding those listed later. Depending on the configured environment, assembly files aren't limited to ".g" (see *User-Defined Syntax*).
- `(-s|--script).FileExt Text` - behaves as file in current working directory with the given file extension and text. As a special case, scripts may import parent-relative ("../") and absolute filepaths.
- `-- List Of Args ...` - the assembler defines 'args' before any includes, defaulting to an empty list. If a '--' separator is included, trailing arguments are forwarded to the assembly.

I imagine the common 'direct' usage is similar to `"glam -f File -- Args"`, while sophisticated use is left to command-line macros (see *Configuration*). Though, command-line macros might favor expressing behavior in the arguments then invoking a script, including modules based on the arguments.

The goal of an assembly module is (almost always) to specify a binary. 

We'll express this as defining 'result' to a binary value or an effectful expression that yields `(Write Binary)` a number of times before returning an exit code. For streaming binary output, the latter is more robust than laziness. (Laziness can be finicky.)

By default, we write the binary to standard output. However, there will be other command-line arguments to adjust this behavior, e.g. `"-o OutFile"`. The assembler can feasibly support projectional editing, debugging, and live coding, updating OutFile when there are no obvious errors.

The assembler does not have any built-in knowledge of CPU architectures and machine-code mnemonics. That's left to libraries. Consequently, our binary output is not limited to executable or linkable files. We could output ray-traced images, JSON configurations, a zipfile of webpages, etc.. 

### Remotes

A content-addressed remote folder is uniquely identified by secure hash of content or DVCS revision history. However, they are not *located* by secure hash. When importing or including a remote module file, the user may provide a list of URLs. The user configuration may suggest a few more.

For DVCS, we'll usually include a tag or branch name. This doesn't replace the hash. Instead, it is useful for shallow cloning, e.g. `"git clone -b Branch --single-branch URL"`, to control the amount of network traffic. We may need to roll back to the specified revision, but this provides a clear opportunity to detect and report available updates.

We'll start with support for 'git', but may add mercurial and darcs later. Aside from hash, URL, and tag, we'll must indicate which tool to use.

## User-Defined Syntax

User-defined syntax is a convenient approach to external DSLs and metaprogramming, and naturally extends to graphical programming. 

When loading a module, a front-end compiler is selected from the provided environment based on file extension, i.e. `Base.env.lang.[FileExt]` should evaluate to a language object that defines 'compile'. This is also the case for "g" files, but in that special case the assembler provides a built-in as a fallback. The only file that must be "g" is the user configuration.

A significant design challenge for user-defined syntax is integration with a development environment: tracing bugs, isolating syntax errors, visualization and editable projections, index and search, autocomplete, autoformat, etc.. The conventional solution is to develop a suite of external tools, IDE plugins, etc. for each syntax. The language object serves is intended to serve this role: aside from 'compile' it may define methods for tooling, e.g. a language-server protocol.

But the conventional solution is not very well integrated. It involves a lot of rework, and is fragile to changes in syntax. I hope we can do better by integrating features into the compiler.

Some design thoughts:

- Parser combinators are a great starting point. Parser combinators can implicitly track parse locations and describe what they 'expect' to see at any given step, providing effective feedback in case of syntax errors.
- To simplify tracing, blame, error isolation, etc. the compiler must avoid directly observing parse results. Even parse errors must be abstracted To enforce this, we can return abstract data from parse operations by default. This might be expressed as an applicative functor. We can also provide built-in combinators for common loop structures.
- To support syntax-driven effects without observing the syntax, we can introduce an eval effect that lifts a user expression into the front-end compiler. The result of eval is then abstracted. This effectively supports some forms of macros.
- It is useful to support multi-pass parses. For example, a first pass might delimit the 'region' for another pass. Even if we drop the first pass result, this can be very useful for error isolation. 
- Aside from source locations, the front-end compiler should be able to inject additional annotations on abstract nodes to support validation, visualization, editable projections, indexing, etc.. 
- Using gensym, we can generate a unique abstract data type for the applicative per file. This is useful to control staging, i.e. it is useless to hold onto the abstract data.

Expressing a compiler without being able to 'see' the binary seems possible in theory, but I'm not entirely convinced that we won't reintroduce the tracing problem via Eval. To gain confidence, I must try it first with the standard syntax. After all, we should be able to override and extend the standard syntax, too. Worst case, I'm back to the conventional approach.

*Note:* We'll convert FileExt ASCII characters to lower-case and strip an initial '.', but otherwise it's just taken as a string. 

### Syntactic Bootstraps

If the final `Self.env.lang.[FileExt].compile` method is different from the Base version, the assembler attempts bootstrap. This involves recompiling with the Self version, repeating until fixpoint is reached.

        bootstrap(ext, base, binary, compiler) =
            let m = compile(compiler, base, binary) 
            let compiler' = 
                m.env.lang.[ext].compile if defined
                or builtin if ext is "g"
            if(compiler == compiler') then m else
            bootstrap(ext, args, compiler')

The built-in compiler is simply treated as one more compiler in the bootstrap cycle, equivalent to itself. A module may simply delete a provided "g" implementation to avoid user-defined feature extensions. In practice, syntax bootstrapping should be relatively rare: it's expensive and easily goes wrong. 

But it has some use cases. Mostly, adding features to the primary ".g" language and shifting dependencies from assembler version to module system.

### Editable Projections

Some user-defined syntax may be graphical. And even for purely textual syntax, we'll often want to integrate some visualizations or provide edit widgets like color pickers. Miscellaneous observations:

- We can only edit terms annotated with source locations, parser context, and an encoder that converts the parse result back into source (aka lenses or prisms). 
- Some content is naturally read-only, e.g. content-addressed remote files, read-only local files. In these cases, we can still support navigation, views, etc.. and partially-editable views are feasible.
- Editable views benefit from some reflection, e.g. support indices and aggregated views across multiple files.
- It is convenient to treat a file that does not exist as equivalent to an empty file for purpose of imports. This way, an import reference obtains a hyperlink and projectional editor.
- I'll want to flexibly mix editable views with rendering test results, providing a round-trip between updates and outcomes. Interactions for filtering or progressive disclosure of tests should be possible.

TBD: This is non-trivial, and I don't have a solid handle on exactly how to approach it.

## Reasoning

What can we feasibly implement to support developers in reasoning about the assembly process and product?

* *Testing*: Sample system behavior under various conditions, and make pass/fail judgements. Should be able to visualize test results in tables and graphs. Should support fuzzing, heuristic exploration of condtions, paralleism, and incremental computing. Ideally supports blame, too.

* *Visualization*: Start with logging and profiling, but extend to graphical views, interactions for progressive disclosure or search, etc.. Perhaps we can model log messages as 'objects' implementing many views, e.g. both plain text and GUI widgets.

* *Tracing*: Maintain metadata to trace outcomes (errors, data, etc.) back to contributing sources. There's a tradeoff between precision and performance. We can feasibly leverage reproducibility by replaying a computation under a few different tracer setups to obtain more precision.

* *Types*: We can annotate our assumptions about data and programs in a machine-checkable way. The assembler makes a best effort to prove or disprove types, possibly using dynamic checks. Unfortunately, gradual typing won't be very effective for sophisticated uses (phantom types, substructural types, etc.).

* *Abstract Interpretation*: Given a representation of a program (e.g. machine code) we can implement an 'interpreter' using variables instead of data. Users add their own assumptions about this interpretation, then we check for conflicts. Essentially, we can mechanically implement a type system scoped to our target.

I envision use of type annotations to catch obvious errors in the assembly process, abstract interpretation as our primary means to reason about the product, and testing in a more ad hoc role. Visualization should build on tests and integrate nicely with a projectional editor.

### Integration

An relevant concern is how automated reasoning interacts with extension, especially breaking changes. Ideally, extensions can suppress expected errors and express updated assumptions. This suggests integration with the namespace, such that we can override the troublesome elements.

We can model module-level type declarations and tests as naming conventions, perhaps `"type:foo"` and `"test:xyzzy"`. The assembler may recognize these conventions then automate testing and type checking when the module is loaded. 

For anonymous assertions, logging, embedded type annotations, etc. we might reference an abstract, static 'channel' declared at the module level. Users may override the channel to disable or reconfigure. It should be feasible to route channels from the user configuration, enabling effective user control over assertions and such.

### Test Monad

Tests can be expressed monadically with simple, useful effects:

- Choice. Non-deterministic choice will support selecting test parameters, simulating race conditions, fuzz testing, etc.. It's also convenient for sharing test setup and parallel evaluation. Choosing from the empty list will cancel a test, neither pass nor fail; useful if our test conditions are out of bounds.
- Status. Essentially a test's 'public' state, expressed as a dictionary. At the end of a test, this should contain test parameters, relevant outcomes, perhaps a log message, i.e. anything we might want to cache or visualize. We can feasibly *animate* the evolution of status as a test runs (perhaps keep a 'frame' parameter in Status for this purpose). We can also use Status for Choice heuristics.

The pass/fail/etc. result is indicated by a final return value.

A test runner is relatively easy to implement, e.g. it's basically just a fusion of runChoice and runMemT to ensure we can view status before tests complete. But heuristics for effective fuzzing are not so easy to implement. And ideally, tests should also support incremental computing.

### Constraints

Constraint systems are a convenient mechanism for abstract interpretation. We can feasibly accelerate constraint systems with Z3 or other solvers. Unfortunately, the accelerator cannot observe the *solution* because it's effectively non-deterministic. But we can use the sat/unsat judgement.

The DSL for constraints can be directly adapted from SMTLIB2. For variables, we can easily use `(Var symbol)` or similar, using a distinct symbol per variable. We easily can control naming conflicts between globally-unique symbols and constructed symbols with local naming strategies.

It is relatively convenient to build a constraint system statefully, within a monad, and perhaps occasionally 'Check' for sat/unsat. This is the approach I intend to pursue for assembly code.

### Messages

As described earlier, log messages should generally be described as *objects* that define methods such as 'text' returning the basic text view, 'icon' for a small representative image, or 'http' to provide an entire browseable web-app to explain each log output. Using objects, we may support 'flavored' views via mixins, and bind Base to to some representation of the host environment. Reflection APIs, access to source locations, etc. might be provided via Base. 

Such messages benefit significantly from lazy evaluation. But they also assume the assembler doesn't immediately exit when the user is finished. We may need a command-line parameter for debug or IDE mode, where the assembler opens a GUI or local HTTP port.

## Assembly-Level Programming

TBD: So, we have tools to output a binary and reason about it. But how should we express machine code, concretely?


## Standard Syntax



# OLD CONTENT

## Syntax Ideas 

### User Data

I hope to develop a lightweight syntax for users to define data constructors, leveraging unique symbols for pattern matching and sealing. We can leverage smart constructors, active patterns, and type annotations.

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

