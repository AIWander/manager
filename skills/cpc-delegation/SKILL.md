---
name: cpc-delegation
reference_tier: 1
description: 'Delegation discipline for CPC and CPC-style multi-AI orchestration. Use when: deciding whether to delegate code to another AI backend, choosing which backend, following archive-first protocol, or applying the 3Q extraction gate.'
---

## Core Rule

**>33 lines of code = delegate via manager. No exceptions.**

If you are about to write more than 33 lines of code inline in a chat response, stop. Submit it as a task to a dedicated backend instead. This keeps chat tokens for reasoning and lets backends do I/O-heavy work with full file access.

## When to Delegate vs Handle Inline

| Scenario | Action |
|----------|--------|
| < 10 lines, single file edit | Inline — just do it |
| 10-33 lines, straightforward | Inline — but consider delegating if multi-file |
| > 33 lines | Delegate — always |
| Multi-file changes | Delegate — even if total lines < 33 |
| Build/compile/test cycle needed | Delegate — backend has shell access |
| Research or analysis question | Inline — no code output needed |
| Quick one-shot answer | Inline or gemini_direct |

## Backend Selection

| Backend | Best For | Characteristics |
|---------|----------|----------------|
| claude_code | Multi-step coding, refactors, file system work | Full reasoning, file access, git, shell. Most capable. |
| codex | Single-file fast edits, quick fixes | Fast, focused, good for isolated changes |
| gemini_direct | One-shot answers, analysis, summaries | Quick, no file access, good for reasoning tasks |
| gpt | Text generation, analysis, alternative perspective | API-based, good for non-coding tasks |

**Default to claude_code** unless you have a specific reason to choose another backend.

## Archive-First Protocol

Before replacing or significantly modifying working code:

1. **Backup the target** — copy or archive the file/config being changed
2. **Verify the backup exists** — don't trust the operation succeeded without checking
3. **Execute the change** — now safe to modify
4. **Report what changed** — state what was backed up, what was modified, and where the backup lives

This applies to delegated work too. When submitting a task that modifies existing files, include archive-first instructions in the prompt.

## 3Q Gate for Extractions

Before extracting a pattern, correction, or discovery into the knowledge base, ask:

1. **Reusable?** — Will this apply to future situations, not just this one?
2. **Specific?** — Is it concrete enough to act on, not just a vague principle?
3. **New?** — Is this information not already captured in an existing Operating file?

**All three must be YES to extract.** Any NO = skip. This prevents knowledge base bloat.

## Anti-Patterns

- Writing 50+ line scripts inline in chat — delegate them
- Asking "would you like me to..." for reversible actions — just do it
- Delegating without context — always include working_dir, relevant file paths, and what success looks like
- Delegating without archive-first — if the task modifies existing files, include backup instructions
- Using the wrong backend — don't send a multi-file refactor to codex (use claude_code) or a quick answer to claude_code (use gemini_direct)
