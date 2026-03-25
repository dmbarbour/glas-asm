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

Users never touch the raw lambdas. Instead, we Scott encode a tagged union to distinguish numbers, lists, symbols, dictionaries, and functions. User-defined types are generally modeled as singleton dictionaries with symbolic keys. This supports ad hoc polymorphism similar to dynamic types.

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

The basic data types are numbers, lists, symbols, dicts, and functions. Data is immutable, i.e. to 'update' a dictionary returns a new dictionary with the update applied. This can be efficient due to structure sharing and clever encodings under the hood.

- Numbers include bignum integers and rationals, without implicit overflows or loss of precision. Exact arithmetic becomes intractable within a loop, and users may need to round numbers. (For high-performance number crunching, we'll rely on *acceleration*.)
- Lists are used for all sequential structures. Large lists are represented by finger-tree ropes under the hood to efficiently support most operations. Binaries are lists of small integers (0..255) and optimized very heavily.
- Symbols are abstract data that support only equality comparisons. Symbols can be constructed in two ways: modules may define guaranteed-unique symbols when imported, and composition of data *excluding* functions may be abstracted as a symbol.
- Dictionaries are finite key-value lookups where the keys are transitively composed of basic data. Tagged unions are modeled as singleton dictionaries. Dictionaries do not support iteration over keys containing symbols, a simple basis for data abstraction.
- Functions are, essentially, dynamically computed key-value lookups. Some computations may diverge. We cannot observe whether two functions are equal, but we can assert they are (via annotation).

User-defined data types will mostly be modeled as tagged unions with declared unique symbols. By hiding the symbol, this effectively serves an abstract data type, enabling the module to control construction and observation.

## Objects

Pure functions can model stateless objects in terms of open recursion via latent fixpoint. A basic object model with mixin composition is `Dict -> Dict -> Dict` in roles `Base -> Self -> Instance`. Here, 'Base' represents the mixin target or host environment, and 'Self' the future fixpoint.

        mix child parent = λbase. λself.
           (child (parent base self) self)
        fix f = let x = f x in x     -- lazy fixpoint
        new spec env = fix (spec env)

Most observations on Base or Self prior to instantiation either diverge on fixpoint or compromise extensibility. Although fixpoint divergence is easy to detect and debug, an opportunity cost to extensibility is invisible and awkward to explain. So, it's best to design a syntax for constructing objects that avoids the pitfalls.

A use case for objects is extensibility in context of mutually recursive definitions. For example, a stateless object may model a grammar. Methods, as dict values, may represent parser combinators for different cases (parsing integers, for example). The ability to update (extend, disable, etc.) a parse rule without rebuilding the entire grammar is convenient when developing variations on a language.

### Explicit Override

To resist accidents, it's useful to syntactically distinguish between introducing a name and overriding a name. Doesn't need to be much, e.g. `=` vs. `:=` is probably sufficient. Will figure this out when I start detailing syntax.

### Stateful Specification

Mixins can easily model state-like operations, e.g. increment a 'counter' in Base while capturing the prior value.

        λBase. λSelf. Base with { 
            myid = Base.idct, 
            idct = 1 + Base.idct 
        }

In this case, the final 'Self.idct' would be the number of identifiers allocated. This pattern easily extends to building tables and other structures. However, it does take some discipline to use correctly when expressed directly in lambda calculus. This could be mitigated by something like writing object definitions within a 'staged state' monad.

### Extended Objects

We'll use basic objects at the module scope, i.e. modules as mixins. But we'll probably want a few more features for modeling objects in the general use case. Potential extensions:

* *multiple inheritance* - Wrap basic mixins with tags, inheritance lists, and linearization algorithms. Linearization algorithm asserts that mixins with the same tag are equivalent, eliminates redundancy, and ensures a consistent merge order for shared mixins. 
  - A front-end compiler can assign a unique symbol as the default tag.
* *lazy instantiation* - Because objects are stateless, there is no need to instantiate more than once. Alongside those inheritance lists, we may include a lazy instantiation of the object's definition.
* *access control and interfaces* - Symbolic keys cannot be iterated. Thus, we can use symbolic keys to control access and override for subsets of methods. It is syntactically awkward to use symbols per method, but per declared interface seems feasible.

Ideally, we can provide a syntactic sugar that makes extended objects convenient to work with, but enabling users to explore alternative object models and further extensions.

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

Effectively, '>>=' captures the continuation into 'Yield'. Unfortunately, the Kleisli composition `>>>` is left-associative, i.e. `((((k1 >>> k2) >>> k3) >>> k4) >>> k5)`. Right-associative `(k1 >>> (k2 >>> (k3 >>> (k4 >>> k5))))` performance is vastly superior. To resolve this, we could lift into semantics (change Yield continuation to a queue), or insist on an accelerator or built-in optimization (similar to tail-call optimization). I favor the latter.

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

## Modules

A module is *represented by* a file, and *represents* a basic mixin object.

To simplify architecture, file dependencies are constrained: a file may reference only local files within the same folder or subfolders (no parent-relative ("../") or absolute paths), or content-addressed remote files (by DVCS revision hash and filename). It's an error to load files or subfolders twice. To simplify tooling, local files and subfolders whose names start with "." are hidden from the module system.

The assembler provides a built-in front-end compiler for ".g" files. However, the module system supports *User-Defined Syntax* aligned to file extensions, detailed later. The front-end compiler constructs a basic mixin object, i.e. `Dict -> Dict -> Dict` in roles `Base -> Self -> Instance` without multiple inheritance (see *Objects*). Front-end compilers logically support unique symbol generation via *Stateful Specification* of objects.

Every module-level definition is modeled as a mixin. A module is integrated by 'including' it as another mixin. But there are a few forms:

* *include Module* - bind included module's Base to host's current Base namespace, sharing Self.
* *include Module at m* - translate inclusion to a component dictionary 'm', i.e. included module Base links to host's Base.m, and the included module's Self is linked to the host's Self.m. 
* *import Module as m* - useful sugar: introduces 'm' with `{ "env": Self.env }` (by default), then applies 'include Module at m'.

The 'include at' and 'import as' forms are useful for lazy loading and access control. In the end, there is only one 'Self' for the entire system. This simplifies deep overrides across component dictionaries, and also directly overriding the dictionaries.

### Configuration

The assembler implicitly loads a configuration module based on the `GLAM_CONF` environment variable or an OS-specific default, i.e. `"~/.config/glam/conf.g"` in Linux or `"%AppData%\glam\conf.g"` in Windows. A small, local user configuration typically extends a large, remote community or company configuration.

The configuration serves several roles:

- *Assembly environment*: define 'env'. This is passed to the assembly as if importing the assembly into the configuration, e.g. `{ "env": Config.env }`. This environment can provide default target information, system includes and shared libraries, etc. for adaptation.
- *Command-line macros*: if the first command-line argument to the assembler does not start with '-', we apply 'cli', which should be a function of type `List of String -> List of String` that returns valid arguments or empty list.
- *Development environment*: Define a loop for '-i' interactive mode. Filter outputs to standard error if non-interactive.
- *Resource management*: ad hoc, e.g. specify GPGPUs available for acceleration, cache locations and replacement heuristcs, history management, shared proxy compilation and cache, search locations for content-addressed remotes, tune assembler JIT or GC heuristics, control expensive tests and checks (e.g. assertions, fuzzing).

Configurations never directly control assembler output: An assembly may ignore your configured environment and substitute its own. Command-line macros may always be written out long form. Resources influence performance and error detection but not a valid binary result.

To support project-specific overrides or sharing of a system configuration, `GLAM_CONF` may list multiple files (same OS-specific separator as the `PATH` variable). We apply these as mixins, each file overriding those listed later.

### Assembly

The assembler receives command-line arguments that express an assembly module as a list of mixins. Though, in practice, it's usually just one file or script. Relevant arguments:

- `(-f|--file) FileName` - list a file to include; first file is included last, overriding those listed later. Depending on the configured environment, assembly isn't limited to ".g" files (see *User-Defined Syntax*).
- `(-s|--script).FileExt Text` - behaves as a remote file with the given file extension and text. Scripts cannot import local files.
- `-- List Of Args` - assembler defines 'args' before including files or scripts. Default is empty list, but caller may override with elements following the '--' separator.

Aside from 'args', the assembly module is implicitly parameterized by the configured 'env'. The assembly module shall define 'result', representing the assembled product, i.e. a binary or folder.

### Remotes

Remote files are content-addressed, typically at DVCS scope. This might be expressed as a dict or object with:

- DVCS protocol (git, hg, darcs)
- DVCS repo revision hash
- filename within repo
- list of repo URLs (backups!)
- tag or branch name(s)

The file is uniquely identified and authenticated by revision hash and filename. The repo URLs support multiple backup search locations; this list may be rewritten by the user configuration. A tag or branch name is used only as a hint for efficient download, such as: `git clone -b Branch --single-branch URL`.

Syntactically, remote files are awkward. They are not concise, and maintenance of revision hashes scattered or duplicated across multiple files is a hassle. The latter can be mitigated by centralizing remotes per repo or project. Regarding concision, I'll need a presentable multi-line import syntax, separate definition of locations, or both.

*Aside:* We aren't restricted to DVCS. Viable alternatives include download of secure-hashed zipfiles, or even individual source files. But my intuition is that DVCS will offer the best development and maintenance experience for content-addressed structure.

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

When loading a module, a front-end compiler is selected from the provided environment based on file extension, i.e. `Base.env.lang.[FileExt]` should evaluate to an object or dict that defines 'compile'. This object may also define auxilliary methods for tooling or the interaction loop, e.g. syntax highlighting or a specialized language-server protocol. To normalize FileExt, we lower-case 'A-Z' and strip initial '.'. In case of ".g" files, the assembler provides a built-in as a fallback, but we favor a user-defined compiler.

The 'compile' method is expressed effectfully, as a parser combinator. The effects are carefully designed to simplify tracing of errors back to sources, isolate parse errors, support edit suggestions.

- Parser combinators are a great starting point. Parser combinators can implicitly track parse locations and describe what they 'expect' to see at any given step, providing effective feedback in case of syntax errors and metadata for tracing.
- To simplify tracing, blame, error isolation, etc. the compiler must avoid directly observing parse results. Even parse errors must be abstracted. To enforce this, we return abstract data from parse operations by default, perhaps expressed via applicative functor. We must provide built-in combinators for optionals and loops.
- In context of error isolation, we cannot model monadic 'state' at compile time in general, but we could support a few special cases where state is easily 'forked' for parsing subcomponents, especially gensym: abstract, unique, unforgeable symbol generation.
- We can feasibly support something like a 'writer' monad for tests, illustrations, editable projections, indexes, etc.. However, if we hope to preserve lazy loading, we must be careful about automatic composition of indices. Use of annotations may mitigate this.
- To support syntax-driven effects without observing the syntax, we can introduce an eval effect that lifts a user expression into the front-end compiler. The result of eval is then abstracted. This effectively supports some forms of macros.
- It is useful to support multi-pass parses. For example, a first pass might delimit the 'region' for another pass. Even if we drop the first pass result, this can be useful for error isolation.
- Compiler keeps module dictionaries (Base, Self) abstract, user only accesses them indirectly. This simplifies monadic tracking of dependencies.

Expressing a compiler without being able to 'see' the binary seems possible in theory, but I'm not entirely convinced that we won't reintroduce the tracing problem via Eval. To gain confidence, I must try it first with the standard syntax. After all, we should be able to override and extend the standard syntax, too. Worst case, I'm back to the conventional approach.

### Syntactic Bootstraps

If the final `Self.env.lang.[FileExt].compile` method is different from the Base version, the assembler attempts bootstrap. This involves recompiling with the Self version, repeating until fixpoint is reached.

        bootstrap(ext, base, binary, compiler) =
            let m = compile(compiler, base, binary) 
            let compiler' = 
                m.env.lang.[ext].compile if defined
                or builtin if ext is "g"
            if(compiler == compiler') then m else
            bootstrap(ext, args, compiler')

A built-in compiler is simply treated as one more compiler in the bootstrap cycle, equivalent only to itself. 

### Editable Projections

Some user-defined syntax may be graphical. And even for purely textual syntax, we'll often want to integrate some visualizations or provide edit widgets like color pickers. Miscellaneous observations:

- We can only edit terms annotated with source locations, parser context, and an encoder that converts the parse result back into source (aka lenses or prisms). We can verify any edits for round-trip between parse and encode.
- Some content is naturally read-only, e.g. content-addressed remote files, read-only local files. In these cases, we can still support navigation, views, etc.. and partially-editable views are feasible.
- Support for navigation, progressive disclosure, etc. benefits from user-local or session state. 
- It is convenient to treat a file that does not exist as equivalent to an empty file for purpose of imports and projections.
- Ideally, we can obtain immediate feedback on edits by visualization of tests. This may require some integration of projections and tests.

TBD: This is non-trivial, and I don't have a solid handle on exactly how to approach it.

## Reasoning

What can we feasibly implement to support developers in reasoning about the assembly process and product?

* *Types*: We can annotate assumptions about programs and data in a composable, machine-checkable way. It is feasible to reason about an executable binary's runtime behavior insofar as it is encoded into types. We may leverage 'type interpreter' functions that compute type annotations from data or AST embeddings, similar to GADTs. In context of gradual or partial type annotations, we must assume not every term is typed, and the assembler makes a best effort to prove or disprove types.

* *Tests*: We can sample subprogram behavior under various conditions. With acceleration, it is feasible to emulate execution of machine code. Tests should support heuristic, non-deterministic choice to include fuzzing and property checking and race conditions, enabling the assembler to heuristically choose tests from a vast space for maximal branch coverage. Tests should be a convenient foundation for visualization of program behavior.

* *Contracts*: Contracts might describe a monadic subprogram's preconditions, postconditions, and invariant assumptions in a locally testable way, perhaps via assertions. A relevant challenge is how to effectively use contracts in context of staged metaprogramming of an executable binary.

* *Theorems*: The main difference between a theorem and a test is a 'forall' or 'there exists', and the need for abstract interpretation to support the proof. In practice, we may need explicit proofs or proof-hints to efficiently verify theorems. 

* *Visualization*: Start with logging and profiling, extend to graphical views of tests, progressive disclosure or search, etc.. I propose to model log messages as dicts or objects implementing multiple views. Plain text is one view, but we can feasibly render icons, interactive widgets, etc..

* *Tracing*: Maintain metadata to trace outcomes (errors, data, etc.) back to contributing sources. There's a tradeoff between precision and performance. We can feasibly leverage reproducibility by replaying a computation under a few different tracer setups to obtain more precision.

* *Abstract Interpretation*: Given a representation of a program (e.g. machine code) we can implement an 'interpreter' using variables instead of data. Users add their own assumptions about this interpretation, then we check for conflicts. Essentially, we can mechanically implement a type system scoped to our target. Can feasibly integrate types via 'type interpreter' functions, and tests via constraint systems.

I envision use of type annotations to catch obvious errors in the assembly process, abstract interpretation as our primary means to reason about the product, and testing in a more ad hoc role. Visualization should build on tests and integrate nicely with a projectional editor.

### Integration

Some extensions are non-monotonic and may 'break' prior assumptions. Ideally, the same extensions that break assumptions can repair them. Extensions are aligned with the namespace via modeling modules as mixins. Thus, we should align types, tests, contracts, visualizations, etc. with the namespace. Further, in context of lazy loading and shared environments, we'll also want to evaluate only 'relevant' reasoning.

In context of these forces, it seems types, tests, contracts, etc. should bind to definitions. It makes sense to name individual tests or contracts for both error reporting and fine-grained overrides, and it might prove useful to name 'overlays' for types, too.

We can feasibly integrate reasoning within extended user-defined objects, not just the module layer, insofar as those objects are specified in the module layer (instead of buried within another function). Ideally, the monad for user-defined syntax supports flexible bindings while structurally guaranteeing relationships to the namespace.

### Tests

Testing samples behavior under a range of conditions. In context of fuzz testing and property testing, it is convenient to blur the lines between tests and theorems, enabling the assembler to make heuristic decisions about about what to test to maximize branch coverage.

Useful properties:

- constraint system, derive test parameters from assumed conditions
  - abstract simulation of environment, e.g. race conditions
- non-deterministic choice to *sample* test parameters, environment
  - both discrete and continuous, i.e. integers and rationals
  - assembler may use abstract interpretation to defer choice 
- status and state, something we can visualize and animate
  - bind to constraint system, use temporal logic for state?
- returns a pass/fail judgement

Monadic expression of tests seems well-aligned to these goals. Perhaps we express test parameters and expectations in context of the constraint model, then sample constraints as a non-deterministic choice. An assembler can feasibly perform whole-volume tests in some cases, but with tests we don't necessarily insist on a full proof.

The assembler may perform tests randomly, but is expected to perform them heuristically. Of course, early heuristics might not be very good. Users are free to favor deterministic testing in these cases. The assembler may cache tests for regression testing, may save checkpoints to simplify replay of discovered errors, and can potentially support work sharing for tests between users via shared cache.

In interactive mode, it is feasible to fuzz test indefinitely. On updates, some old tests may become 'stale' based on incremental computing. It might be useful to visualize stale tests together with new ones, perhaps distinguishing by color. In non-interactive mode, we'll want configurable, heuristic quotas for how much testing to perform.

### Types

The underlying lambda calculus is untyped, but we can express ad hoc type annotations. I imagine type annotations will be incomplete, partial, gradual. The assembler makes a 'best effort' to either verify or contradict type annotations ahead of evaluation. If types are neither proven nor disproven, the assembler emits a warning. 

There are no 'dynamic' type checks during evaluation. Although feasible, dynamic typing has unpredictable performance implications and complicates type-level debugging. That said, dynamic checks may be useful for heuristically tracing blame.

Under these constraints, what types can we express?

- structural data types: numbers, lists, symbols, dicts, tagged unions. We can refine common number types such as rational, integer, or natural. Dictionaries and tagged unions may be 'open' to extension with new cases.
- user-defined nominative types: via tagged unions with guaranteed-unique symbols. Ideally, we can express GADTs and dependent types, so we can express our assumptions even when we cannot fully check them.
- substructural types and session types are feasible within limited scopes. Modeling them in effects handlers is probably the most relevant, e.g. to model channels typed with protocols. 

We cannot directly express type-indexed behavior because type inference at the annotation layer does not influence program behavior. Indirectly, we could ensure that types are aligned with observable structure, such as symbols, which can be used in dispatch. 

*Note:* Type checking in context of gradual types remains vague to me. My intuition is that these should clear up a lot once we start defining a language of type descriptions.

### Constraints

Constraint systems are a convenient mechanism for abstract interpretation. We can feasibly accelerate constraint systems with cvc5, Z3, or other solvers. Unfortunately, the accelerator cannot observe the *solution* because it's effectively non-deterministic. But we can use the sat/unsat judgement.

The DSL for constraints can be adapted almost directly from SMTLIB2. For variables, we can easily use `(Var x)`, where 'x' may be any valid dictionary key. We easily can control naming conflicts via module-level gensym and local naming strategies.

It is relatively convenient to build a constraint system statefully within a monad, and occasionally 'Check' for sat/unsat. We can align constraints with abstract interpretation.

### Visualization

A relevant question is how we express visualizations. Per *integration*, visualizations must align with definitions. We can feasibly support multiple visualizations per definition. But it will likely prove more convienient to bind visualizations to *types*. Then bind types to definitions. This simplifies expression, composition, and reuse of visualizations.

To visualize a function, we can visualize the arguments and results and perhaps some intermediate representations during evaluation. In case of curried arguments, we'll want to tie invocations to both call site and origins of terms or definitions site, and support browsing from either location.

Rendering should be extensible, e.g. support both 'text' console output and SVG. Some visualizations should be interactive, e.g. so we can rotate a 3D graph or apply progressive disclosure. We might model visualizations as dictionaries or objects that define recognized interfaces for common viewers.

### (Tentative) Theorems and Proofs

We might express theorems as tests with an explicit intention for exhaustive testing. We can feasibly annotate both theorems and tests with *proofs* to cover entire ranges more efficiently. The question, then, is what would a proof look like? Perhaps some hints for how to partition constraints, and strategies to skip concrete evaluation. I'll need to review what is feasible here.

## Assembly Programming

Instead of machine-code mnemonics being something an assembler *interprets*, we can model them as monadic operations that the assembler *executes*. Each mnemonic can:

- write some binary data to intermediate staging location
- maintain an abstract representation of program state
- declare ad hoc external structure (.text, .data, .rodata)
  - i.e. so we can deref 'structures' instead of 'pointers'

Ideally, there is no need to forward-declare labels for jumps, leveraging monadic fixpoint and staging to defer the pointer. Ideally, we can also support declarative allocations for static data, stack, and heap, i.e. where the amount allocated depends on future *assumptions* about which fields are present.

*Aside:* Monadic expression is generally more robust and compositional than macros, yet similarly expressive. We are able to define higher-level mneumonics that are inlined.

## Standard Syntax

This section proposes an initial syntax for ".g" files. It should look and feel like assembly proramming in many cases, and favor vertical structure over deep indentation. Associative structure needs special attention because it enables future operations to influence how much is allocated.




