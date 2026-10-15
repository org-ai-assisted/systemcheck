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
- Actual landed account = `who_ami` (set in `preparation`). Working sysmaint session runs as
  account `sysmaint` (uid 1001); unprivileged account is `user` (uid 1000). On Qubes the
  sysmaint boot mode sets `boot-mode.default-user.sysmaint='sysmaint'`, so `who_ami` applies
  there too.
- `in_working_sysmaint_session` = `booted_in_sysmaint_session==true AND who_ami==sysmaint`.
  Use it, not the cmdline flag alone, for any "panel / sudo available" guidance. Requested but
  landed as `user` is a grave silent state: no sudo though the user believes otherwise.

## Autostart grave popups

- Pattern: `check_unrestricted_mode_in_template.bsh`, `check_user_sysmaint_split_boot_mismatch`.
- Register `systemcheck_autostart_functions+=" <fn> "` at fragment top AND invoke via
  `systemcheck_run_function <fn>` in the main `systemcheck` script. The list only GATES which
  functions run in autostart; both are required.
- Emit nothing unless the grave condition holds (keeps autostart quiet otherwise).
- GUI: `${output_x} "${output_opts[@]}" --passivepopupqueuex --passivepopupqueuextitle
  "${passive_popup_title}" --typex error --message "${MSG}" --done`. CLI: `${output_cli} ...
  --messagecli --typecli error --message "${MSG}"`. msgcollector supports `error` severity.

## Security review guidelines

See `agents/security.md`.
