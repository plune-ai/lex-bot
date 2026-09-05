# ADR-0021: A human-authored case is a first-class input; the platform boundary stays where it is

- **Status:** Accepted · **Date:** 2026-09-02
- **Decision in code:** *not yet* — this ADR precedes the implementation (planned as `cairn automate --case <file>`, see umbrella plan `AI-documents/plune/plune-ai/06-plan.md`, epic C-1)
- **Relationship to prior ADRs:** extends ADR-0010 (Cairn does not know about the platform) and ADR-0014 (versioned artifact contract, `stableId`); does not change ADR-0005 (output is `@playwright/test`, POM + `getByRole`)
- **Platform side:** Plune ADR 0024 (platform invokes Cairn as a tool)

## Context

Cairn's inputs are a URL (UI modality) or an OpenAPI document (API modality). Cairn designs the
cases itself. The Plune platform now wants the reverse direction as well: a person writes a
scenario — Gherkin-style or free-form Markdown — and Cairn turns it into a Playwright spec, runs
it, heals it, and hands the result back for review.

The tempting shape is "Cairn as a bot inside the platform": read cases from the API, write
proposals into the review queue. That breaks ADR-0010 — the rule that keeps Cairn useful without
an account, keeps the contract from drifting from two sides, and keeps a single ingestion path
(`plune ingest`, Plune ADR-CI-01).

The market went through this already. Tools that started with "natural language executed by an
LLM at run time" (testRigor, early Momentic) converged on "a human approves the plan → locators
are resolved and cached → real code is exported to the repo" (KaneAI, Shiplight, Checksum,
Playwright's own planner/generator/healer). That is Cairn's existing architecture; the only
missing piece is the input.

## Decision

1. **A human-authored case is a third input**, next to URL and OpenAPI. Cairn reads it from a
   file (the same Markdown case format Cairn already emits, ADR-0014, with `style: gherkin | free`),
   never from a network.
2. **The boundary from ADR-0010 is unchanged.** No `PLUNE_*` variable, no token, no HTTP call to
   the platform. The platform (through `@plune-ai/cli`) writes the case file, invokes Cairn,
   and ingests `runs/<id>/`. Direction of dependency: platform → Cairn.
3. **Steps compile to cached, deterministic actions.** Each scenario step maps to a
   `test.step()` whose locator is resolved once (intent + resolved locator stored alongside the
   spec) and re-resolved only on failure. The LLM is used at generation and repair time, never on
   the green path.
4. **Repo-resident project memory is an input, not a service.** Cairn reads
   `plune/memory/*.md` (or `CAIRN.md`) from the target repo — conventions, data sources, page
   objects, prohibitions — the same way Playwright agents read `.github/` agent files. It does
   not write there; the platform proposes memory changes through review.
5. **Healing stays conservative and visible.** The healer reads `git diff base..head` before
   deciding intent vs. drift; unfixable tests get `test.fixme` with a diagnosis; every heal is a
   diff in `runs/<id>/`, never a silent rewrite.

## Rejected

- *Cairn talks to the platform API.* Breaks ADR-0010; makes the account mandatory; duplicates
  the ingestion path.
- *Execute natural-language steps at run time.* Non-deterministic, slow, expensive; the
  category moved away from it.
- *Strict Gherkin with step definitions as the only style.* Keeps the "step-definition tax"
  users complain about (BlinqIO); free-form Markdown stays the default, Gherkin is a style.

## Consequences

- New CLI surface: `cairn automate --case <file> [--repo <dir>] [--memory <dir>]`; output
  contract unchanged (`report.json`, `testcases/`, `tests/`, `stableId` carried through).
- Traceability gains one link: scenario step ↔ `test.step()` ↔ resolved locator.
- The ISO 29119-4 design layer is bypassed for human cases by default (the human already
  designed the case); `--gaps` can still propose additional cases through the same review path.
