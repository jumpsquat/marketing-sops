# Foundational Docs Builder

Automated production of the four foundational copywriting documents — Research Doc, Avatar Doc, Offer Brief, Necessary Beliefs Doc — for a new brand entering the JumpSquat Marketing copywriting pipeline.

Originally a manual 7-prompt ChatGPT workflow run by a VA. Converted to a Claude Code cloud routine that fires unattended and commits the finished docs to this repo.

## What fires this routine

An API trigger attached to the `Foundational Docs Builder` routine at [claude.ai/code/scheduled](https://claude.ai/code/scheduled). Two primary fire paths:

1. **Manual** — `RemoteTrigger({action:"run", trigger_id:"trig_...", body:{"text": "<payload>"}})` from any Claude Code session.
2. **Webhook** — `POST https://api.anthropic.com/v1/claude_code/routines/{trigger_id}/fire` with the per-routine bearer token. Intended to be called from a Make.com scenario when a new brand is added to the brands tracking spreadsheet, mirroring the "Brand is added to spreadsheet" trigger in the original flowchart.

## Payload format

The `text` field of the fire request must contain:

```
BRAND: <brand name>
URL: <sales page URL>
CATEGORY: <what the brand sells, e.g. "online spiritual wellness community">
```

All three fields are required. The routine exits early if any are missing.

## Folder layout used by the routine

```
sops/foundational-docs/
├── README.md                                   # this file
├── prompt/
│   └── routine-prompt.md                       # full instructions the routine executes
├── templates/
│   ├── avatar-sheet.md                         # extracted from the original Drive template
│   └── offer-brief.md                          # extracted from the original Drive template
└── reference/
    ├── research-part-1.md                      # research methodology, extracted from Drive
    ├── research-part-2.md                      # research methodology, extracted from Drive
    └── argument-structure-transcript.md        # Agora "craft arguments" teaching

active/
└── foundational-docs/
    └── <brand-slug>-<YYYYMMDD>/                # one folder per routine fire
        ├── 00-inputs.md                        # payload + timestamp
        ├── 01-sales-page.md                    # scraped sales copy
        ├── 02-research-doc.md                  # deep research output (2500+ words)
        ├── 03-avatar-doc.md                    # filled avatar sheet
        ├── 04-offer-brief.md                   # filled offer brief
        └── 05-necessary-beliefs.md             # up to 6 "I believe that..." statements
```

The routine commits the `<brand-slug>-<YYYYMMDD>/` folder on a branch named `claude/foundational-docs-<brand-slug>-<YYYYMMDD>` and prints a summary to stdout.

## How the routine differs from the manual SOP

| Manual SOP step | Routine equivalent |
|---|---|
| VA opens ChatGPT, runs Prompt #1 with sales page PDF | `WebFetch` on the sales page URL |
| Prompt #2 attaches Research Part 1 + 2 Drive files | `Read` on `reference/research-part-1.md` + `research-part-2.md` |
| Prompt #3 generates a Deep Research prompt for OpenAI's tool | Claude runs the research itself via `WebSearch` + `WebFetch`, no second tool required |
| Prompt #4 attaches Avatar Sheet Template Drive file | `Read` on `templates/avatar-sheet.md` |
| Prompt #5 attaches Offer Brief Template Drive file | `Read` on `templates/offer-brief.md` |
| New ChatGPT conversation for Prompts #6–7 (ChatGPT context ceiling) | Same Claude session — larger context window handles the whole workflow |
| Prompt #6 pastes the 1,500-word transcript | `Read` on `reference/argument-structure-transcript.md` |
| Prompt #7 re-attaches Avatar + Offer + Research docs | Files already in context from Steps 4–6 in the same session |

## Reference files for the human operator

The human-facing versions of the SOP that predate the routine live on Leo's Desktop:
- `~/Desktop/Foundational-Docs-SOP.html` — original flowchart visualization
- `~/Desktop/Foundational-Docs-SOP-Tango.html` — Tango.ai-styled version

Those are for VA reference. The routine executes from `prompt/routine-prompt.md` in this repo.

## Updating the workflow

To change how the routine behaves:
1. Edit `prompt/routine-prompt.md` on the `main` branch of this repo and push.
2. The next fire picks up the updated prompt automatically — no routine reconfiguration needed.

To swap in a newer Avatar Sheet or Offer Brief template:
1. Replace the file in `templates/`.
2. No other change required — the routine always `Read`s the current version.

## Output discoverability

After a fire:
- Watch the live session at `claude.ai/code/scheduled/{trigger_id}`.
- Once the session completes, `git fetch && git checkout claude/foundational-docs-<brand-slug>-<YYYYMMDD>` locally to pull the outputs.
- The routine also prints a summary with all 6 file paths to stdout (visible in the session transcript).
