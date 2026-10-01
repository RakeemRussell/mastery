# decisions.md

**What this file is for:** a lightweight decision log (ADR-style) for every non-obvious technical
choice made and why.

---

## D-01 — VM hypervisor: VirtualBox instead of Multipass

**Decision:** Use VirtualBox + `VBoxManage` (CLI) to provision the Week 1 lab VM.

**Context:** The curriculum's default recommendation is Multipass, which on Windows requires
Hyper-V as its backend.

**Alternatives rejected:**
- *Multipass* — rejected because Hyper-V is unavailable on Windows 11 **Home** edition (Home can't
  enable the Hyper-V Windows feature at all, only Pro/Enterprise/Education can).
- *WSL2* — rejected per the curriculum's own guidance: WSL doesn't give a full VM lifecycle
  (boot/network/snapshot/destroy), which is explicitly part of what Lab 1 is meant to teach, since
  that lifecycle mirrors how cloud instances work.

**Why VirtualBox:** Works on Windows Home with no Hyper-V dependency, is fully scriptable via
`VBoxManage` from PowerShell (so the CLI-first habit isn't lost), and supports the same
snapshot/restore workflow (`VBoxManage snapshot`) that Multipass's `snapshot`/`restore` commands
provide.

---

## D-02 — VM creation: fully scripted via `VBoxManage`, not the GUI wizard

**Decision:** Build the VM (create, attach disk, attach ISO, unattended OS install) entirely through
`VBoxManage` commands in PowerShell rather than VirtualBox's graphical New Virtual Machine wizard.

**Alternatives rejected:**
- *GUI wizard* — works fine, but produces no reusable record of *how* the VM was built; if the VM
  needs rebuilding later (e.g. after a snapshot mistake), the steps aren't captured anywhere.

**Why scripted:** Every step is a plain command that can be copy-pasted to rebuild the exact same VM
later

**Trade-off accepted:** The unattended install (`VBoxManage unattended install`) is more brittle
than clicking through the graphical installer — it can fail silently on some ISO/VirtualBox version
combinations. Accepted this risk with a documented fallback (drop to `VBoxManage startvm` without
`--type headless` and install manually) rather than defaulting straight to the GUI path.

---

## D-03 — ISO filename resolved dynamically, not hardcoded

**Decision:** Look up the current Ubuntu 24.04 desktop ISO filename at download time by scraping
`releases.ubuntu.com/24.04/`'s directory listing, instead of hardcoding a specific point-release
filename.

**Context:** See INC-02 — a hardcoded filename (`24.04.1`) had already gone stale by the time it was
used.

**Alternatives rejected:**
- *Hardcode and update manually each time* — rejected because it produces the exact failure already
  hit once, with no warning that the failure is a stale version rather than a real error.

**Why dynamic lookup:** The same class of problem (a "latest version" reference going stale) recurs
constantly later in the program — Terraform provider versions, AMI IDs, Kubernetes/EKS versions — so
it's worth building the habit of resolving "current" programmatically now rather than hand-tracking
version numbers.

---
