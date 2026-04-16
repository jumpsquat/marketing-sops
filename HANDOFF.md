# Foundational Docs Builder — Routine Handoff

Paste this into a new routine at https://claude.ai/code/scheduled. That's the whole job.

## What this routine does

Given a brand name + a sales page URL + a category, Claude runs the full Foundational Docs SOP — reads the research methodology and the argument-structure framework from this repo, executes deep web research for the specific product, fills out the avatar sheet and offer brief templates, and writes a necessary-beliefs doc. All four finished documents land in the repo on a dated branch.

## Where the routine reads from (this repo)

| Path | Purpose |
|---|---|
| `sops/foundational-docs/prompt/routine-prompt.md` | The authoritative SOP — every step Claude follows on each fire |
| `sops/foundational-docs/reference/research-part-1.md` | Research methodology, part 1 |
| `sops/foundational-docs/reference/research-part-2.md` | Research methodology, part 2 |
| `sops/foundational-docs/reference/argument-structure-transcript.md` | Agora argument-structure teaching |
| `sops/foundational-docs/templates/avatar-sheet.md` | Avatar Sheet Template |
| `sops/foundational-docs/templates/offer-brief.md` | Offer Brief Template |

## Where the routine writes to (this repo)

Every fire produces a new dated folder under `active/foundational-docs/<brand-slug>-<YYYYMMDD>/` containing six files:

- `00-inputs.md` — the payload the routine was fired with
- `01-sales-page.md` — scraped sales page copy
- `02-research-doc.md` — deep research output (target 2,500+ words)
- `03-avatar-doc.md` — filled avatar sheet
- `04-offer-brief.md` — filled offer brief
- `05-necessary-beliefs.md` — up to 6 "I believe that…" statements

Those files are committed on a new branch: `claude/foundational-docs-<brand-slug>-<YYYYMMDD>`.

## Routine form — field values

| Field | Value |
|---|---|
| **Name** | `Foundational Docs Builder` |
| **Instructions** | paste the prompt below |
| **Model** | `Sonnet 4.6` |
| **Repositories** | `jumpsquat/marketing-sops` |
| **Environment** | `Default` |
| **Trigger** | `API` |
| **Connectors** | none |
| **Permissions → Allow unrestricted branch pushes** | OFF |

## The Instructions prompt (paste this into the routine)

```
You are the Foundational Docs Builder — a Claude Code cloud routine that produces four copywriting documents (Research Doc, Avatar Doc, Offer Brief, Necessary Beliefs Doc) for a new brand entering JumpSquat Marketing's copywriting pipeline.

Your full operating instructions live in this repository at:
  sops/foundational-docs/prompt/routine-prompt.md

On every fire:
1. Read that file with the Read tool immediately.
2. Follow every step in it exactly, using the BRAND / URL / SECONDARY_URL / CATEGORY payload appended to this message.
3. Commit your outputs to a branch named `claude/foundational-docs-<brand-slug>-<YYYYMMDD>` as specified in that file.
4. Print the summary from the Wrapup section to stdout.

The repo file is the source of truth. If any instruction here seems to conflict with it, defer to the repo file.
```

## Input payload format (what you send when firing)

```
BRAND: <brand name>
URL: <primary sales page URL>
SECONDARY_URL: <optional — second sales surface, e.g. ads landing page>
CATEGORY: <what the brand sells, in one short phrase>
```

`SECONDARY_URL` is optional. Everything else is required.

### First test (Eternal Life Tribe)

```
BRAND: Eternal Life Tribe
URL: https://masterclass.samuelbleemd.com/fb_completed
SECONDARY_URL: https://www.skool.com/eternallifetribe/about
CATEGORY: online holistic health / spiritual wellness community led by a medical doctor
```

## Iterating on the routine

The UI prompt is a thin bootstrap — it tells Claude to read the real SOP from the repo. To change routine behavior:

1. Edit `sops/foundational-docs/prompt/routine-prompt.md` locally
2. `git commit` + `git push`
3. The next fire picks up the change automatically

Same pattern for swapping templates (edit `sops/foundational-docs/templates/*.md`) or adding new reference material (drop files into `sops/foundational-docs/reference/` and add a `Read` instruction in the prompt file).

## Cross-references

- **Original human-facing SOPs (Desktop)**: `Foundational-Docs-SOP.html`, `Foundational-Docs-SOP-Tango.html`
- **Routines skill reference**: `/Users/leib/.claude/skills/routines/SKILL.md`
