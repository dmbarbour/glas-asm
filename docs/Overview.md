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

- Data is *logically* Church or Scott encoded. Actual encoding is accelerated.
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
- Dictionaries are finite key-value lookups where the keys are transitively composed of basic data. Tagged unions are modeled as singleton dictionaries. Dictionaries do not support iteration over keys containing symbols, a simple basis for data abstraction.

User-defined data types will mostly be modeled as tagged unions with declared unique symbols. By hiding the symbol, this effectively serves an abstract data type, enabling the module to control construction and observation.

## Objects

Pure functions can model stateless objects in terms of open recursion via latent fixpoint. A basic object model with mixin composition is `Dict -> Dict -> Dict` in roles `Base -> Self -> Instance`. Here, 'Base' represents the mixin target or host environment, and 'Self' the future fixpoint.

        mix child parent = λbase. λself.
           (child (parent base self) self)
        fix f = let x = f x in x     -- lazy fixpoint
        new spec env = fix (spec env)

Most observations on Base or Self prior to instantiation will either diverge on fixpoint or compromise extensibility. Although fixpoint divergence is easy to detect and debug, the opportunity cost to extensibility is invisible and awkward to explain. It's best to ensure syntax avoids potential pitfalls.

Inheritance and override is a useful mechanism for extensibility. For example, we can model a grammar as an object where the methods represent parser combinators, then extend the integer parser. In context of binary specification, available overrides will likely be less structured, more organic, but still useful.

Note that implementation inheritance is the focus, not subtyping or substitutability. Extensibility is producing useful variations of a system without invasive edits. But useful variation doesn't imply monotonic updates. For example, it might be useful to disable a parse rule to restrict a language.

*Aside:* I'll call 'specification' or 'spec' what most OO languages call 'class'. I feel specification has cleaner connotations, notably avoiding connotations of subtyping. 

### Explicit Override

To resist accidents, it's useful to syntactically distinguish between introducing a name and overriding a name. Doesn't need to be much, e.g. `=` vs. `:=` is probably sufficient. Will figure this out when I start detailing syntax.

### Multiple Inheritance

Multiple inheritance is convenient when composing systems that build upon common foundations or frameworks. Given `C:A,B` and `A:F`, `B:F`, where F is the shared framework and C is the composition, we want a final mixin order `C,A,B,F`. Relevantly, F is not duplicated and appears *after* both A and B, ensuring consistent order, though A's view of F is now influenced by B. 

Multiple inheritance is implemented by reifying dependencies then applying a linearization algorithm such as C3. Of course, lambdas are incomparable. And we probably shouldn't compare objects based on shared behavior regardless - it's 'purpose' we want to avoid duplicating. To support linearization, we'll pair functions with tags as proxy to purpose. 

By default, we'll implicitly declare a globally unique symbol to tag every syntactically-defined object. This probably covers 99.9% of use cases. We can let users explicitly provide a symbolic tag to cover the exceptions.

Tags don't need to be globally unique but must not be reused in scope of linearization. To express and verify this local-uniqueness assumptions, we can introduce annotations for asserting incomparable values are equivalent. Although the assembler cannot prove equivalence of functions in general, it at least can warn when not obviously equivalent (i.e. referentially or structurally), or raise an error if obviously not equivalent. 

### Symbolic Method Names

Using short strings as method names, especially generic and context-dependent names like "map" or "draw", easily leads to ambiguities and collisions, especially in context of multiple inheritance or dynamic mixins. There are other weaknesses: no clear mechanism for private names, no feedback on deprecating names.

A robust alternative is symbolic names. Modules may generate unforgeable symbols upon import, then export them. By using symbols as method names, we eliminate ambiguity and we gain object-capability security for individual methods, a secure yet flexible basis for privacy.

At external tooling boundaries, e.g. log message objects or language objects for user-defined syntax, it is awkward to reference names through the module system. In these cases, we can still construct unambiguous symbols based on DNS, e.g `symbol("glam-lang.org/2026/log/text")`.

At the module layer, we're effectively stuck with short strings: they're necessary for concise syntax! But modules are constrained in other ways that mitigate concerns of collision: no multiple inheritance, static mixins only (via include), and *explicit override* still applies.

## Effects

Even without a runtime, effects are convenient for implicit dataflow, backtracking and error handling, flexible composition, extensible behavior, etc.. I'll use Haskell(-ish) syntax to describe these, but it should be easy to translate.

        type Eff rq a =
            | Yield (rq x) (x -> Eff rq a)
            | Return a

We can model a free-er effects monads as either *yielding* a `(request, continuation)` pair or *returning* a final answer. In case of yield, the continuation expects the response type from the request. Of course, in context of untyped lambda calculus and gradual type annotations, enforcement may be a bit shoddy.

We can easily introduce some syntactic sugar:

        (sugar)         (monadic ops)
        a <- op1        op1 >>= \ a ->
        op2             op2 >>
        op3 a           op3 a

*Note:* Haskell also has a *RecursiveDo* sugar, enabling a result to be used before it is defined. I'm less familiar with this desugaring, but I'll surely want RecursiveDo by default. It seems convenient for branching to forward labels, for example. This has consequences for syntax: no shadowing names, because it would be unclear whether you're referring forwards or backwards.

We can specialize the monadic operators for our only monad. Our untyped lambda calculus doesn't offer a direct solution to type-indexed behavior, such as typeclasses, so this is convenient:

        (Yield rq k1) >>= k2 = (Yield rq (k1 >>> k2))
        (Return a) >>= k = k a
        k1 >>> k2 = (>>= k2) . k1

Effectively, '>>=' captures the continuation into 'Yield'. Unfortunately, the Kleisli composition is left-associative, i.e. `((((k1 >>> k2) >>> k3) >>> k4) >>> k5)`. Right-associative `(k1 >>> (k2 >>> (k3 >>> (k4 >>> k5))))` performance is vastly superior. To resolve this, we could lift into semantics (change Yield continuation to a queue), or insist on a built-in optimization (similar reasoning as tail-call optimization). I favor the latter.

Behavior is embodied in the runners aka handlers. It is convenient to express 'stacks' of partial handlers for local subtasks that forward unrecognized requests. Basically, any effect can be encoded except race conditions (outcome is deterministic). I wrote a few examples to help myself grasp this: 

        -- generalize State to indexed, hierarchical Memory
        -- use guaranteed-unique symbols to avoid conflicts
        runMemT m (Yield (Mem idx op) k) = match idx with
            Outer idx' -> Yield (Mem idx' op) (runMemT m . k)
            _ -> match op with
                Get -> runMemT m (k (m.[idx])) 
                Put v -> runMemT (m with { .[idx] = v }) (k ())
                Del -> runMemT (m without idx) (k ())
        runMemT m (Yield rq k) = Yield rq (runMemT m . k)
        runMemT m (Return r) = Return (r, m)

        -- delimited continuations (hierarchical)
        runContT (Yield (Cont (Reset op)) k) = runContT op >>= runContT . k
        runContT (Yield (Cont (Shift fn)) k) = runContT (fn k)
        runContT (Yield (Cont (Outer rq)) k) = Yield (Cont rq) (runContT . k) 
        runContT (Yield rq k) = Yield rq (runContT . k)
        runContT r@(Return _) = r

        -- cooperative threads (round robin, non-preemptive, hierarchical)
        --  with per-thread continuations (needed for mutexes, semaphores)
        runThreadT (Yield (Thread op) k):ts = match op with
            (Spawn t) -> runThreadT (k ()):t:ts
            Pause -> runThreadT (ts ++ [k()])
            (CallCC fn) -> runThreadT (fn k):ts
            (Outer rq) -> Yield (Thread rq) (runThreadT . (:ts) . k)
        runThreadT (Yield rq k):ts = Yield rq (runThreadT . (:ts) . k)
        runThreadT (Return ()):ts = runThreadT ts
        runThreadT [] = Return ()

We can also model runners that scope effects. Though, what to do with unhandled requests isn't well defined.

        -- forbid effects from escaping
        runPure (Return r) = r
        runPure (Yield rq k) = error "unhandled request in runPure" 
 
        runMem m = runPure . runMemT m
        runCont = runPure . runContT
        -- pure 'runThread' is useless

        -- pure choice can be heavily optimized with laziness
        runChoice (Yield (Choice xs) k) = List.flatMap (runChoice . k) xs
        runChoice (Yield rq k) = error "unhandled request in runChoice"
        runChoice (Return result) = List.singleton result

We'll need a library of useful, reusable handlers. However, I hope we deliberately design handlers with flexibility and extensibility in mind! Regular users should rarely feel the need to write custom handlers. Any 'good' monadic API is essentially architecting a framework.

*Note:* It would not be difficult to leverage monadic effects for general-purpose programming, like Haskell. However, I fear dilution of design.

### Commutative Effects

Monads frequently overspecify order. Many effects can be at partially reordered without influencing outcome, yet with a significant impact on performance. To mitigate this, it is useful to model asynchronous and threaded effects.

Asynchronous effects simply buffer requests to perform later. They may return an abstract future in some cases. Regardless, the handler has an opportunity to reorder requests prior to implementing them.

Threaded effects involve running multiple monadic threads. Like runThreadT but without Pause. Instead, the handler actually examines available requests and makes heuristic decisions about which to handle next, or perhaps handling them all. 

        [(Yield BranchingQuery k1), (Yield TightConstraint k2), ...]
        -- constraint-choice runner: let's apply TightConstraint next!

These techniques work well together, e.g. we can buffer asynchronous requests locally per thread, then yield heuristically (to flush the buffer) or when an external response is needed. The main benefit of buffering would be doing more work per step, which is mostly useful in context of spark-based parallelism.

## Modularity

A module is represented by a file. Modules may reference other files within the same folder or subfolders, or a content-addressed remote. We forbid Parent-relative ("../") and absolute filepaths. These constraints ensure folders are location-independent and temporally stable, yet editable by conventional means. 

Modules are modeled as mixins objects with limited effects during construction, conceptually `Dict -> Dict -> Eff {load | gensym} Dict` (in roles `Base -> Self -> Instance`). The module type is fully abstracted (see *User-Defined Syntax*). But we'll discuss it in these terms here.

A few mechanisms to integrate loaded modules:

* *include* - bind included module's Base to host's current Base, treating prior definitions as mixins. Share Self. Effectively applies a module as a mixin. Adds arbitrary names to namespace.
  * *include at* - scoped includes; applies to dictionary defined wihin Base instead of directly to Base.
* *import as* - introduces a dictionary with `{ "env": Self.env }` (by default) then translate include to this dictionary. 

Scoping enables lazy loading. Import as vs. include at supports *Explicit Override*. Note that we do not instantiate anything: there is only one Self for an entire configuration or assembly. This maximizes extensibility, ensuring we can override names defined deeply within imported modules. 

Dependencies between files must form a directed acyclic graph. However, each import or include is independent for gensym and Base, and it's awkward to maintain scattered references to content-addressed remotes. In most use cases, we'll share definitions through 'env.\*' instead of loading a module twice.

*Note:* We also forbid files and subfolders whose names start with ".". These are reserving for external tooling.

### Configuration

The assembler implicitly loads a configuration module based on the `GLAM_CONF` environment variable or an OS-specific default, i.e. `"~/.config/glam/conf.g"` in Linux or `"%AppData%\glam\conf.g"` in Windows. A small, local user configuration typically extends a large, remote community or company configuration.

The configuration serves several roles:

- *Assembly environment*: define 'env'. This is passed to the assembly as if importing the assembly into the configuration, e.g. `{ "env": Config.env }`. This environment is analogous to system includes and shared libraries, supporting adaptive assembly. 
- *Command-line macros*: if the first command-line argument to the assembler does not start with '-', we apply 'cli', which should be a function of type `List of String -> List of String` that returns valid arguments or empty list.
- *Development environment*: Define a loop for '-i' interactive mode. Filter outputs to standard error if non-interactive.
- *Resource management*: ad hoc, e.g. specify GPGPUs available for acceleration, cache locations and replacement heuristcs, history management, shared proxy compilation and cache, search locations for content-addressed remotes, tune assembler JIT or GC heuristics, control expensive tests and checks (e.g. assertions, fuzzing).

Configurations never directly control assembler output: An assembly may ignore your configured environment and substitute its own. Command-line macros may always be written out long form. Resources influence performance and error detection but not a valid binary result.

To support project-specific overrides or sharing of a system configuration, `GLAM_CONF` is not limited to one file. Users may list multiple files (same OS-specific separator as the `PATH` variable). We apply these as mixins, each file overriding those listed later.

### Assembly

The assembler receives command-line arguments that express an assembly module as a list of mixins. Though, in practice, it's usually just one file or script. Relevant arguments:

- `(-f|--file) FileName` - list a file to include; first file is included last, overriding those listed later. Depending on the configured environment, assembly files aren't limited to ".g" (see *User-Defined Syntax*).
- `(-s|--script).FileExt Text` - behaves as a remote file with the given file extension and text. Scripts cannot import local files.
- `-- List Of Args` - assembler defines 'args' before including files or scripts. Default is empty list, but caller may override with elements following the '--' separator.

Aside from 'args', the assembly module is implicitly parameterized by the configured 'env'. The assembly module shall define 'result', representing the assembled product, i.e. a binary or folder.

### Computed Modules

In some cases, it is convenient to compute some text then interpret it as an include or import with a specified file extension. We already do this via '-s' when specifying an assembly. We can easily support scripts as a special case for import or include within a module, too.

### Remotes

The assembler will support content-addressed remote folders or files. The source is uniquely identified and authenticated by secure hash of content or DVCS revision. However, remotes are not *located* by secure hash. The developer may supply a list of locations to search. The user configuration may rewrite this list to suggest alternatives, adjust priorities, etc..

For DVCS, we also specify a tag or branch. Not to replace the hash, but to reduce network traffic, e.g. `"git clone -b Branch --single-branch URL"`. If the branch has updated, we can may force to an earlier revision but also warn that the remote has been updated so users may update.

I intend to start with support for 'git'. Perhaps add mercurial and darcs later. It is feasible to hash individual files and access them via HTTP (like the dhall configuration language), but my intuition is that DVCS offers the superior maintenance experience.

### Access Control

Idiomatically:

        import ... as libfoo
        env.foo = libfoo.api if defined, else libfoo
        bar = ... env.foo.xyzzy ...

Modules don't have *export control*, per se. To support intuitions for 'include', and to maximize opportunity for extension and override, there are no 'private' definitions. But we do control the 'env.\*' environment available to imports. As a simple idiom, modules may define a dictionary 'api.\*' as a public API, then distribute only public definitions where feasible.

*Note:* The assembler may know of this idiom and warn whenever it discovers a module that defines an unused, public 'api'.

## Assembler

Primary behavior and inputs are detailed in *Modularity*. Roughly:

- load a configuration (`GLAM_CONF`) 
- construct an assembly (`-f -s --`)
- evaluate then extract the 'result'

By default, we expect a binary result and extract to standard output. However, the assembler supports a few other filesystem-aligned options for extraction:

- *Expectation:* data type of result
  - `--binary` (default) - result is binary 
    - simply a list of integers in 0..255
  - `--folder` - result models folder as dict 
    - dict keys are file and folder names
    - dict values are binaries or folders
- *Extraction:* where to put result
  - `--stdout` (default) - write binary to standard output
    - incompatible with folders and interactive development
  - `--discard` - extract for testing; drop the data
    - still accessible in interactive mode, e.g. HTTP request
  - `(-o|--out) Destination` - output to named file or folder

Machine-code mnemonics are left to libraries and syntactic sugars. Assuming accelerators and user-defined syntax, we can adapt this 'binary' assembler to many targets: ray tracing, typesetting, websites, simulations, blueprints, etc.. 

### Interaction

Instead of repeatedly asking an assembler to evaluate a result, we can ask an assembler to repeatedly evaluate a result, i.e. external versus internal loops. The internal loop enables some optimizations, e.g. for incremental computing. But the main benefit is that the assembler process sticks around and is available for interrogation. Some useful possibilities: language server protocol, REPL, graphical debug views, progressive disclosure, editable projections, integrated development, etc..

The assembler shall support interactive mode via simple command-line switch:

- `--batch` (default) - evaluate and extract result once, return
- `(-i|--interactive)` - maintain result, configurable interface

To avoid cluttering command-line arguments, and to keep the executable small, the assembler asks the user configuration to define an interaction loop. The loop may observe environment variables, assembler capabilities, and assembly definitions. Thus, with a few conventions, we can specialize the loop to an assembly or task.

The assembler limits external effects:

- *Filesystem:* only '-f' and `GLAM_CONF` files, siblings, and subfolders are visible. No parent-relative ("../") or absolute filepaths. The assembler cannot update read-only files.
- *Network:* listen on configured ports for TCP or Unix Domain Sockets connections. Cannot initiate arbitrary connections. There may be a few specialized operations, e.g. for local DVCS folders or content-addressed remotes. 
- *TTY*: Standard input and output as implicit network connection. Standard error disabled. Supports REPL or TUI.
- *GUI*: No native GUI. Supports browser-based GUI and other 'remote' GUI protocols.

The interaction loop shall be expressed as an object that primarily defines a transactional 'step' method. This step runs repeatedly, subject to transaction-loop optimizations: optimistic concurrency on non-deterministic choice, incremental computing, await relevant update after abort. Updates to the user configuration may influence future steps. Effects are abstracted: effectful operations use constructors linked via object Base. 

Interactive mode runs until voluntarily halted (via effects API) or externally killed (by OS).

### Debugging

Developers support debugging via annotations, e.g. types and tests, logs and profiles, tracing and blame. Effective debugging of assembly will inevitably depend on acceleration. With acceleration of an abstract machine, it is feasible to test machine code by emulation. With an SMT solver, it is feasible to test code via abstract interpretation.

Although we do not require use of type annotations, we'll not discourage them. Enforcement is best effort, warning if an error is neither proven nor disproven. I hope to support a debuggable visualization of the typechecking process, and to support constraint-based heuristic analysis to isolate errors similar to [Cornell's SHErrLoc project](https://www.cs.cornell.edu/projects/SHErrLoc/).

Debugging is best performed in context of interactive development. Access to replay greatly simplifies some tracing and analysis methods. A GUI view simplifies attention, visualization, and comprehension. And because everything *except* non-deterministic choice in named tests is highly reproducible (unlike runtime errors) it is no problem to fire up the IDE for interactive debugging. 

In non-interactive mode, we'll print some messages to standard error in a relatively conventional manner. The configuration may provide some filters. Then we instead report a number of skipped messages by domain and severity. We can heuristically cache failed tests (name, checkpoint, sequence of choices) for replay in interactive mode.

*Note:* In interactive mode, the IDE may disable standard error to support TUI, and the IDE may do whatever it wants with messages.

### Live Programming

Another process may continuously "run" an assembly result, watching for changes and integrating them. In context of executable machine code, this requires non-trivial setup, or at least restrictions on the function expressed by the machine code.

Although the assembler does not implement live programming directly, it should at least ensure atomic updates. That is, instead of replacing files, it first writes a temporary file then uses 'rename' in Linux or 'ReplaceFile' in Windows. Readers in Windows should then open the file in FILE_SHARE_DELETE mode.

In case the remote process accesses results through a network protocol such as HTTP, we should also make it easy to access an effective summary of version info, e.g. an aggregate hash of the contributing local files. We might use the hash in an in ETAG.

*Note:* If we discover writing to disk is a latency bottleneck, we can consider extending extraction modes with shared-memory double buffering or similar features. But profile first!

### History

Deterministic functions, location-independent folders, and content-addressed remote modules all contribute to reproducibility. But we still cannot reproduce the output if we cannot reproduce the initial conditions. To improve practical reproducibility, the assembler should automatically maintain a sufficient history to reproduce prior assemblies, ideally with structure sharing and effective pruning.

In practice, this may require copying local files and maintaining a local DVCS repo to represent the history. Though, we'll also need a little attention on merging history for concurrent assembler processes. (I wonder if darcs would be a good fit here.)

## User-Defined Syntax

User-defined syntax is a convenient approach to external DSLs and metaprogramming, and provides an opportunity for graphical programming.

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
- Users should never directly see module Base or Self. We should restrict module-level names to strings. This simplifies translation for 'include at'. We can simultaneously restrict object-level names to symbols.

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

* *Testing*: Sample system behavior under various conditions, make pass/fail judgements. We should be able to visualize tests in tables and graphs, and the test process in many cases. Should support fuzzing, heuristic exploration of conditions, parallelism, and incremental computing.

* *Visualization*: Start with logging and profiling, but extend to graphical views, interactions for progressive disclosure or search, etc.. I propose to model log messages as mixins implementing multiple views. Plain text is one view, but we can feasibly render icons, interactive widgets, etc..

* *Tracing*: Maintain metadata to trace outcomes (errors, data, etc.) back to contributing sources. There's a tradeoff between precision and performance. We can feasibly leverage reproducibility by replaying a computation under a few different tracer setups to obtain more precision.

* *Types*: We can annotate our assumptions about data and programs in a machine-checkable way. The assembler makes a best effort to prove or disprove types, possibly using dynamic checks. Unfortunately, gradual typing won't be very effective for sophisticated uses (phantom types, substructural types, etc.).

* *Abstract Interpretation*: Given a representation of a program (e.g. machine code) we can implement an 'interpreter' using variables instead of data. Users add their own assumptions about this interpretation, then we check for conflicts. Essentially, we can mechanically implement a type system scoped to our target.

I envision use of type annotations to catch obvious errors in the assembly process, abstract interpretation as our primary means to reason about the product, and testing in a more ad hoc role. Visualization should build on tests and integrate nicely with a projectional editor.

### Integration

An relevant concern is how automated reasoning interacts with extension, especially extensions that represent breaking changes. 

Ideally, extensions have an opportunity to both suppress expected errors and 'fix' them by updating assumptions. Extensions are aligned with the namespace. This suggest it's best to align types, tests, visualizations, etc. with the namespace where feasible. The structure of this alignment is flexible, so long as it admits erasure and override.

There are use cases for embedded type annotations and assertions. These anonymous structures can be 'fixed' only by updating the host function. Arguably, that is exactly what should occur: when used correctly, an embedded annotation is like a fuse in a system that says 'break here if this assumption changes', and should be written to be as weak as possible.

Embedded annotations will inevitably be used incorrectly. But worst case isn't too bad, e.g. override a few definitions or fork a library to weaken unnecessarily tight constraints. Maybe communicate a little (issue reports, pull requests).

Overall conclusion here: no new constraints, but enable and encourage 'named' tests, types, visualizations, etc..

### Tests

Requirements for tests:

- non-deterministic choice to support fuzz testing, simulate race conditions, etc..
  - both discrete and continuous, e.g. integers and rationals
  - assembler may use abstract interpretation to guide choice 
- status for test parameters and outcomes, a record we can graph, visualize, animate
- log outputs, not just as annotations but as something we can place on test timeline
- return a pass/fail judgement

Monadic expression of tests is well-aligned to these goals. However, users cannot implement handlers for non-deterministic choice or abstract interpretations. Testing will require assembler support. The assembler may heuristically or configurably cache tests to reduce rework and support regression testion. This involves capturing a continuation and sequence of non-deterministic choices (or ranges of choices). For long-running tests, the assembler can also maintain intermediate checkpoints for efficient replay.

In interactive mode, it is feasible to continue fuzzing indefinitely. On updates, some old tests may become 'stale' based on incremental computing. It might be useful to visualize stale tests together with new ones, perhaps distinguishing by color. In non-interactive mode, we'll want configurable, heuristic quotas for how much testing to perform.

### Types

The underlying lambda calculus is untyped, but we can express type annotations. I imagine type annotations will be incomplete, partial, gradual. The assembler makes a 'best effort' to either verify or contradict type annotations ahead of evaluation. If types are neither proven nor disproven, the assembler emits a warning. 

There are no 'dynamic' type checks during evaluation. Although feasible, dynamic typing has unpredictable performance implications and hinders type-level debugging. That said, dynamic checks may be useful for heuristically tracing blame.

Under these constraints, what types can we express?

- structural data types: numbers, lists, symbols, dicts, tagged unions. We can refine common number types such as rational, integer, or natural. Dictionaries and tagged unions may be 'open' to extension with new cases.
- user-defined nominative types: via tagged unions with guaranteed-unique symbols. Ideally, we can express GADTs and dependent types, so we can express our assumptions even when we cannot fully check them.
- substructural types and session types are feasible within limited scopes. Modeling them in effects handlers is probably the most relevant, e.g. to model channels typed with protocols. 

We cannot directly express type-indexed behavior, i.e. no typeclasses, operator overloading, or multimethods. Annotations cannot influence outcomes. But it is feasible to ensure types are aligned with structure in some way to simulate type-indexed behavior.

*Note:* Type checking in context of gradual types remains vague to me. My intuition is that these should clear up a lot once we start defining a language of type descriptions.

### Constraints

Constraint systems are a convenient mechanism for abstract interpretation. We can feasibly accelerate constraint systems with cvc5, Z3, or other solvers. Unfortunately, the accelerator cannot observe the *solution* because it's effectively non-deterministic. But we can use the sat/unsat judgement.

The DSL for constraints can be directly adapted from SMTLIB2. For variables, we can easily use `(Var symbol)` or similar, using a distinct symbol per variable. We easily can control naming conflicts between globally-unique symbols and constructed symbols with local naming strategies.

It is relatively convenient to build a constraint system statefully, within a monad, and perhaps occasionally 'Check' for sat/unsat. This is the approach I intend to pursue for assembly code.

### Messages

I propose to model log messages as mixins. In the simplest case, a message defines 'text' for printing to a console. But we can feasibly extend the message with multiple views and metadata for search and filter. Mixins can easily express templated messages with good defaults. And the assembler may provide some reflection capabilities, access to locations, etc. as arguments to Base.

Although messages are stateless, it seems useful to model session-local (or user-local) state for stateful views, such as progressive disclosure.

## Assembly Programming

Instead of machine-code mnemonics being something an assembler *interprets*, we can model them as monadic operations that the assembler *executes*. Each mnemonic can:

- write some binary data to intermediate staging location
- maintain an abstract representation of program state
- declare ad hoc external structure (.text, .data, .rodata)
  - i.e. so we can deref 'structures' instead of 'pointers'

Ideally, there is no need to forward-declare labels for jumps, leveraging monadic fixpoint and staging to defer the pointer. Ideally, we can also support declarative allocations for static data, stack, and heap, i.e. where the amount allocated depends on future *assumptions* about which fields are present.

*Aside:* Monadic expression is generally more robust and compositional than macros, yet similarly expressive. We are able to define higher-level mneumonics that are inlined.

## Standard Syntax

This section proposes an initial syntax for ".g" files. It should look and feel like assembly proramming in many cases, and favor vertical structure over deep indentations. Associative structure needs special attention because it enables future operations to influence how much is allocated.




