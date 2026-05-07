---
name: japan-trip-deploy
description: >-
  Run the project's review-commit tool to commit unstaged changes, push to the
  remote, and/or deploy to the Raspberry Pi via SSH + docker compose. Use when
  the user says "commit et", "kaydet", "pushla", "yükle", "deploy et",
  "Pi'ye gönder", "Pi'ye deploy et", "en son değişiklikleri gönder",
  "hepsini yap", or any English equivalent like "commit and deploy",
  "push to pi", "ship it". Lives at tools/review-commit/.
---

# japan-trip-deploy

The repo has a custom CLI agent at `tools/review-commit/` that handles the full
commit → push → Pi deploy chain. It uses local Ollama (qwen2.5:3b) for AI, so
it is free to run.

## When to use

User intent → action chain:

| User says | Action plan |
|---|---|
| `commit`, `kaydet` | commit only |
| `push`, `yükle` | push only |
| `deploy`, `Pi'ye deploy` | deploy only |
| `pushla` (after committed work) | push + deploy |
| **`en son değişiklikleri gönder`** | **commit + push + deploy** |
| `Pi'ye gönder`, `hepsini yap` | commit + push + deploy |

The tool itself recognises these phrases via keyword matching, so just pass
the user's command verbatim.

## How to invoke

Always run from the tool's directory with `--yes` for non-interactive mode:

```bash
cd tools/review-commit && npm run review -- --yes "<user's command>"
```

Examples:

```bash
cd tools/review-commit && npm run review -- --yes "deploy"
cd tools/review-commit && npm run review -- --yes "en son değişiklikleri gönder"
cd tools/review-commit && npm run review -- --yes "commit"
```

`--yes` auto-confirms every prompt:
- plan approval → yes
- commit message approval → yes (uses AI-generated message as-is)
- push branch confirmation → yes

## Workflow

1. **Confirm with the user first** when the action involves push or deploy.
   These touch remote systems (GitHub, the Pi). Use `AskQuestion` with the
   plan summary before invoking the tool. Skip confirmation for plain
   `commit`-only requests — those are local and reversible.

2. **Show staged changes preview** if the user is unsure:

   ```bash
   git -C /Users/sevimm/Documents/Projects/japan-trip status --short
   ```

3. **Invoke the tool** with `Shell`:

   ```bash
   cd tools/review-commit && npm run review -- --yes "<command>"
   ```

   - `block_until_ms: 120000` (Pi deploy + docker rebuild can take ~60s)
   - The tool streams Pi docker output; surface the final lines to the user.

4. **Report the result**:
   - Success: short summary (commit hash if committed, "deployed to Pi" if deployed).
   - Failure: include the exit code and last 10 lines of stderr. Exit codes:

     | Code | Meaning |
     |---|---|
     | 0 | success / user cancelled |
     | 1 | provider not ready (Ollama down, no model) |
     | 2 | AI run error |
     | 3 | git commit failed |
     | 4 | git push failed |
     | 5 | SSH / Pi deploy failed |
     | 6 | Pi config missing in `.env` |

## When NOT to invoke automatically

- User asks "what would happen if I deploy?" → **explain**, don't run.
- User asks to **review** changes without deploying → run `git diff` and
  summarise; don't call the tool.
- User specifies a different commit message format or wants to write the
  message themselves → **don't use `--yes`**. Run interactively in their
  terminal instead and tell them: `cd tools/review-commit && npm run review`.
- No changes in the working tree and the request is `commit` only → tell the
  user there's nothing to commit instead of invoking the tool.

## Tool internals (for context only)

- Provider: Ollama by default (`AI_PROVIDER=ollama`, `OLLAMA_MODEL=qwen2.5:3b`).
- Pi target: `mennano@192.168.1.60:agora-voice-chatbot-web` (from `.env`).
- Deploy command: `git pull origin main && docker compose down && docker compose up -d --build`.
- Source: `tools/review-commit/review.ts` (entry), `ai.ts` (provider), `deploy.ts` (SSH).

To switch to Cursor SDK instead of Ollama: edit `.env` → `AI_PROVIDER=cursor`
and add `CURSOR_API_KEY`.
