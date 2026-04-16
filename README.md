# marketing-sops

Marketing SOPs and Claude Code cloud routines for **JumpSquat Marketing LLC**.

## Layout

```
sops/        # One subfolder per SOP. Contains prompt, templates, reference files.
active/      # Per-routine output — one subfolder per fire. Committed on claude/* branches.
```

## Routines in this repo

| Routine | Folder | Trigger type |
|---|---|---|
| Foundational Docs Builder | `sops/foundational-docs/` | API (optional Make.com webhook) |

## How routines use this repo

Each Claude Code cloud routine configured against this repo:
1. Clones the repo in a fresh Anthropic-managed cloud environment.
2. Reads its prompt + templates + reference files from `sops/<routine-name>/`.
3. Writes its outputs to `active/<routine-name>/<subject>-<date>/`.
4. Commits to a `claude/<routine-name>-<subject>-<date>` branch and pushes.

See `sops/<routine-name>/README.md` for per-routine details.
