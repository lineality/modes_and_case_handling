#### mode_case_handling_framework_summary



# Mode & Case Handling Bullet Points Summary
## Framework: Three Modes & Rules

### Three Modes of Operation:
1. Production-Release Mode: 
- Never panics, halts, or leaks data.
- Uses 2-byte error codes (no heap, no strings, no PII).
- Smoothly handles all cases via "Let It Fail & Recover" (with optional logging).
- Uses a Three-Level Recovery Hierarchy 

2. Debug Mode:
- Similar to production but logs verbose diagnostics (heap allowed).
- Uses `eprintln!` for errors and `debug_assert!` for internal invariants (gated to avoid running in test/production).
- Optional debug printing/logging for inspection.

3. Test Mode:
- Uses `assert!` in test functions.
- Deliberately crashes/panics to generate stack traces for debugging.



### Rules:

1. All functions return `Result<T, YourProjectError>`. The error payload is always the 2-byte `YourProjectError` enum. All operations are expected to eventually fail in some way; all such failures must be smoothly handled.

2. Production-Release Mode (Result-Error case-handling):
- never panics
- no heap in error-code-return from functions
- no print or log of that code by the function before the code is returned

3. Separate Debug-Mode Assertions from Cargo-Tests:
- `debug_assert!` must be gated with `#[cfg(all(debug_assertions, not(test)))]` to exclude from test mode and production mode builds.

4. Use Gated Verbose Debug-Mode Diagnostics:
- Use `eprintln!` (gated with `#[cfg(debug_assertions)]`) for debug-only output. Heap is allowed/needed here.

5. Test-Mode Isolation:
- Test code must be gated with `#[cfg(test)]`. 
- Avoid using test-mode tests inside functions for stable code.

6. Use a Fieldless Enum error-code system.

7. Use Enforced-Custom-Types for Value-Integrity.

8. Use a Three-Level Recovery Hierarchy for recovery and state.


### Enforced-Custom-Types & Value-Integrity:
- Use `struct`/`enum`/`impl` to enforce value boundaries for inputs and intermediate values. This ensures invalid states (including bit-flips, corruption) are caught and handled as errors, not passed silently.
- required: add a validity-recheck into the .get() method
- Optionally use a .validity_recheck() method
- While a custom type in Rust can enforce and validate that a value is within the definition boundaries, this check only happens once when the constructor is run ( .new() ), assuming that the constructor was correctly written to carry out that check correctly. If memory-corruption happens after that initial check there are no automatic checks that will catch that the value is now invalid (e.g. as the value is returned from and accepted by another function). Additional .validity_recheck() methods can be made and used to manually check at specific points.
- Using custom types helps to manage compatibility within the design (e.g. narrowing scope of function inputs)
- vanilla custom types (without additional validity-checks) guard against design errors (coding-mistakes) and 'expected' errors.
- additional validity-rechecks can add guards against some electrical, hardware, and adversarial based errors, or 'unexpected errors.'
- Getting and mutation/updating use methods that must be manually designed to check & enforce boundary checks and validity.
#### Public & Private
- note: the individual rules for structs, enums, impls, and their combinations, vary.
- A struct CAN be safely pub (so other modules can use the type name in signatures).
- The struct's inner fields should NOT be pub (so other modules cannot bypass checks).
- Example of safety for combinations: if an enum variant carries data then do not put raw unbounded primitives inside that; rather, put a bounded struct inside.

### Three-Level Recovery Hierarchy ("Let It Fail & Recover")
Production-release failures should fit one of three bounded recovery tiers:
- Recovery Tier 1: Micro-Retry (Local) — If err.is_retryable() is true, a caller repeats the bounded operation (with backoff/sleep).
- Recovery Tier 2: Step Fallback / Safe Degradation (Subsystem) — This is at a level above which any retry-errors would be thrown. If non-retryable, or retry attempts are exhausted, the subsystem safely aborts the current command, and handles what to do next. This is highly case dependent, and might include trying another function, reverting to a previous state, exiting silently, etc. Some internal state may need to be reset. Ultimately move on with or without logging the case or "error."
- Recovery Tier 3: Macro Re-initialization — An outer loop reinitializes state and continues execution without halting. The largest case for Tier 3 reboots the entire program.

#### Recovery & State-Recovery:
Recovery tiers represent functional layers of 'state' in terms of 'recovery levels' to plan for, e.g. what to do when some state-values may not exist. Tier 1 covers the state in question. Tiers 2 and 3 range from small to large 'reboot/retry' scales up to the whole program. Full system restart will likely often have at least some state that it can be restarted with (resulting in no noticeable interruption), sometimes implying that state should be managed outside of retry-loops, to retain intact values.


### Error Handling System
- Single Fieldless Enum:
  A global `YourProjectError` enum (e.g., `#[repr(u16)]`) defines all error codes.
  Properties:
  - No heap (2-byte `Copy` values).
  - No `String` or `io::Error` payloads (prevents PII leaks).
  - Exhaustive `match` for retryability (`is_retryable()` method).
  - Append-only codes: Never reuse or renumber.

- Error Code Table Rules:
  1. Codes are unique and permanent (e.g., `Fs32tFromStrInputTooLong = 101`).
  2. Reserve blocks per module/feature (e.g., 100–199 for `FixedSize32Timestamp`).
  3. Variant names: `AcronymFunctionCondition` (e.g., `IoPermissionDenied`).
  4. Document codes in the enum’s doc comment.

- Display for Debug Only:
  `Display` impl (for human-readable text) is compiled only for debug/test builds.

#### Error Sites & Propagation:
- Unique codes permit use of '?' because the unique code of the root-cause error is preserved (not obscured) by propagating the error.
- If an error site is of note, that site needs to throw a specific error code.
- This acts as a proxy for manually checking for most internal-invariant type issues.


### Two Detection Patterns
1. If-Detection-Pattern:
   For direct condition checks (e.g., `if len > 32`).
   - Debug: `debug_assert!` (internal invariants only) + `eprintln!`.
   - Production: Return terse error code.

2. Match-A-Function-Call-Pattern:
   For handling `Result` from fallible calls (e.g., `std::str::from_utf8`).
   - Debug: `debug_assert!(false, ...)` (for internal invariants) + `eprintln!`.
   - Production: Drop callee’s error (to avoid heap/PII), return project error code.


## Key Principles
- No Heap in Production: Ban `String`, `format!`, `Box<dyn Error>`, etc.
- No Panics in Production: Use `checked_add`, `.get()`, etc., to avoid silent wraps/panics.
- Defensive Programming:
  - Input Validation: Expected issues (e.g., bad user input) → return error code + debug `eprintln!`.
  - Internal Invariant: "Should-not-happen" checks (e.g., bit-flips) → `debug_assert!` + production catch.
- Retry Logic: Defined per error code (not ranges) in `is_retryable()`.


### Example Code
#### Error Enum
```rust
#[repr(u16)]
pub enum YourProjectError {
    IoNotFound = 50,
    Fs32tFromStrInputTooLong = 101,
    RetryMaxAttemptsZero = 200,
}
```

#### If-Detection-Pattern
```rust
if !condition {
    #[cfg(debug_assertions)]
    eprintln!("ACRO-101: detail: {}", value);
    return Err(YourProjectError::AcroFnCondition);
}
```

#### Match-A-Function-Call-Pattern
```rust
match fallible_call(input) {
    Ok(v) => Ok(v),
    Err(_detail) => {
        #[cfg(debug_assertions)]
        eprintln!("ACRO-101: {}", _detail);
        Err(YourProjectError::AcroFnCondition)
    }
}
```

### Banned in Production
- `unwrap`/`expect`/`panic!`
- Heap allocations (`String`, `format!`, `Box<dyn Error>`)
- Error messages with PII (file/dir paths, user data, etc.)
- Unchecked Arithmetic: Arithmetic operators that can panic in production (use `checked_add`, etc.)


#### Suggestions (TODO: under construction; move to rules, for manageable use of codes?)
- Allow effective "blocks" of codes for functions, to allow coherent numbering and easy incrementing in case two developers collide. e.g. if the last two digits are internal to a function, u16 would allow for 654 functions to each have 99 internal errors (or depending on average function size, ten per function may be enough on average).
- alt phrasing: allow 10 or 100 error-codes per function to manage allocations and changes to codes

