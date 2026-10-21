# Agents Guide

## Project Overview

systemcheck is a system health check tool for Kicksecure and Whonix.
It runs a series of diagnostic functions and reports results via CLI and GUI (msgcollector).

## Tests

Comprehensive tests live in the AI-maintained dist-ai repo, not here:
https://github.com/org-ai-assisted/dist-ai

## Architecture

- Shell scripts in `usr/libexec/systemcheck/` with a `.bsh` file name
  extension are sourced by the main `systemcheck` script.
- `preparation.bsh`'s `preparation` function is run before any other checks, and provides shared helpers.
- `updatecheck` is a related tool that checks for available package updates.
- The `canary` system (including the `canary-download` Python script) fetches and
  verifies Kicksecure's warrant canary. It runs as user `canary` under AppArmor
  confinement.
- `log-checker` is a standalone bash script invoked via `leaprun` (privilege escalation wrapper).
- Configuration is sourced from `/etc/systemcheck.d/*.conf` and `/usr/local/etc/systemcheck.d/*.conf`.

## sysmaint session detection

- `booted_in_sysmaint_session` (preparation.bsh `sysmaint_state_detection`) = kernel cmdline
  has `boot-role=sysmaint`. Boot INTENT only, not the account actually reached.
- Actual landed account = `who_ami` (set in `preparation`). The privileged account is
  `sysmaint` (uid 1001); the unprivileged account is `user` (uid 1000). On Qubes a sysmaint
  boot sets BOTH `boot-mode.kernelopts.sysmaint='boot-role=sysmaint ...'` (so the cmdline flag
  appears there too) and `boot-mode.default-user.sysmaint='sysmaint'`; they move together.
- Two predicates, do not conflate:
  - `running_as_sysmaint` = `who_ami==sysmaint`. The account has sudo. Used only by the
    mismatch check (and by `in_working_sysmaint_session`).
  - `in_working_sysmaint_session` = `booted_in_sysmaint_session==true AND running_as_sysmaint`.
    A real SYSMAINT Session, the ONLY state where the System Maintenance Panel exists (the
    panel lives in the sysmaint session, set up on a `boot-role=sysmaint` boot; the account
    alone -- console su, manual default-user -- has sudo but no panel). Gate ALL "click Open
    Terminal in the System Maintenance Panel" guidance on this (`remediation_instructions`,
    `check_operating_system`), never on the account alone.
- `check_sysmaint_boot_role_mismatch` warns (grave popup) on EITHER direction of
  role-vs-account mismatch: SYSMAINT requested but landed as `user` (no sudo though expected),
  or USER requested but landed as `sysmaint` (holds sudo though a locked-down session was
  expected). Gated on `user_sysmaint_split_installed` (no split package, no mismatch to warn
  about). The factual session line reports by the REQUESTED boot role (the SYSMAINT/USER
  Session labels ARE the boot-menu names); it does NOT restate the mismatch at info severity.
- Autostart/skip gating is an UNANCHORED substring match
  (`case ${systemcheck_autostart_functions} in *"$@"*`), so a registered function name must
  not contain another check's name as a substring -- otherwise registering/skipping one drags
  the other in. `check_sysmaint_boot_role_mismatch` is deliberately NOT prefixed
  `check_user_sysmaint_split` for this reason.

## Autostart grave popups

- Pattern: `check_unrestricted_mode_in_template.bsh`, `check_sysmaint_boot_role_mismatch`.
- Register `systemcheck_autostart_functions+=" <fn> "` at fragment top AND invoke via
  `systemcheck_run_function <fn>` in the main `systemcheck` script. The list only GATES which
  functions run in autostart; both are required.
- Emit nothing unless the grave condition holds (keeps autostart quiet otherwise).
- GUI: `${output_x} "${output_opts[@]}" --passivepopupqueuex --passivepopupqueuextitle
  "${passive_popup_title}" --typex error --message "${MSG}" --done`. CLI: `${output_cli} ...
  --messagecli --typecli error --message "${MSG}"`. msgcollector supports `error` severity.

## Security review guidelines

See `agents/security.md`.
