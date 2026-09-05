---
title: "set -o pipefail Plus grep -q Is a False Negative — But Only on Real Data"
collection: journal
permalink: /journal/pipefail-grep-q-sigpipe/
excerpt: "Three consecutive builds aborted claiming a file was missing from an archive that demonstrably contained it. The unit test passed, because the fixture archive had four entries and the real one had four thousand. grep -q exits on match and SIGPIPEs the writer."
---

**The trick:** never pipe into an early-exiting reader — `grep -q`, `head`, a loop with `break` — under `set -o pipefail`. Capture the output into a variable first, then match against that.

## Issue

Three consecutive builds aborted with `source package is missing <path>` — against a package that visibly contained the file. Unzipping it by hand and looking took about ten seconds.

The unit test covering that exact check passed.

## Root Cause

The check was one line:

```bash
set -o pipefail
unzip -l "$zip" | grep -q " $path$"
```

`grep -q` exits the moment it finds a match. That closes the read end of the pipe, so the still-writing `unzip` takes `SIGPIPE` and dies non-zero. Under `pipefail`, the exit status of the pipeline is the rightmost non-zero status — so the pipeline reports failure **precisely because the file was found**.

What makes this properly nasty is that it is **input-size dependent**.

With a four-entry fixture archive, `unzip` has finished writing and exited before `grep` finds its match and leaves. Nothing is signalled, the pipeline returns zero, the test passes, and the check looks correct forever.

With the real four-thousand-entry package, `unzip` is always still writing when `grep` departs. It failed every single time — which is exactly what a genuinely missing file looks like, and exactly why nobody suspected the check itself for three builds.

## Solution

Remove the pipe. There is nothing to signal if there is no reader to leave early:

```bash
listing="$(unzip -l "$zip")"
case "$listing" in
  *" $path"*) : ;;
  *) die "source package is missing $path" ;;
esac
```

Capture first, then match. No subprocess, no early exit, no `SIGPIPE`.

The regression test now builds a **four-thousand-entry** archive. A small fixture cannot reproduce the bug, so a small fixture is not a test of this behaviour — it is a test of a different program that happens to share source code.

## 💡 Takeaway

- **`pipefail` plus any early-exiting reader is a latent false failure.** `grep -q`, `head`, `read`, a `while` loop with a `break` — all of them can kill a healthy writer and have the shell report it as an error.
- **The failure is size-dependent, which is the worst kind.** It is absent in development, deterministic in production, and its symptom is a plausible message about your data rather than about your code.
- **If your fixture is smaller than production, you have not tested the property you think you tested.** Where a bug's existence depends on scale, the test must reproduce the scale.
- **`pipefail` is still correct.** The fix is not to switch it off — it is to stop building pipelines whose reader leaves first.
