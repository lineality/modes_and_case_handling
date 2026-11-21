
(Under Construction)
### Modes & Case Handling Framework 
2026 Geoffrey Gordon Ashbrook

# Modes and Case-Handling

#### Framework/Template for Mode & Case Handling:
- 3 Modes
- 5 Rules
- 3 Recovery-Levels


### What are the questions?

The overall intention here is to try to framework the overall problem-spaces around what I will call "modes" of processing and compiling code 
look back over time at what has been done and discussed and to explore and 

Each language, and future, each project in each language, may (and likely should) have different answers, but there may be value in having a somewhat standard set of starting questions that:
A. People (individuals and teams) doing a project can use as a planning guide as a communication aid. 

B. People can use to aid a somewhat interdisciplinary discussion across projects and languages, domains, etc.


Maybe Universal Questions:
1. How are you managing Test-Mode?
2. How are you managing Debug-Mode?
3. How are you managing production-release mode?
4. Are there (other) modes that this project needs (or does not need)?
5. How are you using terminology around 'modes' and 'cases, errors, handling,' and 'recovery'?
6. How do you recover state in the event of "errors"?(e.g. 3 levels)
7. Stack, Heap, Memory: Are there rules or priorities around using heap and stack memory?
8. Threads, Concurrency, Atomics: 
9. Error-tracability: What are your needs and plans for tracing errors?
10. Risk, Priorities & Scope: Are there concerns such as security or data privacy? (e.g. in terms of how cases/errors & memory are handled)
11. How maintainable for how long in the future is this project intended for? (e.g. From 'This afternoon, used once ever' to 'Many decades')
12. Are there language-features that you want to preclude from use for this project (e.g. as part of the above answers)?
13. What are the pros and cons of different languages and approaches in the context of the above answers?

14. Is it not correct that everyone tautologically assumes that everyone else is explicitly (authoring a spec) or implicitly (adopting some other official spec, or using a 'language' that enforces a spec) manages modes and cases? Yet, even in Rust (which is a language that is supposed to be unusually explicit about clearly handling safety, modes, and cases) simply outlining basic points of management and giving basic examples is perplexingly convoluted and verbose. Yes, added reasoning, detail-exploration, and context is included in this doc, but the 'bullet point summary' version should be able to cover all salient points (of what it is presumed that everyone is already accounting for by some methodology) in about a one-page outline. And where (online, in books, in courses, in code-bases, in project and product planning) is the authorship and enactment of such Mode & Case specification around the world 2026?


Where a detailed set of coherent answers to the above questions may constitute a 'framework': The goal of the framework described below is not an attempt to create a one-sized-fits-all meta-uber-TOE-framework, but rather to explore one example of a (hopefully close-ish to) coherent approach expressed in one language (Rust).

Hopefully in future, as more such frameworks are compared, it will become clearer what general-universal questions and aspects there are, if there are common domain specific variations, and which areas are too highly domain specific to try to speak about in general terms. Progress on this is a goal.




#### 'Power of 10' Style Modes and Case-Handling in Rust:
- Bullet Point Outline Summary
- More Detailed Discussion



# Compile-Mode-Differentiated Case & Error Handling Policy & Framework: A Three-Mode System using Fieldless Error-Codes

Setting aside issues with terminology,  and the details of specific programming languages, the 2006 'Power of 10' paper sets out a coherent argument for a general need to manage things such as errors across at least universal modes of running and testing code: A real-world-use mode, a debugging mode, and a testing mode. 

In a general and abstract way this could set out something like this:

1. Production (real world use): The satellite can't crash, every issue must be handled smoothly, and no security-issues can be allowed (including no sensitive data leaked in 'error' messages).

2. Debugging should be close to Production, but during errors as much diagnostic data as possible should be reported (no privacy here).

3. Testing makes use of deliberately crashing the program to get deep stack trace granularity about what went wrong, and to detect specific issues.

Since NASA has articulated this, it should be common sense to try to follow these concepts. To do this there are at least two classes of challenges:

1. There are notable ambiguities of terminology (including in the 2006 paper) that need to be untangled and clarified so that a detailed plan is unambiguous.

2. In each language (c, Rust, Zig, etc.) there is a context of what can and should be done and how within that language (and often unique jargon for that language).

This framework attempts to clarify the terminology and to present a preliminary model for how case-handling per mode can be done using Rust. These goals can be laid out in more detail in a page or two of text (not too long), but, as you will see, it is often not trivial to follow these (common sense) rules within the grammar of a language. This raises questions: As in the spirit of 'Nudge' (See Thaler & Kahneman), this mysterious gauntlet of challenges makes it less likely that most software will be robust. Language design should nudge people in better default directions, not worse directions and outcomes.

As a note, while the main focus here is a framework for Rust, having an ever-deeper understanding of how case-handling works in different languages, such as Zig, is likely important for ongoing improvements in the area of mode management and case handling.



## Terminology Problems

One of many terminology problems here is around the jargon for 'production' 'release' builds of code and products. In the wider world 'production' does not have the same meaning that it has in IT and software development. In film, for example, 'in production' means the film is still being shot, and 'post-production' refers to work after shooting scenes and before the film is finished to show in theaters (such as final editing, effects, etc.). For software, web-development, data-science, and cloud-deployment (such as AWS built cloud infrastructure), 'production' is the conventional jargon term meaning 'in real-world use' where there are often 'development,' 'staging,' and 'production,' versions of the system. 'Production' software is the type used by real users, live in the real world. The phrase 'in production' is common to refer to either the production tech-environment or to 'production' meaning real-world-use, or both. (It not clear who decided to describe software that is finished and being used in the real world as 'still in production,' but as the saying going in computer science 'naming things is hard.')

Rust, and likely other languages, have a similar range of phases (and phrases), such as testing with cargo-tests, debug-builds, and there is another jargon-term "release" that is likely comparable to 'production.' These terms and modes are built into the Rust-cargo system by default. You can make your own custom modes too, but those three are always there.

While not ideal, as the average person on the street would have no idea what 'production-release' is supposed to mean, I will try to consistently use the term production-release to refer to the area of final for-real-world-use code/software and product. This hopefully bridges the standard software 'production' jargon and the Rust 'release' jargon, and avoids (as if often difficult in computer science) the problem of alternating between terms (which likely confuses various readers in different ways). The terms "in production" and "release" still exist on their own in the world (the rust term is 'release' not 'Production-release,' developers say 'stage and prod' not 'stage and prod-release,' a 'production environment' is not called a 'production-release environment'), and so these terms may be used in those contexts in this paper. But an attempt is made to consistently refer to production-release code and production-release mode in a consistent way here.


## Three goals (in priority order):
1. Production-release never panics, never halts, never leaks data: Handle and move on.
2. Minimal memory-use in production: no heap in production-release error paths, no error strings compiled into production-release binaries.
3. Ideally: Every error is traceable to one exact location in the code.


There is a lot of room where different projects will go in different directions, such as if and how logging is done, if and how heap-memory is used, and innumerable other case-by-case decisions. The focus here will be the above three goals.

"Static-code-analysis" and avoiding hidden misbehaviors are arguably within the overall message of the 2006 paper and are reasonably generally applicable as sound advice. Memory-use and error-tracing are not strictly necessarily in scope for all programs to be safe and not crash, but the goal here is to try to architect a strict framework for default robustness. Future maintainability is another aspect of this, by being strict we can allow future additions to a project to have a greater chance of being just as sound as the original (instead retractively realizing that the changes broke things in ways that no one ever can ever fully untangle).



## 1. The Three Modes of Output & Case/Error Handling
- Test Mode
- Debug Mode
- Production-Release Mode

Per function: The same function source-code compiles into three behaviors. The function signature and the returned error VALUE are byte-identical in all three builds. The difference is in how the function behaves (per mode) when an error occurs:
- In Test Mode, it can/should crash/panic to produce a detailed stack trace.
- In Debug Mode, it should print or log extensive diagnostic data (and use heap).
- In Production-Release Mode, it must handle cases smoothly (sometimes 'silently'), without crashing, or leaking data (with no heap).


| Mode | Build flag | Panics? | Heap? | Output |
|---|---|---|---|---|
| **Test** (`cargo test`) | `#[cfg(test)]` | yes — `assert!` **in the test function only** | allowed | verbose; asserts on returned error values |
| **Debug** (debug build, not test) | #[cfg(all(debug_assertions, not(test)))] | invariant: yes (debug_assert!); input-validation: no | allowed | invariant: panic + message; input-validation: eprintln! + error returned |
| **Production-release** (release) | always-compiled path | **never** | **never** | terse 2-byte error code returned to caller; no terminal print |


### Note: This framework assumes there is no additional 'shoot-yourself-in-the-foot' action, such as turning debug-compilation (or test code compilation) on in your production-release.(!) As Steve Gibson of GRC says: "Just because you can does not mean that you should."

```toml
# Please do not do this:
[profile.release]
debug-assertions = true
```

#### Recommended: Settings explicitly in Cargo.toml
State production-release settings explicitly in the Cargo.toml, even (especially) where they match defaults, so that a change to any of them is visible:

```toml
[profile.release]
debug-assertions = false   # debug_assert! and #[cfg(debug_assertions)] code compiled out
overflow-checks = false    # plain arithmetic wraps; this framework uses checked_* instead
panic = "unwind"           # see "Panic strategy"
```

In a Cargo workspace, profiles are read only from the workspace-root Cargo.toml. A [profile.release] section in a member crate's Cargo.toml will be ignored (Cargo prints a warning).


#### Panic strategy (panic = "unwind" or panic = "abort")

Where a project goal is to not crash-halt-stop, unwinding may seem less desirable for this framework, but configurations of unwind can be managed to fit the goals of this framework (better than abort-can).

This framework's default-assumptions (real projects will vary) aims for zero panics in production-release, but the standard library and dependencies can still panic. So panic setting decides what happens then (in production-release-mode):

- panic = "unwind" (default): panic unwinds stack of thread it occurs in. A panic in a spawned thread ends only that thread (its JoinHandle::join() returns Err). std::panic::catch_unwind can stop a panic at a chosen boundary, so a Tier-3/Level-3 outer loop can catch it and re-initialize.

- panic = "abort": any panic terminates the whole process immediately. Binaries are smaller, but no in-process recovery is possible.

Some failures terminate the process under either setting: stack overflow, a failed heap allocation (out of memory), and a second panic during unwinding. After the process has terminated, only something outside the program (e.g. systemd, a hardware watchdog timer, or a parent process) can restart it.

### Panic message text: 
Panic prints its message and source location to stderr by default, which in production-release can expose data. Install a silent panic hook at the start of main (for production-release builds only):

```rust
fn main() {
    // Production-release only: suppress panic message text.
    // (One heap allocation at startup, not in an error path.)
    #[cfg(not(debug_assertions))]
    std::panic::set_hook(Box::new(|_info| {}));

    // ...
}
```

### Rules for Modes

1. `debug_assert!` in function bodies (that is not intended to run in cargo tests, e.g. that conflicts) must be "gated" using `#[cfg(all(debug_assertions, not(test)))]` so the code does not run in other modes. Debug code by default DOES run in test-mode. Note: cfg(test) applies to 'own-crate' unit tests, not to "integration tests" (e.g. 'tests/' dir, benches, doctests, other crates, etc.)

2. Verbose diagnostics at a failure site are `eprintln!` gated `#[cfg(debug_assertions)]`. They may use heap (`format!`, `.to_string()`) freely — they are excluded from production-release code.

3. The production-release path (the `if`-check and the `return Err(...)`) is always compiled, in every build. It is the only path that is used in production-release for real-world use.

4. Test-mode code must be gated with #[cfg(test)] and be careful using test-mode tests to debug inside functions. As a rule of thumb, do not use test-mode tests inside functions in stable code, ensuring that all code is testable.

Because production mode is all smooth-cases (effectively silently handling what would be errors), to clearly identify and trace 'errors' test-mode or debug-mode inspection must be used. Values inside a function can only be tested inside a function (i.e. using #[cfg(test)] in the function), in other words: cargo-test is not able to perform tests on specific values at specific places within a function. Only use test-mode inside a function when the desired result of the test is a panic-stack-trace, as this will prevent the rest of the program from running. The risk of stopping the program by using a test-mode test inside a function will prevent later parts of that function from  running. So a cargo-test in a function should be considered a short-term testing-debugging tool (e.g. if you need that stack trace to fix the code, then that is a tool in your toolbox to use), but once the code is stable then (unless in exceptional circumstances) test-mode tests inside functions should be avoided so that the entirety of code is testable and there is no 'untested code' region that was behind another test.

Rule 5. Value-Integrity & Enforced-Custom-Types
Where code can and should enforce value-integrity (e.g. values being outside of a given definition due to design-logic-bugs, bitflips, logical and physical memory issues, etc.) at compile time and run-time:

1. As of late 2026 (Rust-2024 rustc/cargo v1.100.0, Zig v0.17, Odin dev-2026-11) there is no way to automatically protect runtime values
(Note (late 2026): fil-c might have some run-time guards, and zig might be looking to adopting some of those, but this is in very early development)

2. Compile-time values can be guarded using structure-enforced-types using structs, impls, and enums (in Rust) (TODO: Q: Zig 0.17, Odin?, Fil-c?)

3. In some forms, Non-Automatic manual-enforcement is possible using struct/impl/enum systems. 

#### Manual and Periodic, Not Continual and Automatic:
As a way to enforce handling of 'should not happen' value-changes (such as bit-flips, hardware issues, or adversarial attacks) that could happen to values as they move through functions (e.g. after any given check-point and thereby not caught, a struct/impl/enum system does not automatically require that the value-bounds are continually enforced, but can be used to manually check and enforce at specific points (e.g. at the beginning of function to check what is being passed in) to move towards a production-safe no-panic/no-crash smooth-handling of more such cases of invisibly-invalid data-values/memory-state being caught and smoothly-handled as errors, not passed silently (or resulting in a halt/crash/panic).

A constructor for a custom-type performs one run-time value check for custom-types at the time-place-point of value-creation that validates untrusted input (only once) and packages it into a (one-time) verified type. Internal functions can be defined to only accept and return more strictly defined custom-types (not broader generic data-types).

Corruption of values (after creation) is partially possible to guard and handle if optionally built. In some ways emulating the constructor's check, this can check some definition-bounds, but cannot simply identify if an in-bounds corruption-change occurred. (Methods such as error-correcting-codes [ECC] and/or parallel check-values might be used to identify in-bounds changes.) 


# Three-Level Process-Recovery Hierarchy ("Let It Fail & Recover")
Every production-release failure resolves into one of three bounded recovery tiers. Any level (if especially levels 1 and 2) might also make use of 'Value-Check & Value-Recovery' mechanisms as well (see appendix). 
- Level 1 Recovery: Micro-Retry (Local) — If err.is_retryable() is true, the immediate caller repeats the bounded operation (with backoff/sleep).

- Level 2 Recovery: Step Fallback / Safe Degradation (Subsystem) — If non-retryable or attempts are exhausted, the subsystem safely aborts the current command, resets its internal state to a known-safe baseline, drops invalid work, and reports the error code up the stack.

- Level 3 Recovery: Macro Re-initialization (Supervisory) — If an unrecoverable invariant/hardware failure occurs, the outer supervisor loop flushes queues, reinitializes the module/device, and continues execution without taking down the entire runtime.

#### Standard types of recovery-handling include:
1. Panic & core-dump (not for production-release)
2. Return an Error (that then needs to be handled/recovered-from)
3. Attempt repair, reconstruct state from other data and move on. 
4. Re-try the failed process (N times)
5. Backtrack and retry from a lower level.
6. Attempt value-check and value-recovery as part of retrying a process.
7. Use a default value and move on.
8. Do nothing and move on. (e.g. if the process was a 'window-dressing' feature that in no way cascades issues for future state)
9. Log / terminal print


### Disambiguation & Vocabulary 

The original 2006 'Power of 10' rules contain a confusing use of the term "assert" regarding a number of critical valid points vital to the guidance in the paper. 

The 2006 'Power of 10' rules use the single word "assert" when describing three importantly-differing meanings and situations (without providing a way to specify the differences):

1. The assert() macro, which halts the program on failure;
2. The general verb "assert a condition" — to check that something must hold, independent of what happens on failure;
3. A check that, on failure, takes a recovery action (e.g. returns an error) — the opposite of halting.

The 2006 power-of-10 rules demand all three at once: 
1. Tight limits on conditional compilation — even though the standard assert is implemented by the preprocessor and DOES halt (case 1)
2. At least two "assertions" per function (case 2)
3. Never Halting in Production-release: Assertions that take a recovery action rather than halt (case 3)

Such terminology can lead to confusion (e.g. where you are both required to "assert" and prohibited from "assert"ing, because the term "assert" does not have one specific & consistent meaning), especially as there are no clarifying annotations for any use (let along all uses) of the term in the original 2006 power-of-10 paper. The spirit of the paper is clear, but the details become a quagmire.

The term "assert" cannot easily be used precisely here, so this framework aims to consistently use three descriptive terms instead:

1. **required-condition**: the rule itself (e.g. `len <= 32`), independent of build.
2. **requirement-check**: the code that tests a required-condition (the always-compiled `if`).
3. **reaction-to-check**: what happens upon failure, which varies by mode:
  test → panic in the test; debug → panic / verbose eprintln; production-release → terse code returned (possibly a retry-first) (never panic, never halt, never stop: the satellite must not fall out of the sky).

Similarly, this framework also aims to clarify the semantics around "errors" in the same context, where technically an "error" does not refer to all required ways of handling a ~case (in this context), and where there are three importantly distinct modes for how 'case-handling must (or must-not) be done. We need to clarify:

1. the build/compilation mode (production-release / debug / test), and
2. the difference between a case (any condition that must be handled) and an error (a case that is handled by returning Err).

Unfortunately there is no single common-language and technical-term that covers 'error-like-things' across all three modes that we need to manage: test-mode, debug-mode, production-release-mode. The somewhat bulky phrase 'cases that need to be handled' may be the closest we can come, though even that lacks an emphasis on erroneous or problematic-case handling (vs. every part of code that "handles" some logic in a neutral sense). In casual language the word "error" may be the simplest and most accurate term, but as with the term "assert", the word 'error' has been given specific technical meanings that make planning and communication fraught. Which meaning of 'assert' or 'error' is someone using in every instance? Colliding-terms make confusion unavoidable.

Not every "handled case" is technically an "error." The same case may be silently ignored, re-tried, or (in test mode) turned into a 'panic.' Keeping "case" and "error" distinct is a critical part of the source-code management that is implicit in the 2006 power-of-10 guidance, managing specifically different per-mode behaviors in the same 'cases.' We should strive to make more of the implicit wisdom of the spirit of the power-of-10 more explicit and robust for systems-programming and software education more broadly. There is a low probability of all readers of the original 2006 document following a uniform framework with a shared disambiguated lexicon.

## Defensive Programming: "expected" issues vs. "should-not-happen-check"

Another area that can be confusing is 'errors' or 'cases' that logically should-not happen but that in the real world (often) do happen. For whatever reason there is no standard terminology for this concept, and so important defensive-programming checks often appear to be silly mistakes made by a developer (why check for something that in a perfect world cannot happen?) and are then removed by later developers (and then the software crashes 'in production'). 

A solution that hopefully covers this is to standardize two
types of checks. We have "input-validation" that covers the range of 'expected-type'
problems (problems that we know are possible), such as a value being outside of an acceptable range. 
And we have the "should-not-happen-check" for so-called "unexpected" problems (problems that are impossible in an ideal-fictional-world).

Another reason to define these two is to standardize where it may be usually best
to do each type of check.


### Input Validation ("expected") vs. Internal Invariant ("should-not-happen-check")

- **Input validation** (e.g. bad user input, missing file, oversized argument) :
  an EXPECTED case. Requirement-check + terse return
  only, plus debug eprintln.
  Default debugging for input-validation should be non-fatal (eprintln + return) to provide debug-data without panic-halting. 
  Case by case, if a panic-backtrace provides the data needed then panic-assert (for debug) is an option.
  
- **Internal invariant** (a value that in a perfect-world 'cannot happen' but that in real life can be the result of a logic-bug elsewhere in code, a bitflip, hardware failure, adversarial attack, or other corruption): all three modes apply — debug_assert, test assertion on the
  returned value, and the always-compiled production-release catch. a defensive should-not-happen-check

#### Classification heuristic to decide which pattern to use:
Ask the question "Can this condition fail when the code is correct and memory is sound?"
- Yes -> Input validation ("expected" type)
- No -> Internal invariant ("should-not-happen-check")



## 2. The Error Type: One Fieldless Enum

The project error type is a single fieldless `enum` with explicit e.g. `u16`
discriminants. The discriminant IS the error-code.

#### Note on size choice:
- A tiny project might use u8 (fewer than 256 codes).
- u16 is probably the most common: 65,536 codes
- An extra-big project might need u32 or larger.
- u32 would also allow more sub-divisions or large-blocks of codes, e.g. if you wanted regular block sizes and little risk of running out. 2**16 being 65,536 looks inexhaustible, but if your code-base grew to have 200 types of errors that you wanted to separate into blocks, then that caps a regular block size at 327, which looks less inexhaustible.


#### Single, Central YourProjectError Enum Across Module-Files
The ABCProjectError enum is defined once (e.g., in one .rs module file) and is 
imported wherever errors need to be returned or handled. All modules use the 
same enum variants, so error codes are globally unique and consistent.


```rust
// Debug printing (variant names as text) is compiled in for TEST and DEBUG
// builds only. In a production-release build no Debug impl exists, so no
// variant-name strings are baked into the binary — and any stray `{:?}`
// on a YourProjectError in production-release code fails to compile (intended guard).
//
// Why `any(test, debug_assertions)` here but only `debug_assertions` on
// Display and eprintln!: the `test` arm lets this crate's unit tests use
// assert_eq! under `cargo test --release` (worth running: release builds
// wrap on overflow instead of panicking). Display and eprintln! are not
// needed for that, so they stay debug-only.
// Integration tests (tests/ directory) do NOT get the `test` arm, so under
// `cargo test --release` they must compare codes, not variants:
//     assert_eq!(result.err().map(|e| e.code()), Some(NNN));
#[cfg_attr(any(test, debug_assertions), derive(Debug))]

// Always available, in every build:
// Copy/Clone — the error is a trivially-copyable 2-byte value (no heap).
// PartialEq/Eq — lets callers and tests compare codes (`err == YourProjectError::ThisCondition`,
// `assert_eq!`, `matches!`).
#[derive(Copy, Clone, PartialEq, Eq)]

// Fixes the in-memory representation to a u16 so the discriminant IS the
// 2-byte error-code, making `self as u16` well-defined and the size exactly
// 2 bytes.
#[repr(u16)]
pub enum YourProjectError {
    // io categories (mapped from io::ErrorKind at failure sites)
    IoInterrupted = 1,          // retryable
    IoWouldBlock  = 2,          // retryable
    IoTimedOut    = 3,          // retryable
    IoNotFound         = 50,
    IoPermissionDenied = 51,
    IoOther            = 99,

    // one block of codes per file/struct/feature — see Error-Code-Table rules below
    Fs32tFromStrInputTooLong   = 101,
    Fs32tAsStrLenExceedsBuffer = 102,
    Fs32tAsStrInvalidUtf8      = 103,

    RetryMaxAttemptsZero = 200,
}

impl YourProjectError {
    /// The terse numeric error-code. Available in ALL builds. No heap.
    pub fn code(self) -> u16 { self as u16 }
}

pub type Result<T> = std::result::Result<T, YourProjectError>;
```

### Useful Properties of Fieldless Enums (for this use-context)

- `Copy`, 2 bytes. No heap anywhere in the error path.
- NO `String` payloads. NO `io::Error` payloads (`io::Error` is not `Copy`
  and can itself hold a heap string containing paths/PII).
- The compiler enforces both name uniqueness and discriminant-value uniqueness, so no two variants can collide on a name or a code. (See https://doc.rust-lang.org/error_codes/E0081.html)
- Callers can `match` exhaustively.
- Variant payloads are forbidden. If a caller needs a fact (a length, a
  position), the fact justifies a distinct variant, not a payload.
  Payloads are the leak vector this policy exists to close.


### Error-Code-Table rules (tip: show in enum's doc comment)

1. **APPEND-ONLY.** A code number is never reused or renumbered, ever.
   A code in a years-old field log must still mean one thing.
2. A file/struct/feature can get a reserved hundred-block
   (e.g. `100–199 FixedSize32Timestamp`). Blocks are listed in the enum doc.
3. Variant names are `AcronymFunctionCondition` (e.g.
   `Fs32tFromStrInputTooLong`) — unique, traceable, greppable.
4. The number to meaning table lives in the enum's doc comment. It is
   DOCUMENTATION, not shipped binary data.


### Display: for debug builds only

```rust
/// Human-readable text — DEBUG/TEST BUILDS **ONLY**. Never ships.
#[cfg(debug_assertions)]
impl std::fmt::Display for YourProjectError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let text = match self {
            YourProjectError::Fs32tFromStrInputTooLong => "FS32T-101 from_str: input too long",
            // ... one arm per variant ...
        };
        write!(f, "{}", text)
    }
}
```

Implicit in the power-of-10 goals for production-release software is an interesting unique-per-project design question: for software that never stops, halts, panics, etc., how is 'error' information reported? If there is a UI or terminal display, the information may be printed to terminal (without heap) or shown in an info-bar. Or the error-data may be logged or saved to a database. Also, information about the error or 'case-handled' should be as terse as possible, this to manage memory (and avoid heap) but also to prevent any exfiltration of potentially sensitive, private, or security-related data. 

Production-Release binaries can therefore safely contain no error strings. 
A production-release-build log or an info-bar shown to the user can show the error-code-number (e.g. `101`). The number can be looked-up in the Error-Code-Table. 





#### Render the number without heap:

```rust
/// u16 → decimal ASCII in a stack buffer. No heap. Panic-free.
/// Written without indexing, plain arithmetic, or `as` casts,
/// so it passes the lints in "Enforcing the Bans with Lints".
fn u16_to_decimal(value: u16, buf: &mut [u8; 5]) -> &str {
    let mut remaining = value;
    let mut start: usize = buf.len(); // 5: a u16 has at most 5 decimal digits

    // Bounded: `start` decreases by 1 each pass; the loop exits when
    // remaining == 0, or returns "?" if start would go below 0.
    loop {
        // checked_rem / checked_div cannot fail here (divisor is the constant 10).
        let digit = remaining.checked_rem(10).unwrap_or(0);
        remaining = remaining.checked_div(10).unwrap_or(0);

        start = match start.checked_sub(1) {
            Some(next) => next,
            None => return "?",
        };
        match buf.get_mut(start) {
            Some(slot) => *slot = b'0'.wrapping_add(u8::try_from(digit).unwrap_or(0)),
            None => return "?",
        }

        if remaining == 0 {
            break;
        }
    }

    match buf.get(start..) {
        Some(bytes) => std::str::from_utf8(bytes).unwrap_or("?"),
        None => "?",
    }
}
```


### Default Error-Code Rendering responsibility: The boundary renders. Functions return the error-code but do not render. 

Error-producing functions return the error-code, they never print, never log, never touch an 'info bar' in production-release code. Converting a code to human-facing output (u16_to_decimal to 'info bar,' or the log writer) happens at exactly one place per output surface: the top-level handler that owns state, and the logging facility. Rationale: one rendering site per surface means one place to audit for PII and one place to change formatting.

Note: The default case/error-as-code is designed to be most minimal and safest, but that does not prevent production-release-mode uses of that code. It is entirely possible to have a safe-messaging system that would log or print the error-code and a safe explanatory string. E.g. "Contact tech support" or "Irregular input from Device-E detected" or "Check battery" or "Power supply issue." It is safest to leave the error-code as is, but with care it is possible to be both memory thrifty-enough (even no-heap) and safe-enough. Many messages may be useful to send out and be deemed not a security risk. 

Still when reporting a per-code message (in production-release mode), it is recommended that this is done by another function that would be part of how that 'case' is smoothly handled in production, not by the function that returns the error-result-code.

## 3. Retry-or-Permanent Is Defined Per Error-Code (not number-ranges)

Transience (whether an error is retryable) is a property of the error code, 
defined using a single explicit method (for a single source of truth). 
Behavior is defined in one place, making it easy to audit and update.

The is_retryable() method is required to exhaustively list which errors are 
"transient" (able to be retried). 
- Avoids silent misclassification.
- Requires developers to decide for each error whether it is retryable.


```rust
impl YourProjectError {
    /// Transient failure worth retrying?
    /// EXPLICIT list. Deliberately NOT inferred from number ranges:
    /// range rules ("codes < 50 are retryable") silently mis-classify
    /// appended codes and nothing fails to compile. Numbers are identity
    /// only; behavior lives here, in readable code.
    pub fn is_retryable(self) -> bool {
        match self {
            YourProjectError::IoInterrupted
            | YourProjectError::IoWouldBlock
            | YourProjectError::IoTimedOut => true,

            YourProjectError::IoNotFound
            | YourProjectError::IoPermissionDenied
            | YourProjectError::IoOther
            | YourProjectError::Fs32tFromStrInputTooLong
            /* | ...remaining variants... */ => false,
            // NO wildcard -> adding a variant is
            // a compile error until classified
        }
    }
}
```

Retry loops branch on the output of is_retryable() (bounded loop, production-release catch for bad arguments):

```rust
/// Runs `operation` up to `max_attempts` times.
/// Retries only errors where `is_retryable()` is true.
/// Returns the last error when attempts are exhausted.
pub fn run_with_retry<T, F>(max_attempts: u32, mut operation: F) -> Result<T>
where
    F: FnMut() -> Result<T>,
{
    // --- Check: max_attempts >= 1 ---
    // input validation — a request for zero attempts is contradictory.
    // Without this check, max_attempts == 0 would still call operation()
    // once (on failure, `1 >= 0` would then end the loop).
    if max_attempts == 0 {
        // Do not hard code a 2nd source of truth for the code.
        #[cfg(debug_assertions)]
        eprintln!("RETRY-{}: run_with_retry: max_attempts == 0", YourProjectError::RetryMaxAttemptsZero.code());
        return Err(YourProjectError::RetryMaxAttemptsZero);
    }

    let mut attempt: u32 = 1;
    loop {
        match operation() {
            Ok(v) => return Ok(v),
            Err(e) if attempt >= max_attempts => return Err(e), // exhausted: return last error
            Err(e) if e.is_retryable() => {
                /* sleep / backoff */
                // Cannot overflow here (attempt < max_attempts <= u32::MAX),
                // but checked_add keeps the no-plain-arithmetic rule uniform.
                attempt = match attempt.checked_add(1) {
                    Some(next) => next,
                    None => return Err(e),
                };
            }
            Err(e) => return Err(e),                            // permanent: fail fast
        }
    }
}
```

#### Arbitrary examples of Corresponding additions to the enum's io block:
(Append new numbers, do not renumber.)

```rust
    // io categories (mapped from io::ErrorKind at failure sites)
    IoInterrupted      = 1,   // retryable
    IoWouldBlock       = 2,   // retryable
    IoTimedOut         = 3,   // retryable
    IoConnectionReset  = 4,   // retryable
    IoConnectionAborted = 5,  // retryable
    IoBrokenPipe       = 6,   // retryable

    IoNotFound         = 50,
    IoPermissionDenied = 51,
    IoAlreadyExists    = 52,
    IoInvalidInput     = 53,
    IoInvalidData      = 54,
    IoUnexpectedEof    = 55,
    IoOther            = 99,   // explicit policy bucket: non-retryable by rule
```

Matching arms in `is_retryable`:
(no-wildcard `match` forces these to be classified)

```rust
    pub fn is_retryable(self) -> bool {
        match self {
            YourProjectError::IoInterrupted
            | YourProjectError::IoWouldBlock
            | YourProjectError::IoTimedOut
            | YourProjectError::IoConnectionReset
            | YourProjectError::IoConnectionAborted
            | YourProjectError::IoBrokenPipe => true,

            YourProjectError::IoNotFound
            | YourProjectError::IoPermissionDenied
            | YourProjectError::IoAlreadyExists
            | YourProjectError::IoInvalidInput
            | YourProjectError::IoInvalidData
            | YourProjectError::IoUnexpectedEof
            | YourProjectError::IoOther
            /* | ...remaining non-io variants... */ => false,
            // NO wildcard -> adding a variant is a compile error until classified.
        }
    }
```

### The io boundary rule

`io::Error` is converted to a terse code at the failure site. 
`io::Error` (and any path/PII text inside it) is DROPPED at the failure site. 
Only `ErrorKind` is kept, as a discriminant.

Two rules for mapping:

1. Retry-relevant kinds get distinct codes. IoOther is non-retryable.

2. `IoOther` is an explicit policy bucket, not a catch-all
   `IoOther` means "no distinct code assigned, and **non-retryable**." 
   Mapping a kind to `IoOther` = no retrying. 
   If sometime should be retried, it needs its own code.

`io::ErrorKind` is `#[non_exhaustive]`, so the wildcard arm (`_ => IoOther`) is
**required** by the compiler — new kinds can appear in future std versions.
This means you cannot rely on the compiler to force classification of a *newly
added std kind* (unlike your own enum, where the no-wildcard `match` in
`is_retryable` does force it). E.g. Re-audit mapping when updating Rust version.

```rust
impl YourProjectError {
    /// io::Error → terse code. The io::Error (and any path/PII text inside
    /// it) is DROPPED here; only the kind survives as a discriminant.
    ///
    /// Classification note: kinds mapped to `IoOther` are treated as
    /// NON-RETRYABLE by policy (see `is_retryable`). Kinds that should be
    /// retried MUST get their own code, not `IoOther`.
    pub fn from_io_kind(kind: io::ErrorKind) -> YourProjectError {
        match kind {
            // --- transient / retryable: distinct codes required ---
            io::ErrorKind::Interrupted       => YourProjectError::IoInterrupted,
            io::ErrorKind::WouldBlock        => YourProjectError::IoWouldBlock,
            io::ErrorKind::TimedOut          => YourProjectError::IoTimedOut,
            io::ErrorKind::ConnectionReset   => YourProjectError::IoConnectionReset,
            io::ErrorKind::ConnectionAborted => YourProjectError::IoConnectionAborted,
            io::ErrorKind::BrokenPipe        => YourProjectError::IoBrokenPipe,

            // --- permanent: distinct codes for common cases ---
            io::ErrorKind::NotFound          => YourProjectError::IoNotFound,
            io::ErrorKind::PermissionDenied  => YourProjectError::IoPermissionDenied,
            io::ErrorKind::AlreadyExists     => YourProjectError::IoAlreadyExists,
            io::ErrorKind::InvalidInput      => YourProjectError::IoInvalidInput,
            io::ErrorKind::InvalidData       => YourProjectError::IoInvalidData,
            io::ErrorKind::UnexpectedEof     => YourProjectError::IoUnexpectedEof,

            // --- explicit policy bucket: NO distinct code, NON-RETRYABLE ---
            // Required wildcard (io::ErrorKind is #[non_exhaustive]).
            // Everything here is a deliberate "treat as permanent" decision.
            _                                => YourProjectError::IoOther,
        }
    }
}
```


### Notes:
- Retry classifications above are arbitrary examples, not framework-rules
- `io::ErrorKind` is `#[non_exhaustive]`,  it needs the human re-audit


#### More Disambiguation: "Error Site" & "Catch"
Naming things is hard, so "error site" frequently has one 
(hopefully just one) of two different meanings:
1. The file, function, or line of code where an issue, failure or bug occurred.
2. The place in the code where an issue is detected (and responded-to / handled etc.). 


A detection point, as meant here, is the place in a function where a case is detected and handled, for example where the code decides "This is wrong, so we should return error code {N}." 

The commonly used term "Catch" is sometimes roughly equivalent in spirit to how the term 'detect' is used here, but details vary.

JS/TS, Java, and c# use the common 'Try / Catch / finally' system.
cpp and php use only 'Try / Catch'

Zig uses the terms 'try' and 'catch' but the meanings are different:
- zig try: immediately propagate any error up the stack (return that error)
```zig
const value = try mightFail(false);
```

- zig catch can work in a few different ways:
1. "catch" an error to replace it with a specified default value.
If the function-call works, great! 
If the function call fails, then we should do this other thing instead.
```zig
const a: u32 = addTwenty(42) catch 22;␤
```
2. "catch" the error to run a block of code when that happens.
const x = mightFail(true) catch |err| {
// run this code block if it fails
}

As with the word 'assert,' it is important to note collisions between one or more jargon and common-use meanings.


## 4. Two Standardized Patterns for Detection 

We will define two patterns, because there are two ways that code 
can detect an issue (such as an 'error' or a 'case to handle'): 

Both patterns cover the three-modes of behavior (test / debug / production-release).
Consistently using these two patterns should help reviews, audits, and readability.

1. The If-Detection-Pattern: 
Use an if statement where you can ask as an 'if' question:
E.g. "is this length ≤ 32?" or "is this denominator zero?"


2. The Match-A-Function-Call-Pattern
Use a 'match' for functions that can fail (and, coding defensively, 
all functions in this system should return a result).
E.g.  Some function you called told you that it failed. 
You called a function such as from_utf8 or opened a file 
and the check happened inside that function. 
The function you called handed you a 'success-or-failure' result. 


## Communicating Errors "Clearly"

There is context, there are trade-offs, and specific design decisions shape how errors and cases are handled per mode (the whole narrative here). 

### Use of '?'

Case by case: Do not use '?' if doing so would obscure a case, error, or issue (or if an exact site is absolutely needed for a production error (and '?' would obscure that)). It is often best to pass an error along clearly, to raise it clearly, e.g. with '?', e.g. so that the problem that was raised is not obscured (e.g. life in the not-production-appropriate world of python (where none of the modes focused on here are clearly handled well)). 

For example, what are the choices and tradeoffs for production vs. debug mode where a root-source-cause error code is not the same as a site-of-issue code, e.g. a function (with one code (set)) is called in more than one place, and that code is passed on.

E.g. Where the same code may be raised at multiple sites:
- In Production-Release Mode: A production log (or, if more rarely, program exit upon error) will report the shared root code, not which of the N sites raised that code. 
- In Debug-Mode: Debug-prints to log and or terminal (e.g. `eprintln!`) CAN supplement the report by identifying the exact site.
- In Test-Mode you can get a full panic stack trace.

If trace/site-detailed debugging can be done (fittingly) in Debug-Mode, I am guessing that is often a good compromise, e.g. selecting the tradeoff of potentially information preserving brevity for production errors that are not meant to be a full trace.

Possibly important note: This may also be a critical point in arguing for context and project specific design, e.g. for Performant-Flat-ness and against ideological reifications like dogmatic-arbitrary DRY and arbitrary maximalist separation of pedantic 'concerns.' Having the necessary code live natively with a native error-code at each site (so long as that does not confuse or obstruct the compiler or performance-optimization) may (in some cases) be most appropriate. The other side to Casey Muratori's demonstrations that fragmented functions hamstring compilers is that such theoretically "same-ish" functions are (at times) in reality NOT the same when (and exactly how) those (differently implemented) functions get compiled into assembly and machine code, per hardware. One abstraction function used in five places might more appropriately be five unique sites with unique assembly architectures that only share an abstractly taxonomic familiarity with each other (such as a top down view of biology showing that worms and primates are effectively the same single abstract 'concern').

If there is a particular set of tradeoffs where [knowing the specific] site is looking to be critical, that may be an indication that functions need to be flattened and less fragmented. This may (speculatively) be more important for 'hot' regions of code, or (conversely) to safety-critical rare 'cold' regions.

Contrary-wise again, for security sensitive code where performance and debugging are less important than not-exposing potentially security-relevant information about the issue, production messages might be designed to be less site-specific (with other resulting costs being considered worth it). 


#### Common issues obscuring real [causes of] 'errors':
1. ~Remapping / rerouting: e.g. .map_err(|_| SomeOtherCode) replaces the root cause with a not-the-cause code / message / response / handling (try/except is jungle of edge-cases, silent-cases, masked-cases, and missed-cases)
2. Collapse: One generic code (ok_or(ValIndexOutOfBounds)) used at dozens of sites, so a code no longer identifies a site.
3. Re-printing: Diagnostics printed at every level of propagation bury the first/real code/message.


### Choosing production-release error-codes (and use of '?')

Test mode and debug mode can show more testing and debugging feedback (it's in the name), such as panic stack traces, and eprintln diagnostics, debug logs, etc.

Production-release "returns" (or logs) only an error-code, so each project needs to decide what that production-release error-code should reveal or conceal.

#### Common choices:

A. Reuse a function in several places, and return that 'root' function's error-code (e.g. passed up with '?').
   - This error-code says what went wrong and the root-source of detection, but not which call site(s) led there.


B. Flatten: write similar functionality natively at each place it is used, each with its own error-codes.
   - The error-code identifies the exact place.
   - This may go along with writing code that compiles for better performance optimization (not as fragmented).

C. Assign a call-site error-code that deliberately hides the cause (e.g. for security).
   - The caller sees only that this call site failed, not the root cause. This may be done to conceal, or in some cases knowing which caller is having an issue may be more valuable than knowing the root. (E.g. the root cause may be a routine value-clearing or logging function that is called in hundreds of places, and you might need to know which place it failed and where in that caller.)
   - This can be a deliberate choice, unlike the accidental/problematic remapping of error-codes. See: "Common issues obscuring real [causes of] 'errors'".

#### Foreign errors (io::Error, Utf8Error, library errors) and '?'
'?' works on a foreign error only if the project has written a From impl for it (e.g. one that calls from_io_kind).
- With a From impl: '?' works, and every call of that type shares the same error-codes (choice A).
- Without one: '?' does not compile, so each call site must choose its error-code explicitly (choice B or C).
- Some projects deliberately write no 'From' impls for foreign errors, so that `?` on a foreign error fails to compile and every foreign error must be assigned an error-code at its call site.

Choices can differ per function within a project.

Security-sensitive projects may value having fewer distinct error-codes (see above).



 
### The If-Detection-Pattern 
This pattern is for checks that are a condition written as an if statement. 
e.g. ("is X length ≤ 42?", "is Y denominator 0?")

```rust
// --- Check: <the required condition, in words> ---
// <input validation | internal invariant> — <one line: why this check exists>
if !condition {
    // Debug/test diagnostic. May format text (heap OK) because this line
    // is compiled out of production-release builds entirely.
    // Do not hard code a 2nd source of truth for the code.
    #[cfg(debug_assertions)]
    eprintln!("ACRO-{}: fn_name: {}", YourProjectError::AcroFnCondition.code(), value_being_checked);

    // Internal invariants ONLY (delete these lines for input validation):
    // panics in debug builds; compiled out of test and production-release builds.
    // Do not hard code a 2nd source of truth for the code.
    #[cfg(all(debug_assertions, not(test)))]
    debug_assert!(false, "ACRO-{}: fn_name: {}", YourProjectError::AcroFnCondition.code(), value_being_checked);

    // Compiled in EVERY build; the only response that exists in
    // production-release: return the 2-byte code. No print, no heap.
    return Err(YourProjectError::AcroFnCondition);
}

```

Every detection-site gets a cargo test asserting the returned value:

```rust
#[test]
fn acro_fn_rejects_condition_with_code_nnn() {
    let result = the_function(bad_input);
    // .err() avoids requiring Debug on the success type (unwrap_err would).
    assert_eq!(result.err(), Some(YourProjectError::AcroFnCondition));
    // Guards the append-only rule: fails if the code number is ever changed.
    assert_eq!(YourProjectError::AcroFnCondition.code(), NNN);
}
```

Tests for a function's detection sites should live in the same file as the function,
in that file's `#[cfg(test)] mod <filename>_tests` block, so the
Error-Code-Table doc-comment, the detection sites, and their tests are auditable. 

Test names should follow the form
`<acronym>_<fn>_<condition>_with_code_<nnn>`
so that failures can be more easily and clearly traced.


### Testing invariant catch paths (unreachable via the public API)

A production-release catch for an internal invariant is normally unreachable
through public constructors — that is what makes it an invariant. Its
test must construct the invalid state DIRECTLY, which is possible
because tests live in the same file/module and can access
private fields:

```rust
#[test]
fn fs32t_as_str_catches_corrupt_len_with_code_102() {
    // Construct an invalid value directly (same-module private access).
    // This simulates post-construction corruption (e.g. bitflip).
    let corrupt = FixedSize32Timestamp { data: [0u8; 32], len: 99 };
    let err = corrupt.as_str().unwrap_err();
    assert_eq!(err, YourProjectError::Fs32tAsStrLenExceedsBuffer);
    assert_eq!(err.code(), 102);
}
```

Do not add public test-only constructors or pub(crate) fields to enable other tests, that would widen the API surface that the invariant protects.


### The Match-A-Function-Call-Pattern: failure detected via a returned `Result` (not a testable bool)

When the required-condition is checked BY a fallible call (e.g.
`std::str::from_utf8`, a parse, a syscall wrapper), the requirement-check
is the `match` on that call's result (no requirement to test the condition
separately with a duplicate check).

#### Two different match-handling error values:

The Match-A-Function-Call-Pattern involves two different error values of different kinds. This is important for clear handling, for naming and communication, and (if relevant) for heap and memory management.


#### Match-error type 1: The error-code for this system.
This is more clearly no-heap, slim, and predictable. 
In every build-mode, only this 2-byte error-code leaves the function.


#### Match-error type 2: The callee's error, which depends on what function is called. Hopefully it is usually standard a small fixed-size struct the fallible function returns on failure. 
E.g. the Utf8Error is two integers, and stack-only)
```rust
struct Utf8Error {
    valid_up_to: usize,      // the byte position where the input stopped being valid UTF-8
    error_len: Option<u8>,   // length of the invalid byte sequence
}
```
But this is just one clean example, the details are up to each function.
The most memory-use-defensive we can be in production-release code is to '_details' whatever this error is (note: even if memory-use is not a concern, this error-data might contain privacy or even PII related information that should not be exfiltrated in production-release builds).


The callee-function's error value has whatever type that function declared. Many std errors are small stack-only structs, but other std library, and especially 3rd party, errors may own heap allocated inside the callee.

When using any standard-library or third-party function not built 'natively' within a project, heap-use is likely:
- io::Error can sometimes contain an N-length message (using heap)
- String`, `Box<dyn Error>` & `anyhow::Error` can use heap
- Custom library errors with `String`/`Vec` fields can/will use heap.

This framework does not / cannot prevent a dependency from using heap, 
but it can drop what is returned and allocates nothing further to heap. 


All three modes live in the
`Err` arm:

```rust
match fallible_call(input) {
    Ok(value) => Ok(value),   // or continue using `value`
    Err(_detail) => {
        // Verbose diagnostic, heap OK, compiled out of production-release.
        // Runs in debug and test builds; placed first so it prints
        // before the debug-build panic below.
        // Do not hard code a 2nd source of truth for the code.
        #[cfg(debug_assertions)]
        eprintln!("ACRO-{}: fn_name: {}", YourProjectError::AcroFnCondition.code(), _detail);

        // Debug Mode: panic in debug builds ONLY (not test, not production-release).
        // `debug_assert!(false, ...)` is the standard way to panic from
        // inside an Err arm; it compiles out of production-release builds.
        // Do not hard code a 2nd source of truth for the code.
        #[cfg(all(debug_assertions, not(test)))]
        debug_assert!(false, "ACRO-{}: fn_name: {}", YourProjectError::AcroFnCondition.code(), _detail);

        // Production-release Mode: terse code. No heap. No PII. No print.
        Err(YourProjectError::AcroFnCondition)
    }
}
```


**Input-validation version** — identical to the above, but with the `debug_assert!(false, …)` line removed, because a failure on untrusted input is expected, not a bug, for where Debug should not panic:

```rust
// INPUT VALIDATION version — for a call that can legitimately fail on
// untrusted input (e.g. from_utf8 on external bytes). Debug must NOT panic.
match fallible_call(input) {
    Ok(value) => Ok(value),
    Err(_detail) => {
        // NO debug_assert!(false, …) here — this failure is expected, not a bug.

        // Do not hard code a 2nd source of truth for the code.
        #[cfg(debug_assertions)]
        eprintln!("ACRO-{}: fn_name: {}", YourProjectError::AcroFnCondition.code(), _detail);

        // Production-release Mode: terse code. No heap. No PII. No print.
        Err(YourProjectError::AcroFnCondition)
    }
}
```




## 5. Things Production-Release Code Must Not Do

### Banned in Production-Release Error Paths

- `format!`, `to_string`, `String`, `Box<dyn Error>`, any use of heap
- `io::Error` stored inside the project error type.
- `unwrap` / `expect` / `panic!` / body `assert!`.
- Error text containing paths, file contents, env vars, user data, PII,
  or implementation detail. (Production-Release ships no error text at all.)
- Number-range behavior rules ("codes 1–49 mean X").
- Reusing or renumbering an existing error-code.

### Avoiding Production-Release Panic
(Note, this section is likely not 100% exhaustive.)

These operations also panic with no opt-out (so, in production as well), so they should be avoided or handled correctly when used in case-handling:
let x = buf[i];        // panics if i >= buf.len()   — ALL profiles
let s = &data[a..b];   // panics if a > b or b > len — ALL profiles
let q = num / den;     // panics if den == 0, or (signed types) num == MIN and den == -1 — ALL profiles
let r = num % den;     // panics if den == 0, or (signed types) num == MIN and den == -1 — ALL profiles


When indexing, slicing, or dividing in production-release, use non-panicking versions of operations: .get() / .get_mut() instead of [], checked_div / checked_rem instead of / and %, split_at_checked instead of split_at. (These standard-library operations panic in all profiles, importantly each has a non-panicking alternative. This can be easy to miss in a review, because they look like ordinary method calls.)

#### Panic & Non-Panic Alternatives: Other standard-library operations that panic in all profiles, with their non-panicking alternatives

| Panics | Panics-When | Use instead (Non-Panic) |
|---|---|---|
| dst.copy_from_slice(src) | lengths differ | check dst.len() == src.len() first (If-Detection-Pattern) |
| &text[a..b] on a str | a or b is not on a UTF-8 character boundary | text.get(a..b) |
| cell.borrow_mut() on a RefCell | value is already borrowed | cell.try_borrow_mut() |
| duration_a - duration_b | result would be negative | duration_a.checked_sub(duration_b) |
| instant + duration | result overflows | instant.checked_add(duration) |
| iter.step_by(0), slice.chunks(0), slice.windows(0) | argument is 0 | check the argument is nonzero first |
| println! / print! | writing to stdout fails (e.g. closed pipe) | writeln!(std::io::stdout(), ...), which returns a Result |
| std::thread::spawn | the OS cannot create a thread | std::thread::Builder::new().spawn(...), which returns a Result |

Related, not a panic: `as` casts between integer types never panic, but silently truncate (300i32 as u8 is 44). Use u8::try_from(value), which returns a Result.




#### Plain Arithmetic Operators
Rust's plain arithmetic operators (`+`, `-`, `*`, `<<`, `-x` negation, etc.) have build-profile-dependent behavior that is controlled by `overflow-checks` setting:

| Profile | `overflow-checks` Default | Behavior on Overflow |
|---|---|---|
| dev / debug | `true` | **panics** ("attempt to add with overflow") |
| production-release | `false` | **silently wraps** (two's-complement wraparound) | 

With default settings, using plain arithmetic operators:
- In debug build, `255u8 + 1` panics.
- In production-release build, `255u8 + 1` evaluates to `0` and execution continues with a wrong value.

Production-release mode uses checked_add / checked_sub / checked_mul / checked_neg / checked_abs / checked_pow / checked_shl / checked_shr. Where wraparound is intended (e.g. hash or checksum arithmetic), use wrapping_methods (wrapping_add, etc.) to make intent clear.

.abs() and .pow() follow the same profile-dependent rule as operators, e.g. i32::MIN.abs() panics in debug-mode, vs. returning i32::MIN in production-release-mode.

A Note on shifts (<< and >>): shifts do not wrap in a two's-complement way. Bits shifted off the end are discarded without panic in each profile (e.g. 0xFFu8 << 1 is 0xFE). For shifts, overflow-checks control a shift amount greater than or equal to bit width. In debug-mode this panics; in production-release-mode the amount is masked to the bit width. E.g. 
1u8 << 9 is computed as 1u8 << 1, which is 2. checked_shl / checked_shr return None (in such cases).


### Enforcing the Bans with Lints

Some bans can be automatically checked/enforced with Rust-Clippy linter. Turning lints on makes a violation fail a lint run (instead of depending on a manual review to catch the issue).

#### Most bans in this section can be linter-checked:
Crate root files (src/lib.rs or src/main.rs) can be configured to cause these violations to raise an error when running `cargo clippy`.

Place at file top: (src/lib.rs or src/main.rs)
```rust
#![forbid(unsafe_code)] // or permissive #![deny(unsafe_code)]
#![cfg_attr(not(test), deny(
    clippy::unwrap_used,                // .unwrap()
    clippy::expect_used,                // .expect()
    clippy::panic,                      // panic!
    clippy::todo,                       // todo!
    clippy::unimplemented,              // unimplemented!
    clippy::unreachable,                // unreachable!
    clippy::exit,                       // std::process::exit
    clippy::indexing_slicing,           // buf[i], &data[a..b]
    clippy::string_slice,               // &text[a..b] on a str
    clippy::arithmetic_side_effects,    // integer operators that can overflow or panic
    clippy::cast_possible_truncation,   // `as` casts that can truncate
))]
```

Notes:
- Run `cargo clippy` before production-release builds (e.g. in CI). Lints run under `cargo clippy` (not `cargo build`). 
- `not(test)`: Means lints are off in cargo-test-mode (test code can use unwrap, indexing, etc.)
- allowing-unsafe: `deny(unsafe_code)` can be used instead of `forbid(unsafe_code)`: a module that requires unsafe (see Appendix 2, rule 1) can opt out by using `#[allow(unsafe_code)]`, then each such exception is searchable. `forbid` cannot be overridden.
- Not covered by lints: Heap use in error paths, and the "Panic & Non-Panic Alternatives" table above (copy_from_slice, RefCell::borrow_mut, etc.). (These require manual review.)

## Appendix 1: Links & References
- https://en.wikipedia.org/wiki/The_Power_of_10:_Rules_for_Developing_Safety-Critical_Code 
- https://spinroot.com/gerard/pdf/P10.pdf 
- https://spinroot.com/static/index.html 
- https://web.eecs.umich.edu/~imarkov/10rules.pdf 
- https://www.stroustrup.com/JSF-AV-rules.pdf 
- https://www.youtube.com/watch?v=JWKadu0ks20 
- https://en.wikipedia.org/wiki/Static_program_analysis 
- https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ 
- Mike Acton ("Data-Oriented Design and C++," CppCon 2014)


### Appendix 2: Summary of Revised 10 Areas

We can try to make a list of '10 Rust Production Rules' updated for broader general systems programming in 2026 based on NASA's 2006 'Power of 10' rules that were originally narrowly framed for c for embedded-systems. 

These are ideals to be followed where possible and sensible, not as absolute pedantic rules:

1. no unsafe stuff: 
- no recursion  
- no goto 
- no pointers 
- no preprocessor branching
(Term collision note: Technically an 'unsafe code' block in Rust may be required for cases such as naked/assembly code or to interact with a Posix-OS, as in the case of raw-terminals. While use of jargon-'unsafe' blocks should be avoided where possible, the term 'unsafe' may be a jargon-term that does not mean that a specific best-practice rule was violated.)

2. Loops: either firmly bounded or unbounded:
- Upper bound on all normal-loops (to make sure they do **not** keep looping)
- Failsafe for all always-loops to make sure they **do** keep looping (e.g. additional restart layer)

3. Pre-allocate all memory (no dynamic memory allocation)
- Production code should minimize or eliminate use of heap (e.g. very terse error messages that do not leak any user-data)
- Debug and testing often make sense to use heap and this code is not in production-binaries (e.g. detailed error messages)
- Clearly separate lazy-convention from real-need. With tools such as "Buffy'
github.com/lineality/buffy_stack_format_write_module, it is not necessary to use heap for string formatting.

4. Clear Function Scope and Data Ownership: 
Part of having a function be 'focused' means knowing if the function is in scope. Functions should be neither swiss-army-knife functions that do too many things, nor scope-less micro-functions that may be doing something that should not be done. Many functions should have a narrow focus and a short length, but definition of actual-project scope functionality must be explicit. Replacing one long clear in-scope function with 50 scope-agnostic generic sub-functions with no clear way of telling if they are in scope or how they interact (e.g. hidden indirect recursion) is dangerous. Rust's ownership and borrowing rules focus on Data ownership and hidden dependencies, making it even less appropriate to scatter borrowing and ownership over a spray of microfunctions purely for the ideology of turning every sub-operation into a microfunction just for the sake of doing so. (See more in rule 9.)

5. 'Case Handling' per Mode & Defensive Programming: debug-assert, test-assert, prod safely check & handle, not 'assert!' panic in production

6. Manage ownership and borrowing
- Rust is designed to greatly assist here (vs. c).

7. Manage return values: 
- use null-void return values 
- check non-void-null returns
- see above for designing and checking return values to handle cases of invalid other return-value cases.
- always have functions return a 'result' so errors and cases can be handled

8. Manage conditional compilation: Navigate debugging and testing on the one hand and not-dangerous conditional-compilation on the other hand:
- Here 'conditional compilation' is interpreted as significant changes to the overall 'tree' of operation depending on build settings/conditions, such as using different modules and basal functions. E.g. "GDPR compliance mode compilation"
- Any LLVM type compilation or build-flag will modify compilation details, but not the target tree logic of what the software does (arguably). 
- 2025+ "compilation" and "conditions" cannot be simplistically compared with single-architecture 1970 pdp-11-only C or similar embedded device compilation.

9. Communicate: 
- Use doc strings; use comments. 
- Document use-cases, edge-cases, and policies (These are project specific and cannot be telepathed from generic micro-function code. When a Mars satellite failed because one team used SI-metric units and another team did not, that problem could not have been detected by looking at, and auditing, any individual function in isolation without documentation. Breaking a process into innumerable undocumented micro-functions can make scope and policy impossible to track. To paraphrase Jack Welch: "The most dangerous thing in the world is a flawless operation that should never have been done in the first place.")
- Tradeoffs: Decide for each function which error-code the caller should see.
- Rather than having a result hidden in let _ =, allow that result to be shown in debugging

10. Use state-less operations when possible:
- a seemingly invisibly small increase in state often completely destroys projects
- expanding state destroys projects with unmaintainable over-reach


# Appendix 3: Value-Check & Value-Recovery (speculative first draft)

Advanced: using error-catching and error-correcting features to recover from accidental or adversarial value-distortions.

Note: Managing modes and cases is universally assumed scope (however neglected). 'Value-Check & Value-Recovery' are (as of 2026) not considered to be normal scope accounted for in every library / package / program or module. 

1. redundant-checked values (less likely if both copies altered)
2. Hamming code value formats
3. checksum/hash within custom-type struct to confirm that the value was not inadvertently changed as a standalone check-method or built into any use (e.g. update to value) that value-struct.

### Triple-Lock: Value, Hash, ECC

(The term "Triple-Lock" is a nickname or ~mnemonic borrowed from
the UK's pension-rate's self-correction mechanism. The specific-mechanisms are not meant to be mechanically-similar (or even remotely relevant). 
The UK Triple-Lock is also an example of technical decision coordination being difficult
(a.k.a. a serial career-ending political quagmire train wreck). 


The basic idea of the "Triple-Lock" is here meant to refer to a triumvirate (or triumvirātus) relationship, not intended to mean that implementations always need to have exactly 3X parts for every value (see examplifications below).

#### Parts: Value, ECC, Hash
1. 'The Value' (most likely a rust default data-type inside a struct/imp/enum construct; e.g. type:u8, value:42

2. An error-correcting-code version of this value (most likely specific to only this value)

3. A hash/check-sum/check-digit derived from one or more values (as in a value in a struct) (and potentially ECC-values). 

(4. Value scrubbing, defaults, etc. This is perhaps only more distantly connected.)


#### There are a number of tradeoffs and balances here:
- You want to have a single source of truth for a value.
- You want to have a safety-backup of a value.
- You want to be able to recover a 'lost' value.
- You want to be able to validate a value.
- You want to be able to use the value in a normal/expected format (e.g. you cannot interchange specific typed-values in Rust with error-correcting-code representations of those values).
- (And you want basic memory-maintainance sanity underlying that value) (e.g. not starting/initializing with an already stale value that you expect to be something else (e.g. zero/false), and not leaving behind a stale or leaked value that will confuse or leak data to someone else)



#### Categories of Hashes

1. a hash of a program's immutable state (should last lifetime of runtime)

2. hash of const-state values (~all 'constants') (should last lifetime of runtime)

2. mutable state hash

3. a hash of a single struct's state


#### Situations:
1. Values can be [automatically] recovered/re-derived if corrupted (a kind of partial inner-reboot that does not require stopping the program (or the satellite falling out of the sky)

2. Some sort of manual/intervention/action-from-outside-the-program is needed to restore trusted state. (E.g. the name of the user was lost, as the program has no way to 'recover' that automatically, but it could (A. log the error, and B.) re-ask for that datum.

3. 


periodic checks (time, process)

one hash and one simple check

could be a lite-wait xor checksum or a Pearson-hash

could be a not-expensive addition to a code-base

e.g. 


### Scenario Comparison for Cost-Sanity:

1. heavy checking

For every value stored anywhere, have N duplicate versions of that and constant checks.

Most people's inner-troll probably immediately starts with a rub-goldberg
version of this as their paradigmatic satire-farce straw-man of value checking. 

We could invent a deliberately comically inefficient version of this, such as where every value has 10 duplicate backup copies and every one of those has checksums and error correcting codes, and all of those also have check-sums with each-other, and everything is constantly checked in the least optimal way in every hot loop. But such a nightmare is likely more analogous to ideological DRY/'concern-separation' where fragmentation is pedantically multiplied with no contextual (or any other kind of) justification. 

A. Most systems kind of work (sometimes) without value-checking.
B. All value-checking does not need to be heavy-checking.
C. Heavy checking is probably not needed in most cases.
D. Contextually-designed heavy-checking may not need to be egregiously inefficient. 

While an ECC is most likely value-specific, hashes can be more flexible.
You might only need one single hash (not one per value). 

There are also many details around recovery and data-sources. Maybe all you need is one-hash and you can restart the system if there is an issue (that would be a tiny price to pay). Or maybe only a few key values are suitable for ECC, while other values may be disposable or recoverable in other ways. 



Having two/three values instead of one may use 'more memory', but, is this being pennywise, pound foolish? (Are you fretting over 90 vs. 30 state-values and also planning to automatically import serde, and rand, and chrono, serd_jason, and toml, and tokyo, and walkdir, ... and what happened, I only went to the store to buy milk?)

A. Having 60 u-size values instead of 20 u-size state values is most-likely imperceptibly small in the overall memory footprint of the program. 

B. Most programs apathetically use memory-type allocations that are larger than they need to be: e.g. const u64: 42

C. Most programs are sprawling mazes of nested objects and structs and functions with belligerent antipathy towards memory and performance. THAT is the problem. In most cases a memory-efficient, computationally-optimized, and value-secured program would be much smaller and faster than a 2025-status-quo super-sized spaghetti-milkshake-sandwich. 





For a slow safety-critical process, some version of heavy-checking might be fine, but for frivolous programs that are only wanted to be fast, the cost is assumed to be undue. 



2. light check: 1 hash, 1 check, rare

1. a 'lifetime-static' hash check of immutable initialized state and constants (any values expected to not change during run-time).

2. 


E.g. It is probably not unrealistic to imagine a program with one fat-struct containing all or most of the program's state, and where that state is a mix of mutable and immutable values, and where mutable values (on average) change on a scale of hours, not milliseconds or seconds. 

In such a case, how much of a check-burden would it be to have one XOR checksum (or Pearson-Hash) of that state and to check that hash when a given mutable value is updated? This is probably an acceptably small cost (even for a small program, let alone a standard spaghetti bloat fest application). 

If that failed, you could check each value and ECC (checking a few dozen state values would most likely be un-noticable to the user), and the value could be recovered from the ECC (which would very rarely, if ever, happen). This is arguably a 'heavy' value-check (of all values + ecc) yet the cost is probably negligible. (Note: if an immutable struct value was corrupted, there may need to be a struct rebuild or a partial reboot, but given the rareness of memory-corruption this is probably feasible and often would not even be noticed by the user (given that most applications have random 5-sec delays for no known reason).)



### HIX: The Heat-Index Dimension of the problem-space
See: https://github.com/lineality/hix_heatmap_indices_for_compute 

Another factor for planning (for example for weighing the cost of value-checking) is where in the code a check might happen: namely, does checking occur N times in a hot-loop (or hot region of code) (where even a moderate cost could be hugely amplified), or does checking occur infrequently in a 'cold' area of code, where even a heavier check would not interfere with usability or overall resource management?

A heat-index type analysis of code may be an important part of designing an appropriate value-check system (to keep cost low).


Note: In theory this might be part of an approach for memory-safety and logic-safety, checking that 'values' (or memory regions) are not being changed when they are assumed to not be changed.



### Appendix 4: Advanced: Erasure of sensitive values as part of handling
- TODO, pending


#### Appendix 5: 
- example of 2-3 functions + error structures
- maybe unified-example instead of only abstract separate examples (e.g. an extremely simple crate (hello-world-ish) that demonstrates everything here)


# Also See:

- MISRA C / MISRA C++ automotive and industrial coding rules
- CERT C / CERT C++ Carnegie Mellon's secure-coding standards
- JSF AV C++ 2005
- JPL Institutional Coding Standard for C (2009): NASA JPL's standard (builds on 'the Power of 10')
- AUTOSAR C++14 guidelines: automotive
- DO-178C, ISO 26262, IEC 61508: Process standards for aviation, automotive, and industrial software (require that coding rules exist but do not supply any?)

#### Rust-specific:
- Ferrocene (Rust toolchain qualified under ISO 26262 and IEC 61508)
- Rust Foundation's Safety-Critical Rust Consortium (from 2024, developing Rust coding guidelines)
- Clippy's "restriction" lint group


