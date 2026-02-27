# G-Claudex
```
Gemini 3.1 Pro (Plan/Orchestrate)
        │
        ▼  writes → /tmp/session/PLAN.md + CONTEXT.md
        │
   ┌────┴────┐
   ▼         ▼  (sequential, not parallel)
Claude Opus  Codex High
(write)      (review)
   │              │
   └──────┬───────┘
          ▼
    /tmp/session/REVIEW.md
          │
          ▼
   Gemini reads, decides next step
```

Created for Codex CLI + Claude Code CLI + Antigravity (main brain) / cursor / zed or similar 
