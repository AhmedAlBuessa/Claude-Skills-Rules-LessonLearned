---
name: blocking-app-network-access
description: Use when the user wants to fully cut a specific Windows application off from the internet — stop it phoning home, checking for updates, validating a license/subscription, or sending telemetry, ads, promotions, or news, including via helper processes, updater services, or scheduled tasks. Triggers on requests like "block <app> from the internet", "stop <app> reaching its company servers", "prevent <app> from checking for updates / activating / sending data".
---

# Blocking an App's Network Access (Windows)

## Overview

Completely and permanently cut a user-specified Windows application off from the
internet — in EVERY shape, for EVERY reason (update checks, license/activation,
subscription validation, telemetry, crash reporting, analytics, promotions,
news, "phone-home" to the parent company or ANY company). The target app and
ALL of its related processes must never reach the network again, while still
running and uninstalling locally.

**Platform:** Windows 10/11. Uses Windows Defender Firewall + the hosts file +
service/scheduled-task control. No third-party tools required.

## When to Use

- "Block Acme from the internet" / "stop it phoning home"
- "Prevent <app> from checking for updates / activating / checking my license"
- "Cut off <app>'s telemetry / ads / promotions / news"
- User worries about helper files or background services doing the connecting

**Not for:** blocking a whole machine, blocking a website in a browser, or
malware removal. This blocks ONE installed app and everything it ships with.

## The One Thing That Must Happen First

**STEP 0 — ASK FOR THE TARGET.** Before anything else, stop and ask the user to
identify the app. Any ONE of these is enough (more is better):
- App name (e.g. "Acme Editor")
- Process name / executable (e.g. "acme.exe")
- Installation path (e.g. `C:\Program Files\Acme\`)

Confirm back which app you understood, then discover the rest yourself. If you
can't find it, ask again — **never guess or block the wrong app.**

## Default Decisions — Apply Automatically, DO NOT Ask

These are pre-decided. Do NOT pause to ask the user's preference. Just take the
most thorough option and report what you did.

| Decision | Always do | Never ask |
|----------|-----------|-----------|
| **Block scope** | Block EVERY executable in the app's install/related folders (not just the obviously network-facing ones), outbound TCP+UDP, all firewall profiles | "Should I block all binaries or only some?" |
| **Self-heal** | Stop AND disable any related Windows Service, and disable any related Scheduled Task, that could restart/relaunch/re-add access | "Should I disable the service/task or just block it?" |

The ONLY two things you may pause for: **STEP 0 (the target)** and the **manual
"open the app now" confirmation** in the live test. Everything else: decide and
proceed autonomously.

> If you catch yourself about to ask "how aggressive should I be?" or "want me
> to disable the service too?" — STOP. The answer is already yes. Maximum scope,
> neutralize self-heal, keep going.

## Rules of Engagement

1. **Reversible + logged.** Log EVERY firewall rule, hosts entry, service, and
   task you touch. End with a single UNDO script that restores everything.
2. **Elevation required.** All firewall/hosts/service commands need an
   Administrator PowerShell. Detect this EARLY and use the one-click path in
   the Elevation section below — do NOT default to "relaunch Claude".
3. **Never delete the app or its data.** Only block network access.
4. Show the plan, then proceed autonomously — no per-step approval needed.

## Elevation (handle this EARLY, before applying anything)

Blocking ALWAYS needs admin. Check elevation up front — right after discovery,
before the first firewall/hosts/service command — so the user isn't surprised
mid-run.

**Detect:**
```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```
`True` = elevated, proceed. `False` = not elevated, do the hand-off below.

**If NOT elevated — DO NOT just tell the user to relaunch Claude.** That is the
slow path. Instead:

1. First, write the full **Apply** and **Undo** scripts to disk (writing files
   needs no elevation), so everything is ready to run. Use a clear folder like
   `C:\Users\<user>\<App>-Block\` with `Apply-<App>Block.ps1` (carries a
   `#requires -RunAsAdministrator` guard, logs every action, backs up the hosts
   file first) and `Undo-<App>Block.ps1`.
2. Then offer the **one-click elevation** path as the default: the user pastes
   this into the Claude Code input box (the `!` runs it in their session, and
   `Start-Process -Verb RunAs` triggers a single UAC prompt → they click Yes):

   ```
   ! powershell -Command "Start-Process powershell -Verb RunAs -ArgumentList '-NoExit','-ExecutionPolicy','Bypass','-File','C:\Users\<user>\<App>-Block\Apply-<App>Block.ps1'"
   ```

   A new elevated window opens, runs all blocking layers, and (`-NoExit`) stays
   open; results also go to the script's log file.
3. Tell the user the **live monitoring test + static verification can then run
   in THIS same non-elevated session** — `Get-NetTCPConnection`,
   `Resolve-DnsName`, `netstat` are read-only and need no admin. So after the
   UAC run, just continue here; no relaunch required.

Only offer "relaunch Claude Code from an elevated PowerShell" as a SECONDARY
option for users who prefer Claude to drive every apply step interactively.

**Edge cases to mention if the UAC prompt fails:** the user's account must be an
Administrator account (a Standard account's UAC prompt asks for a separate
admin password). Never suggest disabling UAC — the one-click path works with
UAC on.

## Procedure

### 1. Discover (don't assume)
- Find the main exe(s) **and every helper/child binary**: updater, crash
  reporter, "service"/"agent" exe, bundled browsers (CEF/Electron),
  node/python helpers, scheduled-task launchers. Search recursively in the
  install folder, `%ProgramFiles%`, `%ProgramFiles(x86)%`, `%LOCALAPPDATA%`,
  `%APPDATA%`, `%ProgramData%`.
- List running processes belonging to the app with full paths.
- Identify vendor Windows Services and Scheduled Tasks (the "small related apps"
  that connect on the main app's behalf or self-heal the blocks).
- Identify domains/IPs: check config files, hardcoded update/license/telemetry
  URLs, and live connections via `Get-NetTCPConnection` / `netstat -ano` mapped
  to the app's PIDs (or a short `pktmon` capture).

### 2. Block (all layers — defense in depth)
- **Firewall:** per-executable OUTBOUND block for EVERY discovered binary,
  TCP+UDP, all profiles (Domain/Private/Public). Use `New-NetFirewallRule` with
  a name prefix like `BLOCK-<appname>-...` so rules are easy to find/remove.
  Block inbound too if relevant. Verify each rule.
- **Hosts file:** add `0.0.0.0` entries for every vendor/update/license/
  telemetry/analytics/promo/news domain found, plus obvious variants (`api.`,
  `update.`, `license.`, `telemetry.`, `cdn.`, `analytics.` subdomains).
  **Clear the `ReadOnly` attribute on the hosts file FIRST** (other blockers
  often leave it set — see Script Robustness). Then `ipconfig /flushdns`.
- **Anti-trick hardening:** stop AND disable updater services/tasks; block their
  exes by path too (so a restarted service still can't reach the net); block any
  watchdog/relauncher process.

### Script Robustness (learned from real runs — DO ALL THREE)

When you generate an Apply script, it MUST be:

1. **Hosts-attribute-safe.** Before writing the hosts file, clear its ReadOnly
   flag, and restore your intent afterward. A prior blocklist (e.g. an Adobe
   block) commonly leaves `hosts` ReadOnly — `Copy-Item`/read succeeds but
   `Add-Content`/write fails with `Access denied`. Do this in BOTH Apply and
   Undo:
   ```powershell
   $hosts = "$env:WINDIR\System32\drivers\etc\hosts"
   if ((Get-Item $hosts -Force).IsReadOnly) { Set-ItemProperty $hosts -Name IsReadOnly -Value $false }
   ```
   If write STILL fails after clearing ReadOnly, suspect **Controlled Folder
   Access** (Windows Security → Ransomware protection) guarding System32 — tell
   the user to allow the script or temporarily disable it. Ignore unrelated
   Defender noise like error `0x800106ba`.

2. **Fault-isolated per layer — NEVER all-or-nothing.** Do NOT let a global
   `$ErrorActionPreference='Stop'` abort the whole run on one failure. Wrap each
   layer (firewall / hosts / service+task) in its own `try/catch`, log the
   failure, and CONTINUE to the next layer. Defense-in-depth is worthless if one
   layer's error silently skips the others. Print a per-layer PASS/FAIL summary
   at the end.

3. **Idempotent and re-runnable.** Every step is skip-if-exists: firewall rule
   already present → skip; hosts block already there → skip; service already
   disabled → skip. Then recovering from a partial run is just "run Apply
   again" — no bespoke Resume script needed.

### 3. Live Monitoring Test (after blocking)
1. Kill the app and ALL related processes for a clean baseline; confirm nothing
   of the app is still running.
2. Ask the user to manually open the app, and WAIT for confirmation.
3. On confirmation, monitor the app's PIDs for **10 seconds**. Each second:
   re-resolve PIDs (catch new child/helper PIDs by name + install path), run
   `Get-NetTCPConnection` and record any `SynSent`/`Established`/outbound
   attempt to a non-local address (note UDP too), and **print a visible
   countdown** — `Monitoring... 10s left`, `9s left`, … `1s left`, `0s — done`,
   one line per second.
4. Report **PASS** (no outbound attempts) or **FAIL** (list PID, process + full
   path, remote IP:port, state, resolvable domain). For each leak: immediately
   block that exact exe, then have the user close + reopen the app and re-run
   the 10-second test until it comes back PASS.

> A 1-second poll can miss a sub-second phone-home that opens and closes between
> samples. If airtight proof is needed, run a continuous `pktmon` capture for
> the 10 seconds instead of polling.

### 4. Verify (static)
- Re-list created firewall rules; confirm enabled on all profiles.
- Confirm hosts entries dead-end at `0.0.0.0` (`Resolve-DnsName` / ping shows no
  route).

## Deliverables
1. A table of everything blocked (binary paths, firewall rule names, hosts
   domains, services/tasks disabled).
2. The PASS/FAIL result of the 10-second live test, with evidence.
3. A single UNDO script that removes every firewall rule, restores the hosts
   file, and re-enables any service/task disabled.

## Common Mistakes
- **Blocking only the main exe.** Updaters/services/helpers phone home on the
  app's behalf — block every binary, per the default scope.
- **Forgetting self-heal.** A vendor service set to auto-start can recreate
  connections; it must be stopped AND disabled, not just firewalled.
- **Hosts-file-only thinking.** Hosts blocks domains, not raw-IP connections —
  it's a supplement; the per-exe firewall block is the real defense.
- **Asking the user about scope/self-heal.** Pre-decided — see Default Decisions.
- **Running without elevation.** Firewall/service/hosts edits silently fail or
  error without admin rights. Check elevation early and use the one-click
  `Start-Process -Verb RunAs` path (see Elevation) — don't make the user
  relaunch Claude.
- **Hosts `Access denied` even when elevated.** The hosts file's ReadOnly
  attribute (left by a prior blocklist) — clear `IsReadOnly` before writing.
  See Script Robustness.
- **One layer's failure aborting the rest.** A global `ErrorActionPreference =
  'Stop'` lets a hosts error skip the service/task layer. Isolate each layer in
  try/catch and continue. See Script Robustness.
- **Non-idempotent Apply script.** If it can't be safely re-run, a partial
  failure forces a hand-written resume script. Make every step skip-if-exists.
