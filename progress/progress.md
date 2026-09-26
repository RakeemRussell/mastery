# progress.md

**What this file is for:** one row per week of the Zero to Hired program. Focus, whether the
week's deliverable actually shipped, gate status, hours logged, and — the highest-value column —
**what surprised me**. Surprise marks where your mental model was wrong, which is exactly what
interviewers probe for later. Add a new row every Sunday (or whenever you close out a week),
never edit old rows except to fix a typo.

---

## Setup (pre-Week 1)

- **Focus:** Environment setup — provisioning the Linux lab VM
- **Deliverable shipped:** N/A (no lab deliverable this stage — infrastructure only)
- **Gate status:** N/A
- **Hours logged:** ~1.5–2 (across VirtualBox install, PATH troubleshooting, ISO download, VM creation)
- **What surprised me:** Windows Home can't use Multipass's default Hyper-V backend, so the CLI path
  ended up going through VirtualBox + `VBoxManage` instead — a fully scriptable alternative I didn't
  expect to exist. Also: a `winget`-installed tool not being on `PATH` immediately, and Ubuntu's
  point-release ISO filenames changing often enough that a hardcoded filename from days ago was
  already stale.

**What I set up (quick summary):** created the tracking folder structure with two commands —
`mkdir -p ~/mastery/progress && cd ~/mastery && git init` made the folder and turned it into a git
repo, then `touch progress/progress.md progress/incident-journal.md progress/decisions.md` created
these three empty files inside it. Full reasoning behind each command is in `decisions.md`; the
underlying mechanics (inodes, system calls, how git actually stores things) are in a reference note
at the bottom of `incident-journal.md`.