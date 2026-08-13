# Troubleshooting: Runtime Panic

This document explains what a **runtime panic** is in `ndr` (the NetDust interpreter CLI), why it happens, and how to diagnose and report one.

## What is a "runtime panic"?

`ndr` distinguishes between two very different kinds of failure:

| Type | Source | How it's reported |
|---|---|---|
| **Script error** | Your `.nd` script has a mistake (syntax, logic, etc.) | `NetDustResult.Errors`, printed as a normal `✗ failed` message with source context |
| **Runtime panic** | The interpreter itself threw an exception it did not expect | A dedicated pink-on-black `runtime panic` screen, exit code `70` |

A runtime panic means `interpreter.Run()` let a `std::exception` (or an unknown exception) escape instead of returning a normal `NetDustResult`. This is treated as an **internal `ndr` bug**, not a problem with your script — that's exactly what the panic screen text says:

```
this is an internal ndr error, not a bug in your script.
please report it, including the script that triggered it.
```

## Where it comes from in the code

Every code path that calls `interpreter.Run()` wraps the call in a `try`/`catch`:

```cpp
try
{
    result = interpreter.Run(lines, scriptDir);
}
catch (const std::exception &ex)
{
    throw RuntimePanic(ex.what());
}
catch (...)
{
    throw RuntimePanic("unknown internal error");
}
```

`RuntimePanic` is just a marker type (`struct RuntimePanic : std::runtime_error`) used to tell "the interpreter blew up" apart from "the interpreter finished and reported errors normally." Whoever catches `RuntimePanic` further up calls `printRuntimePanic()` and exits with code `70`.

This pattern appears in three places, so a panic can surface slightly differently depending on how you invoked `ndr`:

- **Normal run** (`runInterpreterWithSpinner`) → pink panic screen on stderr, exit `70`
- **`--benchmark N`** → panic screen shown, run counter reported, exit `70`
- **`--json`** → no pink screen; instead you get `{"success":false, ..., "error":"panic: <message>"}`, exit `70`

## Confirm it's actually a panic

Check the exit code and/or the message shape:

- Exit code **`70`** = panic (a `1` is a normal script error, not a panic).
- Normal terminal output shows the pink/black `runtime panic` box.
- `--json` output shows `"error":"panic: ..."`.

If you're seeing a plain red `✗ failed` block with line/column context instead, that's a **script error**, not a panic — fix the `.nd` source instead of filing an internal bug.

## Reproduce with more information

1. **Get the exact panic message.**
   ```bash
   ndr your_script.nd
   ```
   Copy the text inside the pink box (or run with `--json` for a clean, copy pasteable string).

2. **Re run in debug mode** to see how far execution got before it panicked:
   ```bash
   ndr -d your_script.nd
   ```
   `--debug` prints a line by line trace, so the panic should occur right after the last traced line | that line is almost always the trigger.

3. **Minimize the script.** Delete unrelated `room`s/statements until the panic stops reproducing. A minimal repro is the single most useful thing you can attach to a bug report.

4. **Try `--eval`** to rule out file/encoding issues:
   ```bash
   ndr -e "code causing the panic;"
   ```

5. **Check `--profile`** only if the panic seems timing related (e.g., it only happens on slow/fast machines) | this can hint at a race condition rather than a script bug.

## Common internal causes to check first

Since a panic means an *unexpected* exception, the usual suspects when investigating the interpreter/runtime source are:

- **Out of range access** | `std::vector`/`std::string` indexing inside the interpreter without bounds checks (e.g. a `.at()` or `[]` on an empty container).
- **Null or dangling pointers** | especially around the native bridge / module system.
- **`std::regex` exceptions** | malformed patterns thrown from internal parsing.
- **File I/O exceptions** | e.g. `bring`-ing a library file that fails to open/read partway through merging.
- **Threading/spinner interaction** | `runInterpreterWithSpinner` runs the spinner on a separate thread guarded by `termMutex`/`stdoutActivity`. If you only see panics intermittently (not on `-d`, where the spinner is skipped), suspect a race rather than the script itself.
- **`NSApplicationMain_StartRunLoop_IfAnyWindowOpen()`** (macOS only) | if a panic only happens when a native window is involved, check the run loop interaction described in the source comments (interpreter execution intentionally stays on the main thread for this reason).

> Tip: there's a commented out test line in `main.cpp` you can use to sanity check that panic handling itself works end to end:
> ```cpp
> // for panic test
> //throw std::runtime_error("vector index out of range");
> ```
> Uncomment it temporarily to confirm the panic screen / exit code / `--json` path all behave as expected in your build.

## Filing a report

Include all of the following:

- [ ] `ndr -v` / `ndr -i` output (version, arch, platform)
- [ ] The exact panic message (from the pink box or `--json`)
- [ ] The minimal `.nd` script that reproduces it
- [ ] Whether it reproduces with `-d` (debug) and whether it's consistent or intermittent
- [ ] OS/platform (this build targets `arm64` macOS and Linux | note if you're on something else)
- [ ] Exact command line used

## Exit code reference

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Script error / CLI usage error |
| `70` | **Runtime panic** (internal error) |

---

**Bottom line:** a runtime panic is never something to "fix" in your `.nd` script | it's a signal that `ndr` itself hit a case it doesn't handle. Minimize the repro, grab the exact message, and file it against runtime.
