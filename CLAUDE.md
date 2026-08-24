# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commit message conventions

Work in this repo is tracked in YouTrack (project `CSPROD`, `Subsystem: firmware`). The YouTrack VCS
integration is configured against this repo (`blueguardian-co/cerebralstratum-firmware`). Reference the
relevant issue in every commit so YouTrack can link and transition it automatically:

- Always mention the issue ID somewhere in the commit message (e.g. `CSPROD-XXX Fix ...`) to link the
  commit.
- To also transition the issue's state, append a command line: `#<ISSUE-ID> <State>` (e.g. `#CSPROD-XXX
  Fixed` for a Bug Fix issue, `#CSPROD-XXX Done` for a Task) — the state value must match one of the
  issue's actual `State` field options in YouTrack. The command must be on its own line and not wrap.
- If a commit closes more than one issue, use `(#CSPROD-1, #CSPROD-2) Fixed`.
- This requires the committer's git email to be a member of the VCS integration's Committers group in
  YouTrack (Project Settings → VCS Repositories) — otherwise the commit still links but the state
  command is silently ignored.