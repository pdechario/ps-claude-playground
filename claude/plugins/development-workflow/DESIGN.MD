# development-workflow — Design Spec

A Python CLI tool (using the Anthropic SDK directly) that guides software development through a structured, spec-driven workflow. Covers new feature development and bug fixes. Exposes 5 steps, each representing a phase. State persists in JSON files so work survives session restarts. Intended to be open-sourced.

Claude Code integration (slash commands) is an optional layer on top — the core tool runs standalone from any terminal.

---

## How It Works

`workflow.py` is the entrypoint — a Python CLI that calls the Anthropic SDK directly. It manages state, selects the model per step, builds the prompt (injecting only the fields needed from prior steps), calls `anthropic.Anthropic().messages.create(...)`, and writes output JSON. No Claude Code dependency required.

Tools needed by each step (file reads, bash commands, git diff) are implemented as Anthropic SDK tool definitions — Python handles the tool call loop itself.

```
terminal
    $ python workflow.py step context
    ↓ Anthropic SDK (model: Haiku) + custom tools
    ↓ writes .claude/workflow/context.json
    → displays output, prompts user approval / feedback loop

    $ python workflow.py step spec
    ↓ reads context.json (selective fields only)
    ↓ Anthropic SDK (model: Sonnet)
    ↓ writes spec.json
    ...and so on
```

Claude Code integration is optional: a `plugin.yaml` can register these as slash commands so users can run `/design-context` from the IDE, but the Python CLI works without it.

---

## Directory Layout

```
claude/plugins/development-workflow/
├── DESIGN.MD                    ← this file
├── pyproject.toml               ← package definition, deps (anthropic, click/typer)
├── workflow.py                  ← CLI entry: `python workflow.py step <name>`
├── state.py                     ← read/write .claude/workflow/*.json
├── models.py                    ← model selection + prompt caching config
├── tools.py                     ← Anthropic SDK tool definitions (read_file, run_bash, git_diff)
├── steps/
│   ├── context.py
│   ├── spec.py
│   ├── tests.py
│   ├── review.py
│   └── merge.py
├── prompts/
│   ├── context.md               ← system prompt template for each step
│   ├── spec.md
│   ├── tests.md
│   ├── review.md
│   └── merge.md
└── plugin.yaml                  ← optional: Claude Code slash command definitions
```

### State Files (live in the project being worked on)

```
<project-root>/.claude/workflow/
├── workflow.json       ← overall state (current step, step statuses, history)
├── context.json        ← output of /design-context
├── spec.json           ← output of /design-spec
├── tests.json          ← output of /design-tests
├── review.json         ← output of /review-iterate-commit
└── merge.json          ← output of /merge
```

---

## Workflow Navigation (Doubly-Linked List)

Steps are a linked list: `context ↔ spec ↔ tests ↔ review ↔ merge`

### Forward (normal)
After each step completes, the orchestrator suggests the next. User confirms before advancing.

### Backward (revision)
Going back from step N to step M:
1. Mark step M as `in_progress`
2. Mark all steps after M as `pending` (invalidated — their JSON is stale)
3. User re-runs step M and approves the new output
4. Orchestrator prompts to re-run each invalidated step in order (or auto-runs them)

### Within-step iteration
Each step runs a refinement loop before writing JSON:
1. Claude produces candidate output
2. User reads it; can provide feedback ("narrow the scope to auth only")
3. Claude refines — same step, same model, prior attempt stays in context
4. Loop exits on user approval (`approve` / `done` / empty enter)
5. JSON is written only on approval

---

## The 5 Steps

Steps are invoked as `python workflow.py step <name>`. The `/design-*` names are the user-facing aliases (also used as Claude Code slash commands if `plugin.yaml` is installed).

### 1. `context` (`/design-context`)
**Purpose:** Understand the problem space before any design decisions.

**Inputs (user provides at invocation):**
- Brief description of what they want to build or fix
- Optionally: ticket/issue reference

**Claude tasks:**
- Explore the codebase to find affected files, services, utilities
- Identify what must NOT be broken (blast radius)
- Surface ambiguities and open questions
- Propose 2–3 high-level prototype options with tradeoffs

**Output schema — `context.json`:**
```json
{
  "problem_statement": "string",
  "affected_files": ["path/to/file.py"],
  "connected_services": ["ServiceName"],
  "constraints": ["must not break existing session tokens"],
  "open_questions": ["should we support OAuth?"],
  "prototype_options": [
    {
      "name": "Option A",
      "summary": "string",
      "tradeoffs": "string"
    }
  ],
  "step_status": "complete",
  "timestamp": "ISO8601"
}
```

**Model:** `claude-haiku-4-5-20251001` — fast and cheap for codebase exploration

---

### 2. `spec` (`/design-spec`)
**Purpose:** Make explicit design decisions based on the gathered context.

**Reads from:** `context.json` (fields: `problem_statement`, `open_questions`, `prototype_options`, `constraints`)

**Claude tasks:**
- Resolve the open questions from context
- Select one prototype option with justification
- Define data models, API contracts, interfaces
- Note edge cases that must be handled
- Call out anything deferred to a future iteration

**Output schema — `spec.json`:**
```json
{
  "chosen_approach": "Option A",
  "rationale": "string",
  "data_models": {},
  "api_contracts": ["POST /api/foo → { bar: string }"],
  "edge_cases": ["empty payload", "concurrent writes"],
  "out_of_scope": ["pagination"],
  "open_items": [],
  "step_status": "complete",
  "timestamp": "ISO8601"
}
```

**Model:** `claude-sonnet-4-6` — complex reasoning required

---

### 3. `tests` (`/design-tests`)
**Purpose:** Define the test strategy before implementation begins.

**Reads from:** `context.json` + `spec.json` (selective fields)

**Claude tasks:**
- Identify testing levels needed (unit / integration / e2e)
- Generate concrete test case descriptions for happy path and all edge cases from spec
- Flag dependencies that need mocking vs real implementation
- Note what is hard to test and why

**Output schema — `tests.json`:**
```json
{
  "unit_tests": [
    {
      "name": "returns 401 for missing token",
      "target": "src/auth/login.py",
      "notes": "string"
    }
  ],
  "integration_tests": [],
  "e2e_tests": [],
  "mocking_required": ["EmailService"],
  "hard_to_test": ["race condition in session expiry — needs mock clock"],
  "step_status": "complete",
  "timestamp": "ISO8601"
}
```

**Model:** `claude-sonnet-4-6`

---

### 4. `review` (`/review-iterate-commit`)
**Purpose:** Pre-commit review — catch regressions, version issues, linting issues; draft the commit.

**Reads from:** `spec.json` + `tests.json` + live `git diff`

**Claude tasks:**
- Run linting (via `run_bash` tool definition)
- Review the diff against the spec (did we implement what we designed?)
- Check for version upgrade concerns (dep bumps, breaking API changes)
- Draft a commit message
- List any blockers that should prevent commit

**Output schema — `review.json`:**
```json
{
  "linting_status": "pass | fail",
  "linting_issues": [],
  "spec_adherence": "full | partial | diverged",
  "divergences": ["added extra param not in spec: reason X"],
  "version_concerns": ["upgraded pydantic 1→2: validators are breaking"],
  "blockers": [],
  "commit_message": "feat: add OAuth login support\n\nLong description...",
  "step_status": "complete",
  "timestamp": "ISO8601"
}
```

**Model:** `claude-haiku-4-5-20251001` for mechanical checks; auto-escalate to `claude-sonnet-4-6` if divergences or blockers are found

---

### 5. `merge` (`/merge`)
**Purpose:** Wrap up — update changelog, docs, Claude context; verify nothing was left behind.

**Reads from:** all prior JSON files

**Claude tasks:**
- Write or append a `CHANGELOG.md` entry
- Update relevant documentation files
- Add/update a `CLAUDE.md` section describing what this change does and where to find it
- Scan for stale TODOs or commented-out code
- Draft PR description

**Output schema — `merge.json`:**
```json
{
  "changelog_entry": "### vX.Y.Z\n- Added OAuth login...",
  "docs_updated": ["docs/auth.md"],
  "claude_md_section": "string",
  "stale_todos": [],
  "pr_description_draft": "string",
  "step_status": "complete",
  "timestamp": "ISO8601"
}
```

**Model:** `claude-haiku-4-5-20251001`

---

## Token Efficiency

| Technique | How |
|-----------|-----|
| Selective injection | Each step reads only the fields it needs from prior JSONs, not the full file |
| Prompt caching | System prompts (step instructions, style guides) use `cache_control` header — stable across iterations |
| Model tiering | Haiku for exploration/mechanical tasks, Sonnet for reasoning (see table below) |
| Summarization hook | If a prior JSON exceeds a token threshold, Python runs a Haiku summarization pass before injecting it |

### Model Tier Table

| Step | Model | Reason |
|------|-------|--------|
| context | `claude-haiku-4-5-20251001` | Fast codebase exploration |
| spec | `claude-sonnet-4-6` | Complex design reasoning |
| tests | `claude-sonnet-4-6` | Deep edge case coverage |
| review | `claude-haiku-4-5-20251001` (default) | Mechanical; escalates to Sonnet on divergence |
| merge | `claude-haiku-4-5-20251001` | Doc formatting |

---

## Open Questions (to resolve before implementation)

1. **Tool scope per step:** Which tools does each step need? All steps likely need `read_file` and `run_bash`. Only `context` needs broad file traversal; `review` needs `git_diff`. Define the minimal tool set per step to avoid unnecessary surface area.

2. **Distribution:** Install via `pip install -e .` from the plugin directory, or publish to PyPI? For open-source sharing, PyPI is better UX. For now, local editable install is fine.

3. **API key:** Read `ANTHROPIC_API_KEY` from env var; fall back to `.env` in the project root. Document this clearly in setup instructions.

4. **Claude Code integration:** Verify how `plugin.yaml` should reference the CLI command so slash commands work correctly from the IDE. This is secondary — solve after the CLI works standalone.

---

## First Steps (Implementation Order)

1. Set up `pyproject.toml` with `anthropic` and `click` (or `typer`) as deps; verify a basic SDK call works
2. Scaffold `state.py` — the state manager is the foundation everything else depends on
3. Define `tools.py` — implement `read_file`, `run_bash`, `git_diff` as Anthropic SDK tool definitions
4. Build `context` step end-to-end — one working step as the template for the rest
5. Write `prompts/context.md` — iterate on the system prompt with real usage before generalizing
6. Add the remaining 4 steps following the same pattern
7. Implement backward navigation and within-step iteration loop
8. Token optimization pass — selective field injection, prompt caching, summarization hook
9. Write setup docs — install instructions, `ANTHROPIC_API_KEY` config, optional Claude Code integration

---

## Verification

- Run `python workflow.py step context` on a real small feature; confirm `context.json` is written with correct schema
- Run `python workflow.py step spec`; confirm it reads only the needed fields from `context.json` and produces valid `spec.json`
- Navigate backward: re-run `context` step, verify all downstream JSONs are marked `pending`
- Test iteration loop: give feedback mid-step, verify Claude refines without writing JSON until approved
- Cost check: run a full workflow end-to-end, inspect Anthropic API logs for model tiers and cache hit rate
