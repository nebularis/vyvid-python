---
name: testing
description: Diagnosing and speeding up a slow pytest suite and writing Python tests that stay fast. Use when a test or check task is slow, when adding or parallelising a suite with pytest-xdist, when a suite may run short of memory, when sizing workers, when profiling a run, when a repository-wide text scan must work on every platform, or when reading a slow run as a regression.
---

# Testing in Python

This skill's tasks span the activities testing and its kinds, such as unit-testing, regression-testing, conformance-testing and performance-testing, with performance-tuning and scenario-and-test-data, and a task often needs several of them, named together and separated by commas. Read each row's flags and the line under it, where an entry's boundary, its successor and the debt against it are named.

## The defaults for every suite

These are how a suite is built from the start, not fixes applied once it is slow. A suite that departs from one says why in its task description.

- **Parallel by test.** The task runs `pytest -n auto`, with `--dist worksteal`, or `--dist loadgroup` where a module's tests share one slow per-process computation and are put in one `xdist_group`. Nothing a test writes is shared between workers: temporary directories per test, no file written in the tree.
- **Sized to the memory available.** The suite's `conftest.py` answers `pytest_xdist_auto_num_workers` from the memory free and its own measured peak per worker (below), and `PYTEST_XDIST_AUTO_NUM_WORKERS` still overrides it.
- **Benchmarks are not part of `check` either.** A test that measures belongs under `{{benchmark_root}}` and fails on a wrong result, never on time.
- **Measured before and after.** A change made for speed records the wall time, the worker count and the peak memory, and keeps every assertion, as the next section says.
- **A new check is seen to fail before it is relied on.** Break what it checks, watch it fail, then restore it. A check that has never failed may be checking nothing.

## Memory: sizing workers

Most runs happen on an 8 GB laptop with an editor and other tools open, so a suite must not take more memory than the machine has free. Give the suite as many workers as fit in the memory available less a 1 GiB reserve, up to the default count and never fewer than two, which runs longer and smaller. Measure the suite's own peak per worker with `-n 0`, and use that figure, not a guess. Any suite that can run short of memory, or that starts more than two workers, sizes its workers this way.

More workers than cores that do real work add contention, not speed. On four performance and four efficiency cores, 8 workers gave the wall time of 4 and doubled the summed test time.

## Measure before changing anything

A slow suite's cause is a hypothesis until profiled. Guessing wrong costs more than measuring.

- Check the machine before reading a slow run as a regression. On a laptop, Low Power Mode made the same suite 3.5 times slower (126 s against 36 s), and memory pressure, a video call or a sync client moved it by tens of seconds. Compare two commits by running both on the same machine in the same state.
- Profile the suite as one real run, with the task's own flags and a JUnit report, and read the wall time, the slowest files and the slowest tests. A tail made of a few big files, or one stalled test, shows up here. A run reports nothing until it ends, so running each file in its own process with a timeout marks the one that hangs. Its sum exceeds a real run.
- `python -m pytest <file> -q -p no:cacheprovider --durations=40 --durations-min=1.0` for one file's own ranking.
- Before changing a shared fixture, capture a baseline you can diff against exactly:
  ```powershell
  python -m pytest <files> -q -p no:cacheprovider -rA 2>&1 |
    Select-String '^(PASSED|FAILED|SKIPPED|ERROR)' | Sort-Object > before.txt
  ```
  or, in a POSIX shell:
  ```bash
  python -m pytest <files> -q -p no:cacheprovider -rA 2>&1 | grep -E '^(PASSED|FAILED|SKIPPED|ERROR)' | sort > before.txt
  ```
  After a change, regenerate the same way and `Compare-Object before.txt after.txt`, or `diff before.txt after.txt`. Anything other than an expected cosmetic difference is a regression until explained.
- **A whole-suite number can be misleading once several independent causes are at play.** One measurement: re-running the slowest family of modules together in one process, after adding a shared cache, showed almost no improvement over running them as separate subprocesses (289s against 293s, within noise), even though the cache was demonstrably correct and working. Profiling confirmed why: for a suite whose tests mostly validate necessarily-distinct, freshly mutated graphs, the dominant cost is the sheer number of `pyshacl.validate()` calls (SPARQL constraint evaluation), which no cache collapses, since no two calls share input. Parallelising the *files themselves* (`pytest-xdist`) was what moved the number, not sharing parsed graphs. Don't assume a caching layer is the fix: measure the combined effect, not just that the cache itself works.

## `pytest-xdist`: the first thing to try

`-n auto --dist worksteal` spreads the tests over workers, and an idle worker takes tests from a busy one. The first version, `--dist loadfile`, ran each file in its own worker so that files were independent by construction. It was the single highest-value change in that work, roughly a 3.6x reduction with byte-identical outcomes, but a file is then the unit of work and the largest file sets a floor on the run. Measured on two machines with all tests passing, `worksteal` was 9 to 11 percent faster, and it cost about 2 percent more summed test time because each worker parses what it needs. A shared, session-scoped fixture is still worth doing, but each `xdist` worker gets its own copy, so cross-file sharing happens only for tests that land on the same worker.

More workers than cores doing real work do not help. On a laptop with four performance and four efficiency cores, 8 workers gave the same wall time as 4 and doubled the summed test time, since the same work ran at half the speed per worker. A summed time that doubles with the worker count is contention, not slow tests. `PYTEST_XDIST_AUTO_NUM_WORKERS` sets what `auto` means for one machine.

## Lessons from a large suite

A suite went from 4m12s serial to about 1m35s on a loaded machine. What paid, in order: `-n auto --dist loadgroup`, a session-wide translation of each distinct query text once for a suite that calls `pyshacl.validate` directly in many places, running the children of a hash-seed sweep side by side (a thread pool over `subprocess.run`), memoising a pure function that parses SPARQL, and parsing a fixture once per process. A module whose tests share one slow per-process computation goes in one `xdist_group` (`--dist loadgroup`), or every worker repeats it. Count parses before caching them: a profile showed 5901 SPARQL parses of 1675 distinct texts, and the update texts of a sweep are distinct per history, so only reads gain. The summed CPU time rises with the worker count on 4 performance plus 4 efficiency cores, so keep `auto`. Not tried successfully: sharing rdflib's update parse tree between sweep children by pickle (`ParseResults` did not round-trip).

## Repository-wide text scans: don't shell out to `git grep`

`git grep` has two real, confirmed cross-platform failure modes, not hypothetical ones:

- **On Windows, a subprocess's captured output decodes with the console codepage (`cp1252`), not UTF-8**, by default when a parent process doesn't force it. A real test crashed with `UnicodeDecodeError: 'charmap' codec can't decode byte 0x81` for exactly this reason. `text=True` alone does not fix it, decode explicitly as UTF-8 instead of shelling out at all.
- **POSIX ERE has no `\b`, and some platforms' `git grep -E` silently ignore it rather than erroring on it**, so a pattern using `\b` can pass on one platform while finding nothing real on another. Use `([^A-Za-z0-9_]|$)` instead, which means the same thing on every platform.

Put one helper in the suite's shared `{{shared_fixtures}}` that replaces a `git grep` subprocess call: it lists files with `git ls-files` (tracked, and untracked but not ignored, so a file just written is scanned and an ignored directory never is), reads every file as UTF-8 explicitly, and matches line by line the way `git grep` does (so `^` and `$` anchor one line, not the whole file). In a checkout with no `.git` it raises, since a scan there cannot tell an ignored file from a tracked one.

**The one real cost of including untracked files**: it sees them too, which `git grep` would not. This bit a roll-out for real: a new test fixture file contained a literal string as test data for exercising the pattern matcher itself, and a retired-term scan picked it up as a false positive. Fixed the same way the suite already handled its own constants file: add the new file to the scan's own allow-list. If this keeps happening, it is a sign the roots are too broad, not that the approach is wrong (skip a named list of build and dependency directories, namely `.git`, `__pycache__`, `node_modules`, `.venv`, `.build`, `build`, `dist`, `target`, `.pytest_cache`, and similar).

## Guard "supplied or defaulted" at the entry point

When a library function requires a parameter explicitly, the only place it can be silently defaulted, such as a `--now` flag defaulting to the wall clock, is the CLI's own argument handling.
