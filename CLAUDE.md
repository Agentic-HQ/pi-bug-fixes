# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A documentation-only repo tracking bug fixes Steve contributes to the open source Pi ecosystem. There is no application code, no package manager, and no build, lint, or test commands. Deliverables are Markdown reports and analysis; the actual code fixes are submitted upstream as PRs, not committed here.

Upstream projects:
- `badlogic/pi-skills` — where issues are raised (e.g. issue #49, a medium-severity dependency security bug).
- `earendil-works/pi` — same maintainer; its `CONTRIBUTING.md` is treated as the governing contribution process for pi-skills too, since pi-skills has none of its own.

## Layout and conventions

- `docs/tickets/<upstream-repo>/<issue-number>/` — one folder per upstream issue being worked (currently `docs/tickets/pi-skills/49/`). All documentation belongs under `docs/`; only `README.md` stays at the root.
- Inside each ticket folder, numbered exchange files `NNN-<author>-<what>.md` (e.g. `001-steve-prompt.md` followed by `002-claude-report.md`) form an ordered conversation about that ticket. Continue the sequence rather than overwriting earlier files, and keep them in the ticket folder, not the repo root.

## Writing reports

Reports are read by Steve, a developer comfortable with TypeScript but not an expert. When a report is requested:
- Put a TL;DR summary at the top and a clickable table of contents beneath it.
- Aim for a ~15 minute read; leave detail out rather than compress it in.
- Explain TypeScript or dependency-management basics briefly where they affect the choice of fix.
- Before recommending a fix, check how the upstream repos manage dependencies and how comparable open source projects handle the same class of issue, and cite upstream files or issues by URL.
