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
This is the routine I want you to build.

The routine produces four foundational copywriting documents for any new brand entering the copywriting pipeline — a deep research doc, a completed avatar sheet, a completed offer brief, and a necessary-beliefs doc listing up to six "I believe that…" statements the prospect must hold before purchase. These four documents are the foundation every downstream copywriting project (sales pages, ads, email sequences) is built on.

Each fire receives a text payload with four fields: BRAND, URL (primary sales page), optional SECONDARY_URL (e.g. an ads landing page), and CATEGORY. The routine parses those fields, fetches the sales page copy, internalizes a research methodology and an argument-structure framework stored in the repository, runs fresh web research against the specific product, and writes the four docs in sequence — each one informed by the ones before it.

The full step-by-step SOP — every file path, template, output format, and naming convention the routine needs — lives in the repository at `sops/foundational-docs/prompt/routine-prompt.md`. The routine reads that file on every fire and follows it exactly. The templates (Avatar Sheet, Offer Brief), reference methodology (Research Part 1 & 2), and argument-structure transcript that file points to are all in the same repo under `sops/foundational-docs/`.

Outputs are committed to a new `claude/foundational-docs-<brand-slug>-<YYYYMMDD>` branch and summarized to stdout on completion. The repository file is the source of truth.
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
