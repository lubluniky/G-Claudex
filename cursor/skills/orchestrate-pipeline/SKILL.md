---
name: orchestrate-pipeline
description: Three-role headless pipeline — Gemini orchestrates, Claude Opus writes code, Codex reviews. Use when the user asks to run the full agent pipeline, delegate implementation+review, or orchestrate multi-agent work.
version: 2.0.0
---

# Orchestration Pipeline

You are the **Planner and Orchestrator** (Gemini 3.1 Pro).
You NEVER write code yourself. You think, plan, delegate, and synthesize results.

---

## Architecture

```
Gemini 3.1 Pro (Plan/Orchestrate)
        │
        ▼  writes → /tmp/session/$TASK_ID/PLAN.md + CONTEXT.md
        │
   ┌────┴────┐  sequential, not parallel
   ▼         ▼
Claude Opus  Codex
(implement)  (review, read-only)
   │              │
   └──────┬───────┘
          ▼
    /tmp/session/$TASK_ID/review.md
          │
          ▼
   Gemini reads, decides next step
```

Context is shared via `/tmp/session/<task_id>/` — reliable without MCP.

---

## Session Bootstrap

```bash
TASK_ID=$(date +%Y%m%d_%H%M%S)
SESSION_DIR="/tmp/session/$TASK_ID"
mkdir -p "$SESSION_DIR"
echo "1" > "$SESSION_DIR/iteration.txt"
```

Directory layout:
```
$SESSION_DIR/
  PLAN.md           ← you write (Gemini)
  CONTEXT.md        ← accumulated shared memory across iterations
  claude_raw.json   ← raw Claude JSON output (contains session_id)
  claude_output.md  ← extracted Claude result
  codex_prompt.md   ← assembled prompt for Codex
  review.md         ← Codex review output
  CLAUDE_SESSION    ← stored session_id for claude --resume
  iteration.txt     ← current iteration counter (max 3)
```

---

## Phase 1 — Plan (Gemini writes PLAN.md)

Write PLAN.md before invoking any subagent. This is the contract between agents.

```bash
cat > "$SESSION_DIR/PLAN.md" << 'EOF'
# Task Plan

## Objective
<one sentence — what needs to be built or changed>

## Constraints
<technical constraints, stack, style rules>

## Steps for Claude (implement)
1. <specific step>
2. <specific step>

## Acceptance Criteria for Codex (review)
- [ ] <verifiable criterion>
- [ ] <verifiable criterion>

## Files in Scope
- path/to/file.ext
EOF
```

---

## Phase 2 — Claude Opus (Code Writer)

Claude has full tool access: Bash, Read, Edit, Write.

### First run (new session):
```bash
CONTEXT=$(cat "$SESSION_DIR/CONTEXT.md" 2>/dev/null)
PLAN=$(cat "$SESSION_DIR/PLAN.md")
PROMPT="${CONTEXT:+$CONTEXT

---

}$PLAN"

claude -p "$PROMPT" \
  --model claude-opus-4-6 \
  --output-format json \
  --allowedTools "Bash,Read,Edit,Write" \
  > "$SESSION_DIR/claude_raw.json" 2>&1

# Store session_id for resume on follow-up turns
jq -r '.session_id' "$SESSION_DIR/claude_raw.json" > "$SESSION_DIR/CLAUDE_SESSION"
jq -r '.result'     "$SESSION_DIR/claude_raw.json" > "$SESSION_DIR/claude_output.md"
```

### Resume existing session (follow-up turns):
```bash
CLAUDE_SESSION_ID=$(cat "$SESSION_DIR/CLAUDE_SESSION")

claude -p "<follow-up instruction from review FAILED items>" \
  --model claude-opus-4-6 \
  --resume "$CLAUDE_SESSION_ID" \
  --output-format json \
  --allowedTools "Bash,Read,Edit,Write" \
  > "$SESSION_DIR/claude_raw.json" 2>&1

jq -r '.result' "$SESSION_DIR/claude_raw.json" > "$SESSION_DIR/claude_output.md"
```

### Error handling:
```bash
IS_ERROR=$(jq -r '.is_error' "$SESSION_DIR/claude_raw.json")
if [ "$IS_ERROR" = "true" ]; then
  # Downgrade to sonnet and retry once (no resume — different model)
  claude -p "$PROMPT" \
    --model claude-sonnet-4-6 \
    --output-format json \
    --allowedTools "Bash,Read,Edit,Write" \
    > "$SESSION_DIR/claude_raw.json" 2>&1

  jq -r '.session_id' "$SESSION_DIR/claude_raw.json" > "$SESSION_DIR/CLAUDE_SESSION"
  jq -r '.result'     "$SESSION_DIR/claude_raw.json" > "$SESSION_DIR/claude_output.md"
fi
```

---

## Phase 3 — Codex (Reviewer, read-only sandbox)

Codex is **stateless by design** — pass full context every time. No resume.

```bash
cat > "$SESSION_DIR/codex_prompt.md" << EOF
# Code Review Task

## Original Plan
$(cat "$SESSION_DIR/PLAN.md")

## Claude's Implementation Output
$(cat "$SESSION_DIR/claude_output.md")

## Your Task
Review the implementation against the acceptance criteria above.
Return structured feedback only. Do NOT rewrite or modify any files.

### Required output format:
- PASSED: criteria that pass
- FAILED: criteria that fail (with file:line references where possible)
- SUGGESTIONS: optional non-blocking improvements
- VERDICT: APPROVE | REQUEST_CHANGES
EOF

codex exec \
  -m gpt-5 \
  -c 'model_reasoning_effort="high"' \
  --sandbox read-only \
  --ephemeral \
  --output-last-message "$SESSION_DIR/review.md" \
  "$(cat "$SESSION_DIR/codex_prompt.md")"
```

> **Notes:**
> - `--output-last-message` writes via the CLI process, not the sandboxed agent — so `read-only` sandbox does not block it.
> - `/tmp/session/...` is readable by the sandbox (read-only means no writes from inside the agent, not from the CLI wrapper).
> - For large prompts pass via stdin: replace the last argument with `-` and pipe: `... --output-last-message "$SESSION_DIR/review.md" - < "$SESSION_DIR/codex_prompt.md"`
> - `--ephemeral` suppresses session rollout files on disk — correct for stateless reviewer role.

---

## Phase 4 — Gemini Synthesizes

Read all outputs and decide:

```bash
echo "=== PLAN ===" && cat "$SESSION_DIR/PLAN.md"
echo "=== CLAUDE OUTPUT ===" && cat "$SESSION_DIR/claude_output.md"
echo "=== CODEX REVIEW ===" && cat "$SESSION_DIR/review.md"

VERDICT=$(grep "VERDICT" "$SESSION_DIR/review.md" | awk '{print $NF}')
ITERATION=$(cat "$SESSION_DIR/iteration.txt")
```

### Decision logic:

**VERDICT = APPROVE** → task done.
```bash
# Update shared memory
cat >> "$SESSION_DIR/CONTEXT.md" << EOF

## Iteration $ITERATION — $(date)
### What was done
$(head -20 "$SESSION_DIR/claude_output.md")

### Verdict
APPROVE
EOF

echo "Task complete after $ITERATION iteration(s)."
```

**VERDICT = REQUEST_CHANGES** and iteration < 3 → loop back to Phase 2.
```bash
NEXT=$((ITERATION + 1))
echo "$NEXT" > "$SESSION_DIR/iteration.txt"

# Extract FAILED items from review.md and write a targeted follow-up prompt
FAILED=$(awk '/^- FAILED:/,/^- [A-Z]/' "$SESSION_DIR/review.md" | grep -v "^- [A-Z]" | head -20)

# Resume Claude with correction instruction (reuses session context)
claude -p "Fix the following review failures:

$FAILED

Do not change anything that passed review." \
  --model claude-opus-4-6 \
  --resume "$(cat "$SESSION_DIR/CLAUDE_SESSION")" \
  --output-format json \
  --allowedTools "Bash,Read,Edit,Write" \
  > "$SESSION_DIR/claude_raw.json" 2>&1

jq -r '.result' "$SESSION_DIR/claude_raw.json" > "$SESSION_DIR/claude_output.md"
# Then re-run Phase 3 (Codex review)
```

**Iteration ≥ 3 and still REQUEST_CHANGES** → escalate to user.
```bash
echo "ESCALATE: 3 iterations exhausted. Manual intervention required."
echo "Last review:"
cat "$SESSION_DIR/review.md"
```

---

## Context Passing Rules

| What                  | How                                                         |
|-----------------------|-------------------------------------------------------------|
| Gemini → Claude       | Prepend `CONTEXT.md` + `PLAN.md` into prompt               |
| Claude turn N → N+1   | `--resume <session_id>` — native session memory             |
| Gemini → Codex        | `codex_prompt.md` contains full plan + Claude output        |
| Codex is stateless    | Always pass full context in prompt; never use resume        |
| Cross-task memory     | `CONTEXT.md` accumulates — prepend at start of each task   |

---

## Invariants (never break these)

- NEVER run `claude` without `--output-format json` — you need `session_id`
- NEVER run Codex without `--sandbox read-only` in review role
- NEVER skip writing `PLAN.md` — it is the shared contract between agents
- ALWAYS check `is_error` before reading `.result` from Claude JSON
- ALWAYS update `CONTEXT.md` after a review-approved cycle
- NEVER loop more than 3 iterations — escalate to user on failure
- Codex is stateless — no resume, pass everything in prompt each time
- Claude model IDs: `claude-opus-4-6` (primary), `claude-sonnet-4-6` (fallback)
- Codex: `-m gpt-5 -c 'model_reasoning_effort="high"'` — model and effort are separate flags
- Codex stdout is dirty by default — always use `--output-last-message <file>` to capture clean result
