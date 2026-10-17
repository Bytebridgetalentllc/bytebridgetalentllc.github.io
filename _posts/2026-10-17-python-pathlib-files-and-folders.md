---
layout: post
title: "Working with Files and Folders in Python Using pathlib"
date: 2026-10-17
description: "A beginner guide to Python's pathlib: build paths, read and write files, find files with glob, and change files safely. Standard library only."
---

Almost every program touches files. It reads a config file, writes a report, or cleans up old logs. Many people still do this with plain strings, like `"data/" + name + ".csv"`. That works until a slash goes missing, or the code runs on Windows, where folders are split with a backslash.

Python has a better tool built in. It is called `pathlib`. This post shows the parts you will use every day. You do not need to install anything.

**Tested with:** Python 3.13.5 on Linux 6.12. Standard library only. Every sample below was run, and the output shown is the real output.

## What is a path?

A **path** is the address of a file or folder on your computer, like `data/reports/sales.csv`. A **relative path** starts from the folder your program is running in. An **absolute path** starts from the top of the disk, like `/home/app/data/sales.csv`.

The `pathlib` module gives you a `Path` object. An **object** here just means a value that also has useful actions (called **methods**) attached to it. So instead of cutting and joining strings by hand, you ask the path for what you need.

![Five steps for working with files using Python pathlib: build a path, inspect its parts, create folders and files, find files with glob, and change files safely](/assets/python-pathlib-files-and-folders.png)

*The five everyday pathlib jobs, from building a path to changing files safely.*

## 1. Build a path and read its parts

You join parts of a path with the `/` operator. Python puts the right separator in for your system, so the same code works on Linux, macOS and Windows.

```python
from pathlib import Path

report = Path("data") / "reports" / "2026" / "sales.csv"

print(report)              # the whole path
print(report.name)         # file name with extension
print(report.stem)         # file name without extension
print(report.suffix)       # the extension
print(report.parent)       # the folder that holds the file
print(report.with_suffix(".json"))  # same path, new extension
```

Output:

```text
data/reports/2026/sales.csv
sales.csv
sales
.csv
data/reports/2026
data/reports/2026/sales.json
```

The **extension** (or **suffix**) is the end part after the dot, like `.csv`. The **stem** is the name without it. The **parent** is the folder one level up. On Windows, the first line would print with backslashes, and you would not change any code.

Note that building a path does not create anything on disk. A `Path` is only an address until you act on it.

## 2. Create folders, then write and read text

```python
from pathlib import Path

out_dir = Path("output") / "notes"
out_dir.mkdir(parents=True, exist_ok=True)   # safe to run many times

note = out_dir / "hello.txt"
note.write_text("first line\nsecond line\n", encoding="utf-8")

text = note.read_text(encoding="utf-8")
print(text.splitlines())
print(note.exists(), note.is_file(), out_dir.is_dir())
```

Output:

```text
['first line', 'second line']
True True True
```

Two settings here save a lot of trouble:

- `parents=True` creates any missing folders on the way, like `output` before `notes`.
- `exist_ok=True` means "do not raise an error if the folder is already there." This makes your script safe to run again.

**Encoding** is the rule that turns text into bytes on disk. If you leave it out, Python picks a default that can be different on each computer. Writing `encoding="utf-8"` every time keeps your files the same everywhere.

`exists()`, `is_file()` and `is_dir()` check what is really on disk right now.

## 3. Find files with glob

A **glob pattern** is a simple search pattern for file names. `*` means "any text", so `*.log` means "every file that ends in `.log`".

```python
from pathlib import Path

base = Path("logs")
for name in ["app/a.log", "app/b.log", "worker/c.log", "worker/readme.txt"]:
    f = base / name
    f.parent.mkdir(parents=True, exist_ok=True)
    f.write_text("x\n", encoding="utf-8")

print(sorted(p.name for p in base.glob("*/*.log")))   # one level down
print(sorted(p.as_posix() for p in base.rglob("*.log")))  # every level
print(sorted(p.name for p in base.iterdir()))        # direct children only
```

Output:

```text
['a.log', 'b.log', 'c.log']
['logs/app/a.log', 'logs/app/b.log', 'logs/worker/c.log']
['app', 'worker']
```

- `glob()` matches the pattern exactly as you write it. Here `*/*.log` means one folder down.
- `rglob()` means "recursive glob". It looks in every folder below, at any depth.
- `iterdir()` lists only what is directly inside one folder.

We wrap the results in `sorted()` because the order files come back in is not fixed. Sorting makes the output the same on every run. `as_posix()` prints the path with forward slashes on every system, which is handy for logs and tests.

## 4. Change a file safely

Imagine your program is writing a settings file and it crashes halfway. Now the file is half old and half new, and the next start fails. A common fix is:

1. Write the new content to a **temporary file** (a short-lived scratch file) in the same folder.
2. When the write is finished, swap it into place with one call to `replace()`.

```python
import os
import tempfile
from pathlib import Path


def write_text_safely(target: Path, text: str) -> None:
    """Write to a temp file in the same folder, then swap it in."""
    target.parent.mkdir(parents=True, exist_ok=True)
    fd, tmp_name = tempfile.mkstemp(dir=target.parent, suffix=".tmp")
    tmp = Path(tmp_name)
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            f.write(text)
        tmp.replace(target)          # one step: old file or new file, never half
    except BaseException:
        tmp.unlink(missing_ok=True)  # clean up if anything went wrong
        raise


settings = Path("output") / "settings.json"
write_text_safely(settings, '{"mode": "test"}\n')
write_text_safely(settings, '{"mode": "live"}\n')
print(settings.read_text(encoding="utf-8").strip())
print(sorted(p.name for p in settings.parent.glob("*.tmp")))  # no leftovers
```

Output:

```text
{"mode": "live"}
[]
```

`tempfile.mkstemp()` makes a new file with a unique name, so two runs never fight over the same scratch file. `replace()` renames the temp file over the target. If the target already exists, it is replaced. On Linux and macOS, a rename inside one file system is **atomic**, which means other programs see either the whole old file or the whole new file, never a mix. That is why the temp file must be in the same folder as the target.

`unlink()` deletes a file. `missing_ok=True` means "do not raise an error if it is already gone."

## 5. A small real task: find old log files

Here is a short script that finds `.log` files older than 7 days. It uses a **dry run**, which means it only reports what it would delete. You check the list first, and only then turn the delete on.

```python
import os
import time
from pathlib import Path


def find_old_files(folder: Path, pattern: str, days: int) -> list[Path]:
    cutoff = time.time() - days * 24 * 60 * 60
    return sorted(p for p in folder.rglob(pattern)
                  if p.is_file() and p.stat().st_mtime < cutoff)


# Set up a test folder: one file 10 days old, one file new.
folder = Path("cleanup_test")
folder.mkdir(exist_ok=True)
old = folder / "old.log"
new = folder / "new.log"
old.write_text("old\n", encoding="utf-8")
new.write_text("new\n", encoding="utf-8")
ten_days_ago = time.time() - 10 * 24 * 60 * 60
os.utime(old, (ten_days_ago, ten_days_ago))   # pretend it is old

to_remove = find_old_files(folder, "*.log", days=7)
print("Would remove:", [p.name for p in to_remove])

DRY_RUN = True
for p in to_remove:
    if DRY_RUN:
        continue
    p.unlink()
print("Still here:", sorted(p.name for p in folder.iterdir()))
```

Output:

```text
Would remove: ['old.log']
Still here: ['new.log', 'old.log']
```

`stat()` returns facts about a file. `st_mtime` is its **modification time**: when it was last changed, in seconds. We use `os.utime()` only to make one test file look old, so you can see the script work without waiting a week.

## Quick habits to keep

- Build paths with `/`, not by adding strings.
- Read parts with `.name`, `.stem`, `.suffix` and `.parent`, not by slicing text.
- Use `mkdir(parents=True, exist_ok=True)` so scripts can run again safely.
- Always pass `encoding="utf-8"` when you read or write text.
- Sort glob results when the order matters.
- For important files, write to a temp file and `replace()` it.
- Start any delete script as a dry run.

These few habits make file code shorter, easier to read, and safer to run on any system.

## Sources

- [pathlib: Object-oriented filesystem paths (Python docs)](https://docs.python.org/3/library/pathlib.html)
- [tempfile: Generate temporary files and directories (Python docs)](https://docs.python.org/3/library/tempfile.html)
- [os.replace and os.utime (Python docs)](https://docs.python.org/3/library/os.html#os.replace)
- [glob: Unix style pathname pattern expansion (Python docs)](https://docs.python.org/3/library/glob.html)

---

## About ByteBridge Talent LLC

![ByteBridge Talent LLC logo](https://raw.githubusercontent.com/Bytebridgetalentllc/Bytebridgetalentllc/main/bytebridge-logo.png){: width="64"}

ByteBridge Talent LLC is based in Grand Island, Nebraska. Our software engineering and IT staffing teams work on software development, cloud and DevOps, and data. Learn more at [bytebridgetalent.com](https://bytebridgetalent.com).
