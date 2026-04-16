# Foundational Docs Builder — Routine Handoff

Everything you need to stand up the **Foundational Docs Builder** cloud routine in the Claude Code web UI and fire it against your first brand.

Total time: **~2 minutes** to create + configure, then one click to fire.

---

## Pre-flight (30 seconds)

1. **Delete the probe routine.** Go to https://claude.ai/code/scheduled → click `Env ID Discovery Test (probe)` → click the trash icon (top right of the detail panel) → confirm. It's a leftover from the API-creation attempt that couldn't resolve your env_id.
2. **Confirm this repo exists.** https://github.com/jumpsquat/marketing-sops — should show the `sops/foundational-docs/` tree and this `HANDOFF.md`. If you don't see it, `gh repo view jumpsquat/marketing-sops` from terminal.
3. **Confirm you're signed in to Claude Code.** https://claude.ai/code/scheduled — you should see the empty routines list (or just the probe you're about to delete).

---

## Create the routine (60 seconds)

At https://claude.ai/code/scheduled click **New routine** (or equivalent button). Fill in the form with these values:

### Field values

| Field | Value |
|---|---|
| **Name** | `Foundational Docs Builder` |
| **Instructions** | paste the prompt in the box below |
| **Model** | `Sonnet 4.6` |
| **Repositories** | add `jumpsquat/marketing-sops` |
| **Environment** (cloud icon next to repo) | `Default` (already selected) |
| **Select a trigger** | click **API** |
| **Connectors** tab | leave empty |
| **Permissions** tab → "Allow unrestricted branch pushes" | leave **OFF** — routine only needs `claude/*` branches |

### The Instructions prompt (paste this verbatim)

```
You are the Foundational Docs Builder — a Claude Code cloud routine that produces four copywriting documents (Research Doc, Avatar Doc, Offer Brief, Necessary Beliefs Doc) for a new brand entering JumpSquat Marketing's copywriting pipeline.

Your full operating instructions live in this repository at:
  sops/foundational-docs/prompt/routine-prompt.md

On every fire:
1. Read that file with the Read tool immediately.
2. Follow every step in it exactly, using the BRAND / URL / SECONDARY_URL / CATEGORY payload appended to this message via the /fire endpoint's "text" field (or the Run dialog's input box).
3. Commit your outputs to a `claude/foundational-docs-<brand-slug>-<YYYYMMDD>` branch as specified in that file.
4. Print the summary from the Wrapup section to stdout.

The repo file is the source of truth. If any instruction here seems to conflict with it, defer to the repo file.
```

Click **Save**. The routine appears in your list at `claude.ai/code/scheduled`.

---

## Generate the API bearer token (only if you want webhook firing)

In the routine detail view, click the pencil → find the **API** trigger → click **Generate token**. **Copy it immediately and save it somewhere secure (1Password, a note, etc.) — the UI shows it once and can never retrieve it again.**

You don't need this token to fire via the Run button — only for external webhooks (Make.com, curl, etc.).

---

## Fire the first test

### The test payload (paste into the Run dialog)

```
BRAND: Eternal Life Tribe
URL: https://masterclass.samuelbleemd.com/fb_completed
SECONDARY_URL: https://www.skool.com/eternallifetribe/about
CATEGORY: online holistic health / spiritual wellness community led by a medical doctor
```

### Three ways to fire

**Option 1 — Run button (simplest for testing):**
1. Open the routine detail view at `claude.ai/code/scheduled/<your-trigger-id>`
2. Click **Run** (or the equivalent manual-fire button)
3. When prompted for input text, paste the payload above
4. Confirm

**Option 2 — curl (for webhook-style testing):**

```bash
curl -X POST https://api.anthropic.com/v1/claude_code/routines/TRIGGER_ID/fire \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: experimental-cc-routine-2026-04-01" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "BRAND: Eternal Life Tribe\nURL: https://masterclass.samuelbleemd.com/fb_completed\nSECONDARY_URL: https://www.skool.com/eternallifetribe/about\nCATEGORY: online holistic health / spiritual wellness community led by a medical doctor"
  }'
```

Replace `TRIGGER_ID` (the `trig_01...` ID from the routine URL) and `YOUR_TOKEN` (from the API token generation step).

**Option 3 — Make.com webhook (later, for auto-firing on new brands):**
Point a Make.com scenario at the same `/fire` endpoint with the same headers. Configure the body template as:

```json
{"text": "BRAND: {{brand}}\nURL: {{sales_url}}\nSECONDARY_URL: {{secondary_url}}\nCATEGORY: {{category}}"}
```

Source the variables from whatever triggers the scenario (new row in a Google Sheet, form submission, etc.).

---

## Watch it run

Open `claude.ai/code/scheduled/<your-trigger-id>` — the active session appears at the top of the routine's Runs list (orange spinner = running, green = done, red = failed). Click it to see the live transcript.

Expected runtime: **8–15 minutes**. Most of the time goes to Step 3 (deep research via WebSearch + WebFetch — typically 10+ queries plus several page fetches).

---

## Success criteria

When the session finishes, you should see:

1. **A new branch on GitHub**: `claude/foundational-docs-eternal-life-tribe-<YYYYMMDD>`
2. **Six committed files** in `active/foundational-docs/eternal-life-tribe-<YYYYMMDD>/`:
   - `00-inputs.md` — the payload + timestamp
   - `01-sales-page.md` — scraped copy from both ELT URLs
   - `02-research-doc.md` — 2,500+ word research doc, 10 labeled sections
   - `03-avatar-doc.md` — filled Avatar Sheet Template
   - `04-offer-brief.md` — filled Offer Brief Template
   - `05-necessary-beliefs.md` — ≤6 "I believe that…" statements, each with "Why necessary" + "Shifting evidence"
3. **A summary printed to stdout** (visible in the session transcript) listing the brand, all 6 file paths, and the branch name
4. **No `Flags:` warnings** in the summary (if there are any, they tell you what needs manual follow-up — e.g. a sales page WebFetch failed)

Pull the branch locally to review:
```bash
cd ~/Desktop/marketing-sops
git fetch
git checkout claude/foundational-docs-eternal-life-tribe-<YYYYMMDD>
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Run is stuck on orange spinner for 30+ min | Cloud env can't resolve (was my earlier problem) or research step is genuinely long — check transcript | If the transcript shows no activity past "spinning up environment," kill the run and re-check that the routine has `Default` env selected. If transcript shows research queries firing, just wait. |
| `01-sales-page.md` starts with `ERROR: Sales page unreachable` | WebFetch blocked (403/timeout) | Both URLs in the payload might fail simultaneously if the ads landing page is geo-fenced or bot-detected. The routine continues using whatever sales page content did load plus the BRAND/CATEGORY context. If both fail, swap `URL` to a different public page for that brand and re-fire. |
| Research doc under 2,000 words | Research tools returned thin results on first pass | Open the transcript, see which queries were skipped or errored. You can re-fire manually with a more specific CATEGORY hint, or add a note to the prompt at `sops/foundational-docs/prompt/routine-prompt.md` and `git push` — the next fire picks up the change. |
| Necessary Beliefs doc has no structure | Step 6 (transcript load) was skipped | Check that `sops/foundational-docs/reference/argument-structure-transcript.md` exists on `main`. If it was deleted accidentally, restore and re-fire. |
| Git push rejected at wrapup | Branch protection rule or auth issue | Routine prints the commit SHA + local path. Pull the session logs, clone the routine's working tree if needed, push manually. |
| `Permission denied: cannot push to main` | Expected — "Allow unrestricted branch pushes" is OFF | Not a bug. Routine should never push to `main`; it pushes to `claude/foundational-docs-<brand>-<date>`. If you see this error, the routine is misbehaving — file a note. |

---

## How to iterate on the routine's behavior

The UI prompt is a thin bootstrap. **The actual workflow lives in `sops/foundational-docs/prompt/routine-prompt.md`** in this repo. To change how the routine behaves:

1. Edit that file locally
2. `git commit` + `git push` to `main`
3. The next fire picks up the change automatically — no UI edit, no redeploy

To swap in a newer Avatar Sheet or Offer Brief template:
1. Replace the file in `sops/foundational-docs/templates/`
2. `git push` — next fire uses the new version

To add new reference material (e.g., a second copywriting transcript):
1. Drop the file in `sops/foundational-docs/reference/`
2. Add a `Read` instruction for it in `routine-prompt.md`
3. `git push`

---

## Cross-references

- **Original human-facing SOPs (Desktop)**: `Foundational-Docs-SOP.html`, `Foundational-Docs-SOP-Tango.html`
- **Routine skill reference**: `/Users/leib/.claude/skills/routines/SKILL.md` (localized for your account)
- **Plan file for this build**: `/Users/leib/.claude/plans/humming-tickling-leaf.md`
