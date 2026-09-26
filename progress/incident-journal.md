# incident-journal.md

**What this file is for:** every break/fix, every real error you hit and resolved — whether it's
a deliberate Break/Fix Friday scenario or just something that broke while you were trying to do
something else. Format: symptom → hypothesis → confirmed → root cause → fix → prevent →
**surprised**. That last line matters more than the rest — it's the edge of what you didn't know.
By week 26 this becomes your interview story bank for "tell me about a time you debugged something."

---

## INC-01 — `VBoxManage` not recognized (setup, ~10 min)

**Symptom:** `VBoxManage --version` returned `CommandNotFoundException` immediately after installing
VirtualBox via `winget install -e --id Oracle.VirtualBox`.

**Hypothesis:** Either the install silently failed, or the install succeeded but PowerShell's
current session doesn't know the new `PATH` entry yet.

**Confirmed:** `Test-Path "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"` returned `True` —
the binary existed on disk, so this ruled out a failed install.

**Root cause:** `winget` writes the updated `PATH` to the Windows registry, but an already-open
PowerShell process doesn't re-read environment variables from the registry — it only has the
`PATH` it started with.

**Fix:** Added the VirtualBox install directory to `PATH` for the current session
(`$env:Path += ";C:\Program Files\Oracle\VirtualBox"`) and then made it permanent at the machine
level via `[Environment]::SetEnvironmentVariable(...)` from an elevated PowerShell.

**Prevent:** After any `winget install` of a CLI tool, open a **new** terminal window before
assuming the install failed — don't debug in the same session that predates the install.

**Surprised:** That a successful install can still produce a "command not found" error purely
because of *when* the terminal was opened relative to the install — the tool and the shell were
both right, just out of sync with each other.

---

## INC-02 — Ubuntu ISO download returned 404 (setup, ~5 min)

**Symptom:** `Invoke-WebRequest -Uri "https://releases.ubuntu.com/24.04/ubuntu-24.04.1-desktop-amd64.iso"`
returned `Not Found` from the Ubuntu mirror server.

**Hypothesis:** The specific point-release filename (`24.04.1`) no longer exists on the server —
Ubuntu periodically replaces older point releases with newer ones under the same `24.04/` directory.

**Confirmed:** Fetching the directory listing directly (`Invoke-WebRequest -UseBasicParsing`
against `https://releases.ubuntu.com/24.04/`) and filtering `.Links` for anything matching
`desktop-amd64.iso$` returned a different, newer filename (`ubuntu-24.04.5.1-desktop-amd64.iso`).

**Root cause:** Hardcoding a specific point-release version number in a URL is fragile — Ubuntu's
`24.04/` directory only ever hosts the *current* point release, and old ones get removed once
superseded.

**Fix:** Scraped the directory listing at request time to find the current filename instead of
hardcoding one, then downloaded using that discovered name.

**Prevent:** For any "latest version" download, prefer scripting a lookup of the current filename/
version over hardcoding one — especially for anything that will be re-run more than once (this
exact problem will recur for Terraform providers, AMIs, and Kubernetes versions later in the
program).

**Surprised:** How quickly a copy-pasted command from even a few days earlier had already gone
stale — the URL structure was stable, but the specific file wasn't, and there was no error message
warning that the version itself was the problem rather than the URL being wrong.

---

## REF-01 — Mechanism notes: how the tracking-folder commands actually work

*Not an incident — nothing broke here. Filed as a reference note rather than a real entry above, so
it doesn't get mixed up with actual break/fix material later. Covers what's really happening
underneath:*

```bash
mkdir -p ~/mastery/progress && cd ~/mastery && git init
touch progress/progress.md progress/incident-journal.md progress/decisions.md
```

**`mkdir -p`:** creating a directory allocates a new inode — the data structure holding a file or
directory's metadata (permissions, owner, timestamps, pointers to its data) — and adds an entry for
it in its parent directory's listing. `-p` makes `mkdir` walk the given path component by component
(`mastery`, then `progress`), checking at each step whether that piece already exists before
creating it, instead of trying to create only the final component directly and failing if a parent
is missing.

**`cd`:** the shell keeps a "current working directory" as part of its own process state — visible
on Linux as the `cwd` symlink under `/proc/<pid>/cwd`. `cd` issues the `chdir()` system call, asking
the kernel to update that pointer for the shell's own process only. It has no effect on any other
open terminal or process, which is why a fresh terminal window always resets to wherever it started.

**`git init`:** creates a hidden `.git/` directory holding Git's internal state: an `objects/`
folder (empty at this point) where every version of every tracked file's content will eventually be
stored as compressed, SHA-1-addressed blobs; a `refs/` folder where branches are literally small
text files containing a commit hash; a `HEAD` file pointing at the current branch; and a `config`
file for repo settings. At the moment `git init` runs, none of that has anything to track yet — it
installs the tracking apparatus, it doesn't start tracking any specific file.

**`touch`:** for a file that doesn't exist, this makes the same underlying system call a program
uses to open a file for writing, then closes it immediately without writing any bytes — producing a
real, zero-length file. For a file that already exists, `touch` instead updates its modification
timestamp via `utime()` without touching its contents.

**Why the new files aren't tracked yet:** a file existing on disk and a `.git/` folder existing in
the same directory are independent facts. Git only builds its picture of "what changed" when you
run `git status`, `git add`, or `git commit` — until then, files created by `touch` are just
ordinary files that happen to sit inside a directory Git is watching.