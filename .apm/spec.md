---
title: APM x skogai-routing @-link Integration
modified: Spec creation by the Planner.
---

# APM Spec

## Overview

This project customizes the `agentic-pm` (APM) CLI's own Claude Code template output so that generated files reference other files using skogai-routing's `@`-link convention where warranted, instead of the plain-path-only strings APM's placeholder system currently emits. The core problem: APM's build-time placeholders (`{SKILL_PATH:}`, `{COMMAND_PATH:}`, `{AGENT_PATH:}`, `{GUIDE_PATH:}`) resolve to bare paths regardless of whether the referenced content should be eagerly loaded into an agent's context or merely listed for later discovery — a distinction skogai-routing's conventions already make deliberately. Scope is the Claude target only, executed as a full path through APM's own pipeline: template changes, build, validation, a real `git push`, and a manual GitHub Release trigger, published as a custom APM release consumable via `apm custom --repo skogai/skogai-pm`. Equally important as the deliverable itself: this run is the first real exercise of APM's own Planner → Manager → Worker coordination and handoff mechanism in the User's environment — success means both the template change landing correctly and the coordination process proving itself out.

## Workspace

- **Working repository:** `/home/skogix/skogai-pm` (single repo). `origin` = `git@github.com:skogai/skogai-pm.git`; `upstream` = `git@github.com:sdi2200262/agentic-project-management.git` (the official APM source). This repo is already shaped as a customization fork — `apm custom --repo skogai/skogai-pm` is the intended consumption path, not a new repository.
- **Build system (working target):** `build/index.js`, `build/build-config.json` (per-assistant target registry), `build/processors/placeholders.js` (placeholder resolution, the primary file this project modifies), `build/processors/templates.js`. `templates/**/*.md` are the build-time template sources; `.claude/`, `.agents/`, `.codex/`, `.apm/` at repo root are already-installed *output* from a prior `apm init` run, not sources to edit directly.
- **Reference (read-only, external):** the `skogai-routing` Claude Code plugin, cached at `~/.claude/plugins/cache/skogai-market/skogai-routing/88ad23fb764f/`. Its key reference docs — `skills/skogai-routing/references/at-linking.md` (`@`-link mechanics: eager-loaded at read time, real-filesystem source of truth, no wildcard globs, ~6 levels of recursive depth) and `skills/skogai-routing/references/claude-md-routing-rules.md` (rule 1: routers stay lightweight and route rather than contain; rule 2: `@path` = eager-loaded, plain `path` = listed for on-demand discovery; a router-vs-content-loader distinction) — are the authority for correctness on this project. This plugin path is cache-managed, not vendored into this repo; treat the path as a convenience pointer and prefer invoking the `skogai-routing` skill directly if the cached path has moved or changed.
- **Reference (read-only, external):** `skills/skogai-routing/schemas/` (`defs.schema.json`, `document.schema.json`, `router.schema.json`, etc.) and `skills/skogai-routing/scripts/validate_router.py` exist in the same plugin and describe a broader frontmatter+XML-section file-typing system. Explicitly out of scope for this project — noted here only so the placeholder mechanism below doesn't foreclose extending it later.
- **CI:** `.github/workflows/release-templates.yml` exists and is `workflow_dispatch`-only (manual trigger, optional version input, `permissions: contents: write`) — it does not fire on push. Publishing requires a push followed by a separate, explicit workflow trigger.
- **Access:** `gh` CLI is authenticated (account `Skogix`, `repo` scope) — sufficient for push and for triggering `workflow_dispatch`.
- **No existing `CLAUDE.md`** at repo root; created fresh during Rules Analysis.

---

> **Notes:** The User's stated priority is proving out APM's Planner/Manager/Worker coordination and handoff mechanism itself, with the `@`-link change as the vehicle — weigh the Manager's runtime judgment calls (dispatch, handoff timing, review cadence) accordingly rather than optimizing purely for shipping the template change fastest. The User explicitly declined to force an artificial Handoff or add a special approval gate before the release-workflow trigger; default framework behavior applies to both. The User favors linking to existing authoritative docs (here, `skogai-routing`'s own reference files) over writing new explanatory documentation — apply this same preference to any documentation this project produces.

## Design Decisions

### Placeholder mechanism: additive `_LINK` placeholders, not inferred or global

**Decision:** Introduce new placeholders — `{SKILL_LINK:name}`, `{COMMAND_LINK:name}`, `{GUIDE_LINK:name}`, `{AGENT_LINK:name}` — parallel to the existing `{SKILL_PATH:}` / `{COMMAND_PATH:}` / `{GUIDE_PATH:}` / `{AGENT_PATH:}` placeholders in `build/processors/placeholders.js`. For the `claude` target, a `_LINK` placeholder resolves to the same path as its `_PATH` counterpart, prefixed with `@` (e.g. `@.claude/skills/apm-communication/SKILL.md`). For every other target (copilot, antigravity, cursor, opencode, codex), `_LINK` resolves identically to `_PATH` (plain path) — `@`-linking is a Claude Code-specific mechanic per `at-linking.md`, so non-Claude output is unaffected by this project.

**Alternatives considered:**
- *Infer eager-vs-discovery automatically* (e.g. from file type or a naming convention) — rejected. Rule 2's actual test ("does the agent need this immediately vs. can it look it up") is a judgment call about each specific reference's role, not a mechanical property of the referenced file. Automating it would guess wrong in both directions.
- *Make all resolved paths `@`-links for the claude target* — rejected. Violates rule 2 directly: several existing placeholder usages (e.g. archive explorer agent config, guide cross-references consulted only on demand) are discovery references, and eagerly loading everything bloats every Worker's and Manager's context on every read.
- *Single placeholder with a mode argument* (e.g. `{SKILL_PATH:name:eager}`) — rejected in favor of the additive form. A parallel placeholder name keeps each template call site's intent legible at a glance without a third positional argument, and requires no change to the existing `_PATH` placeholders' behavior or call sites.

**Consequence:** deciding which existing `{X_PATH:name}` call sites in `templates/**/*.md` should become `{X_LINK:name}` is manual, per-site judgment against `claude-md-routing-rules.md` rule 2 and the router-vs-content-loader distinction — this classification work is the bulk of the implementation Task, not the placeholder mechanism itself.

### Scope boundary: mechanism + classification, not a routing-philosophy rewrite

**Decision:** This project changes the placeholder *mechanism* (how a reference is emitted) and the *classification* of existing reference call sites (which ones should be eager vs. discovery). It does not restructure APM's overall template/document organization to match skogai-routing's router-vs-content-loader philosophy more broadly (e.g. rewriting APM's own command files to be lightweight routers per rule 1). That broader alignment, along with the `defs.schema.json` type-schema extension noted under Workspace, is deliberately deferred.

## Publishing Path

**Decision:** The full pipeline is in scope, through to a real publish: edit `templates/**/*.md` and `build/processors/placeholders.js` → `npm run build:release` → validate the build output → commit and `git push` to `origin` (`skogai/skogai-pm`) → manually trigger `.github/workflows/release-templates.yml` via `workflow_dispatch` (a plain push does not do this) → resulting GitHub Release is consumable via `apm custom --repo skogai/skogai-pm`. No additional User approval gate is required before the workflow trigger beyond the Manager's default review of completed work — the User was explicit that this doesn't need special handling.

## Success Criteria

1. `_LINK` placeholders exist in `build/processors/placeholders.js` and resolve correctly per target (verified: `@`-prefixed for claude, identical to `_PATH` output for all other targets).
2. Existing `_PATH` call sites in `templates/**/*.md` have been reviewed against skogai-routing rule 2 and the router-vs-content-loader distinction; sites warranting eager loading are converted to `_LINK`.
3. `npm run build:release` succeeds and produces a `claude.zip` bundle whose extracted files contain the expected `@`-links at the expected locations and plain paths elsewhere.
4. Changes are pushed to `origin`, the release workflow is triggered, and the resulting release is installable via `apm custom --repo skogai/skogai-pm`.
5. The coordination process itself — Task assignment, any Handoffs that occur, Manager review, and reporting back to the User at natural stopping points — is legible and reported on, independent of whether the template change was technically difficult.
