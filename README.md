# Contribution 1: Add `overlay` String Expression to Daft

**Contribution Number:** 1  
**Student:** capyBearista  
**Issue:** https://github.com/Eventual-Inc/Daft/issues/3792  
**Status:** Phase III Complete

---

## Why I Chose This Issue

Daft is an open-source distributed DataFrame library written in Rust with Python bindings. Issue #3792 tracks a list of string functions Daft is missing compared to PySpark, and contributors take them one at a time. `overlay` is one of those unchecked functions and no one else had claimed it in the issue thread when I commented.

`overlay(src, replace, pos[, len])` writes the string `replace` over `src` starting at position `pos`, replacing `len` characters. It is a small, well-defined string operation with clear behavior specified by PySpark, which makes it a good first contribution: bounded scope, single function, and a clear definition of "done."

I wanted something concrete and reviewable rather than sprawling. The repo also has recent merged examples of similarly sized string functions (`strip`, `concat_ws`), which gives me a proven pattern to follow.

**Issue comment:** https://github.com/Eventual-Inc/Daft/issues/3792#issuecomment-4786037852

---

## Understanding the Issue

### Problem Description

Daft does not yet implement the `overlay` string function. `overlay(src, replace, pos[, len])` overlays the string `replace` onto `src` starting at character position `pos` (1-indexed, matching PySpark), replacing `len` characters of `src`. If `len` is omitted, it defaults to the length of `replace`. Example: `overlay("AAAAAAAAAA", "BBB", 3)` produces `"AABBBAAAAA"`.

### Expected Behavior

A user should be able to call `daft.functions.overlay(col("src"), col("replace"), pos, len?)` and get back a string column with `replace` written over `src` at `pos`.

### Current Behavior

The function does not exist. Importing it fails with `ImportError: cannot import name 'overlay' from 'daft.functions'`. Asking Daft's internal function registry for a function named `"overlay"` triggers a Rust panic with the message `Function was missing an implementation`, because no one has registered that name.

### Affected Components

The fix is mostly in the Rust side of the codebase, in the directory that holds Daft's string functions (`src/daft-functions-utf8/`), plus a small Python wrapper (`daft/functions/str.py` and its `__init__.py`) and a new test file. The existing `replace` function in the same Rust directory is a good template because it also takes several string arguments.

---

## Reproduction Process

### Environment Setup

Linux x86_64. I already had `uv` (Python package manager), `rustup` (Rust toolchain manager), Python 3.14, and GNU Make installed.

The Daft repo pins a specific nightly Rust compiler through a `rust-toolchain.toml` file, which `rustup` installs automatically the first time you build. Setup was three commands:

1. `make .venv` to create the Python virtual environment and install all Python dependencies.
2. `make build` to compile the Rust code and install Daft into the venv. This took about 5.5 minutes.
3. A sanity check: I ran an existing string-function test (`test_concat_ws::test_basic_separator`) and it passed, confirming the environment works.

No errors during setup.

### Steps to Reproduce

1. From the Daft directory, activate the virtualenv: `source .venv/bin/activate`
2. Try to import the function:

   ```python
   from daft.functions import overlay
   ```
3. **Expected (since it is not implemented):** `ImportError: cannot import name 'overlay' from 'daft.functions'`
4. **Actual:** `ImportError: cannot import name 'overlay' from 'daft.functions'`

   As a second check, ask Daft's internal function registry directly:

   ```python
   col("src")._call_builtin_scalar_fn("overlay", lit("BBB"), 3)
   ```
   **Actual:** The Rust side panics with `Function was missing an implementation`, because the registry has no entry named `"overlay"`.

5. I also searched the whole codebase for any Rust code that defines or names a function `"overlay"` and found nothing, so the function is genuinely absent rather than just unexposed.

### Reproduction Evidence

- **Branch link:** https://github.com/capyBearista/Daft/tree/feat-overlay-3792
- **Commit showing reproduction:** https://github.com/capyBearista/Daft/commit/1e5e717e8679c2201dfb3ea79afe2d7599851812 (`repro_overlay.py` tries to import `overlay` and crashes with `ImportError`)
- **Build log tail:** `Finished 'dev' profile [unoptimized + debuginfo] target(s) in 5m 32s` / `EXIT=0`
- **My findings:** The function is not registered anywhere. The Python import fails with `ImportError`. I also tried calling Daft's internal function registry directly for the name `"overlay"`; this triggers a Rust panic (`Function was missing an implementation`) that Python surfaces as a `pyo3_runtime.PanicException`, which is not caught by an ordinary `except Exception`, so I kept the reproduction script to just the import check for simplicity.

---

## Solution Approach

### Analysis

Daft exposes every string function through a registry of named functions. Each string function is a small Rust struct that implements a fixed interface (a name, the actual computation, type checks, and a docstring) and then registers itself by name. Python looks up functions by that name. For `overlay`, no struct exists and no name `"overlay"` is registered, so both the Python import and the Rust dispatch fail. The fix is to add the missing function following the same template the existing string functions use.

### Proposed Solution

Add a new `overlay` function that takes a source string, a replacement string, a starting position, and an optional length, and returns a new string with the replacement written in. Match PySpark's 1-indexed position and default the length to the length of the replacement when omitted. Handle edge cases explicitly: out-of-range positions, null inputs, and the case where the overlay runs past the end of the source. Then expose it through the Python wrapper and add tests.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `overlay` is missing from Daft's PySpark-parity string function set. There is no implementation, no registration, and no Python binding. The fix adds the function end to end.

**Match:** The existing `replace` function (in `src/daft-functions-utf8/src/replace.rs`) is the closest template. It is a string function that takes several arguments, the same shape `overlay` needs. The merged `strip` and `concat_ws` functions show the Python-side wrapper and test conventions.

**Plan:**
1. Write the Rust implementation in a new file `src/daft-functions-utf8/src/overlay.rs`, following `replace.rs` as a template. The function takes `src`, `replace`, `pos`, and optional `len`, and computes the overlay per row.
2. Register the new function in `src/daft-functions-utf8/src/lib.rs` so the registry knows about it.
3. Rebuild with `make build` so the new function is compiled in.
4. Add the Python wrapper `overlay(...)` in `daft/functions/str.py`, with a docstring and example matching the style of the existing `replace` wrapper.
5. Re-export `overlay` from `daft/functions/__init__.py` so users can do `from daft.functions import overlay`.
6. (Optional) Add an `overlay` method on the `Expression` class in `daft/expressions/expressions.py` so `col("src").overlay(...)` chaining works.
7. Add `tests/expressions/test_overlay.py` covering: basic overlay, omitted length, length shorter than the replacement, position past the end of the source, null inputs, and a broadcasted scalar replacement.

**Implement:** https://github.com/capyBearista/Daft/tree/feat-overlay-3792 (commits to land in Phase III)

**Review:** Before opening the PR, re-read `CONTRIBUTING.md` and follow its conventions: Conventional Commits title (`feat(utf8): add overlay string function`), small single-purpose PR, disclose any AI-assisted scaffolding in the PR description, and run `make doctests` so the docstring example renders correctly.

**Evaluate:** Run the new tests (`pytest tests/expressions/test_overlay.py`) and the Rust unit tests for the utf8 module. Then run the existing string-function test suite to confirm nothing else broke.

---

## Testing Strategy

### Unit Tests

- [x] Basic overlay, default `len` (`overlay("AAAAAAAAAA", "BBB", 3)` -> `"AABBBAAAAA"`)
- [x] Explicit `len` matching `len(replace)`
- [x] `len` smaller than `replace` (replace grows the result)
- [x] `len` larger than `replace` (replace shrinks the result)
- [x] `pos` at start (`pos=1`)
- [x] `pos` at end
- [x] `pos` past the end of `src` (clamped, `replace` is appended)
- [x] `pos=0` clamped to `1`
- [x] Empty `replace`
- [x] Empty `src`
- [x] Multiple rows in a single column
- [x] Null `pos` propagates a null result row
- [x] Non-string `src` raises

These are the Python tests in `tests/expressions/test_overlay.py` (13 tests, all passing). The Rust unit tests in `src/daft-functions-utf8/src/overlay.rs` cover the same `overlay_str` kernel directly (8 tests, all passing).

### Integration Tests

- [x] Rust unit tests (`cargo test -p daft-functions-utf8 --no-default-features --lib overlay`): 8 of 8 pass.
- [x] Python tests (`DAFT_RUNNER=native .venv/bin/pytest tests/expressions/test_overlay.py`): 13 of 13 pass.
- [x] Regression sweep on the existing utf8 suite (`DAFT_RUNNER=native .venv/bin/pytest tests/expressions/test_utf8.py tests/recordbatch/utf8`): 116 of 116 pass, no regressions.

### Manual Testing

- Built the Rust extension (`make build`, ~14s incremental) and imported `from daft.functions import overlay` (the import that raised `ImportError` in Phase II is now successful).
- Ran a smoke test: `daft.from_pydict({"src": ["AAAAAAAAAA"]}).with_column("o", overlay(daft.col("src"), "BBB", 3)).collect()` returns `{"src": ["AAAAAAAAAA"], "o": ["AABBBAAAAA"]}`.
- Compared against a working function (`replace`): same DataFrame path, returns `"XXXXXXXXXX"` for `replace("AAAAAAAAAA", "A", "X")`, confirming the runner, registry, and wiring are healthy.

---

## Implementation Notes

### Phase III Progress

**What I built:**
- The Rust UDF `Overlay` in `src/daft-functions-utf8/src/overlay.rs`, modeled on the existing multi-arg utf8 UDFs (`replace`, `substring_index`, `substr`). It takes `(src: Utf8, replace: Utf8, pos: Int64, len: Int64?)` and produces a `Utf8` array. The kernel iterates row-by-row, broadcasting scalar `pos`/`len` to the column length.
- Registration in `src/daft-functions-utf8/src/lib.rs` (three lines: `mod overlay;`, `pub use overlay::*;`, `parent.add_fn(Overlay);`).
- Python wrapper `overlay(expr, replace, pos, len=None)` in `daft/functions/str.py`, with a docstring example. Re-exported from `daft/functions/__init__.py`.
- 8 Rust unit tests covering the kernel directly (`overlay_str`).
- 13 Python tests covering the user-facing path (`daft.from_pydict` + `df.select(overlay(...))`).

**Challenges faced:**
- My first Rust compile had 7 errors (closure return types, `&DataType` vs `DataType` comparisons, `Option<&&Series>` from a stray `.as_ref()`). I restructured the implementation so the typed kernel takes `&Utf8Array`/`&Int64Array` directly, leaving the `Series -> typed array` unwrapping in the `call()` method. After that, the code compiled cleanly.
- The first two Rust unit tests I wrote had wrong expected values (I miscounted PySpark's "len larger/smaller than replace" cases). The implementation was correct, the test fixtures were wrong. Fixed the expectations and re-ran.
- During the build, `rustfmt` silently reformatted ~40 pre-existing `.rs` files in `daft-functions-utf8` because the repo's `rustfmt.toml` enables unstable features and the existing code was not compliant. I reverted those unrelated files before committing so the Phase III commit is scoped to the `overlay` feature.

### Code Changes

- **Files modified:** 5 total, all scoped to the `overlay` feature.
  - `src/daft-functions-utf8/src/overlay.rs` (new, 249 lines: UDF + 8 tests)
  - `src/daft-functions-utf8/src/lib.rs` (3 lines: register `Overlay`)
  - `daft/functions/str.py` (40 lines: `overlay()` wrapper)
  - `daft/functions/__init__.py` (2 lines: import + `__all__` entry)
  - `tests/expressions/test_overlay.py` (new, 91 lines: 13 tests)
- **Key commits on the branch:** https://github.com/capyBearista/Daft/commits/feat-overlay-3792
  - `1e5e717e8` - Phase II reproduction script (`repro_overlay.py`).
  - `4e8c03bc9` - Phase III implementation (5 files, 385 insertions, all tests passing).
- **Approach decisions:**
  - Two-commit branch (Phase II repro + Phase III implementation) to keep the history clean and easy to review.
  - The Rust kernel uses character-based indexing for consistency with the rest of Daft's string functions. Out-of-range `pos` and `pos=0` are clamped.
  - The `len` argument defaults to `len(replace)`, matching PySpark.
  - For simplicity and consistency with `replace`, I did NOT add a chained `Expression.overlay()` method; users call `daft.functions.overlay(col("src"), "X", 3)` or `from daft.functions import overlay`.
  - I did not modify any pre-existing files other than the 3-line registration in `lib.rs`.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
