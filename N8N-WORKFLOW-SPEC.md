# Foundational Docs Builder — n8n Workflow Spec

This document is the complete handoff for an n8n workflow builder to construct the **Foundational Docs Builder** automation. Reading top to bottom gives everything needed: trigger, services, node sequence, prompts (verbatim), error handling, cost ledger, and setup checklist.

A working reference implementation already exists as a Claude Code cloud routine in this same repo (`sops/foundational-docs/`). The n8n version targets the same outputs and uses the same templates/reference assets — only the orchestration changes.

---

## 1. Overview

### What this workflow does

When a new brand is added as a row in a Google Sheet, the workflow autonomously produces four foundational copywriting documents and commits them to a new GitHub branch:

1. **Research Doc** — deep prospect research (1,500–2,500 words, cited sources)
2. **Avatar Doc** — completed Avatar Sheet template
3. **Offer Brief** — completed Offer Brief template
4. **Necessary Beliefs Doc** — up to 6 "I believe that…" statements the prospect must hold before purchase

These four docs are the foundation every downstream copywriting project (sales pages, ads, email sequences) is built on.

### Architecture (high-level flow)

```
[Google Sheets: New row added]
            │
            ▼
   ┌────────────────────┐
   │ Validate inputs    │ ── missing fields ──► Slack error + Sheet status="Error"
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Status="Processing"│
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Step 1: Scrape     │ ── Jina fails ──► Firecrawl ──► fails ──► use Sales Copy Override
   │ sales page(s)      │                                            ──► all fail ──► error
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Read research      │ ── pulls research-part-1.md +
   │ methodology files  │     research-part-2.md from this repo via GitHub API
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Step 3: Perplexity │ ── single API call, returns 1,500-2,500 word research doc
   │ deep research      │
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Read avatar +      │ ── from this repo via GitHub API
   │ offer templates    │
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Step 4: Claude     │ ── fills avatar template
   │ fills Avatar Sheet │
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Step 5: Claude     │ ── fills offer brief template
   │ fills Offer Brief  │
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Read transcript    │ ── from this repo via GitHub API
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Step 7: Claude     │ ── 6 "I believe that..." statements
   │ writes Beliefs Doc │
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Commit all 6 files │ ── creates branch n8n/foundational-docs-<slug>-<date>
   │ to new GitHub      │     and pushes 6 files
   │ branch             │
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Update sheet:      │
   │ Status="Done",     │
   │ Branch URL set     │
   └────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Slack notify       │ ── #foundational-docs-runs
   └────────────────────┘
```

---

## 2. Service stack

| # | Service | Purpose | Auth | Cost |
|---|---|---|---|---|
| 1 | **Google Sheets API** | Trigger (new row) + status updates back to sheet | Google OAuth (existing JumpSquat Workspace) | Free |
| 2 | **Jina Reader** (`https://r.jina.ai/`) | Primary sales page scrape — converts any URL to clean markdown | None (free, no API key needed) | Free |
| 3 | **Firecrawl** (`https://api.firecrawl.dev/v1/scrape`) | Scrape fallback when Jina returns thin/empty/blocked | API key in `Authorization: Bearer` header | $9/mo for 3K pages |
| 4 | **Perplexity API** (`https://api.perplexity.ai/chat/completions`) | Step 3 deep research (model: `sonar-pro`) | API key in `Authorization: Bearer` header | ~$3–5 per research call |
| 5 | **Anthropic Claude API** (`https://api.anthropic.com/v1/messages`) | Steps 4, 5, 7 (template fills + beliefs synthesis), model `claude-sonnet-4-5-20250929` or latest | API key in `x-api-key` header | ~$0.10–0.30 per call × 3 = ~$0.30–0.90/brand |
| 6 | **GitHub API** (`https://api.github.com/`) | Read templates from this repo + write 6 output files on a new branch | Personal Access Token in `Authorization: Bearer` header | Free |
| 7 | **Slack** | Notify `#foundational-docs-runs` channel on completion + errors | Incoming webhook URL or OAuth | Free (existing workspace) |

---

## 3. Google Sheet schema

**Spreadsheet name suggestion**: `Foundational Docs Pipeline — Brands`

**Sheet name**: `Brands` (single sheet/tab)

| Column | Type | Required at row creation? | Description |
|---|---|---|---|
| `Brand` | text | **yes** | Brand name (e.g. "Eternal Life Tribe"). Used to generate `<brand-slug>` (lowercase, hyphenated). |
| `URL` | text (URL) | **yes** | Primary sales page URL. The workflow tries this first. |
| `Secondary URL` | text (URL) | optional | Second sales surface (e.g. ads landing page, about page). Fetched alongside primary for combined context. |
| `Category` | text | **yes** | Short phrase: "online holistic health community", "Skool community for ADHD adults", etc. |
| `Sales Copy Override` | long text | optional | Raw sales page text to use instead of scraping. If populated, workflow skips URL fetching entirely. Use when URLs are known to be 403/Cloudflare-gated. |
| `Status` | enum | workflow-managed | Empty → "Processing" → "Done" / "Error" |
| `Branch URL` | text (URL) | workflow-managed | Set on completion: `https://github.com/jumpsquat/marketing-sops/tree/n8n/foundational-docs-<slug>-<date>` |
| `Run Date` | datetime | workflow-managed | UTC timestamp set when workflow finishes (success or error) |
| `Notes` | long text | workflow-managed | Error details, warnings, or "OK" |
| `Cost` | currency (USD) | workflow-managed | Estimated total API cost for this run |

The trigger fires when a row has `Brand`, `URL`, `Category` populated AND `Status` is empty.

---

## 4. Setup prerequisites (must exist before first run)

In order:

### 4.1 GitHub
- Repo `jumpsquat/marketing-sops` exists (it does)
- Generate a fine-grained Personal Access Token (PAT) with these scopes on this repo only:
  - **Contents**: Read and write
  - **Metadata**: Read
- Save token in n8n credential store as `github-marketing-sops-pat`

### 4.2 Google Sheets
- Create the spreadsheet `Foundational Docs Pipeline — Brands` in the `jumpsquatmarketing@gmail.com` Drive
- Add the column headers from §3 to row 1
- Save Google OAuth credential in n8n as `google-sheets-jumpsquat`

### 4.3 Anthropic
- Create API key at https://console.anthropic.com/settings/keys
- Save in n8n credential store as `anthropic-api-key`

### 4.4 Perplexity
- Create API key at https://www.perplexity.ai/settings/api (requires paid plan, $20/mo minimum buy-in for API access)
- Save in n8n credential store as `perplexity-api-key`

### 4.5 Firecrawl
- Sign up at https://firecrawl.dev — Hobby plan ($9/mo) for 3K scrapes
- Save API key as `firecrawl-api-key`

### 4.6 Slack
- Create channel `#foundational-docs-runs` in the JumpSquat workspace
- Either:
  - **Incoming Webhook** (simplest): Slack app → Incoming Webhooks → Add to channel → save webhook URL
  - **Slack OAuth** (if you want richer messages): n8n Slack credential
- Save webhook URL or OAuth credential as `slack-foundational-docs`

### 4.7 Verify reference assets are in the repo
The workflow reads these from `jumpsquat/marketing-sops` on `main`:
- `sops/foundational-docs/reference/research-part-1.md`
- `sops/foundational-docs/reference/research-part-2.md`
- `sops/foundational-docs/reference/argument-structure-transcript.md`
- `sops/foundational-docs/templates/avatar-sheet.md`
- `sops/foundational-docs/templates/offer-brief.md`

(All five already exist on `main`.)

---

## 5. Node-by-node spec

Each node listed in execution order. `{var}` denotes data flowing from a previous node; n8n's expression syntax in production will be `{{$json.var}}` or similar.

### Node 1 — Trigger: Google Sheets — On Row Added

- **Type**: `n8n-nodes-base.googleSheetsTrigger`
- **Credential**: `google-sheets-jumpsquat`
- **Document**: `Foundational Docs Pipeline — Brands`
- **Sheet**: `Brands`
- **Trigger event**: Row Added
- **Polling interval**: 1 minute
- **Output filter** (so the node only fires on actionable rows): row has non-empty `Brand`, `URL`, `Category` AND empty `Status`
- **Output**: row data including `Brand`, `URL`, `Secondary URL`, `Category`, `Sales Copy Override`, plus the row index

### Node 2 — Function: Validate inputs and prepare metadata

- **Type**: `n8n-nodes-base.function`
- **Code**:

```javascript
const row = $input.first().json;
const required = ['Brand', 'URL', 'Category'];
const missing = required.filter(k => !row[k] || String(row[k]).trim() === '');

if (missing.length > 0) {
  throw new Error(`Missing required fields: ${missing.join(', ')}`);
}

// Slug the brand name
const slug = row.Brand.toLowerCase()
  .replace(/[^a-z0-9\s-]/g, '')
  .trim()
  .replace(/\s+/g, '-');

// Today in UTC as YYYYMMDD
const now = new Date();
const yyyymmdd = now.toISOString().slice(0, 10).replace(/-/g, '');

return [{
  json: {
    ...row,
    brandSlug: slug,
    runDate: yyyymmdd,
    runTimestamp: now.toISOString(),
    branchName: `n8n/foundational-docs-${slug}-${yyyymmdd}`,
    outputFolder: `active/foundational-docs/${slug}-${yyyymmdd}`,
    rowIndex: row.rowNumber,
  }
}];
```

- **On error**: route to Node 14 (error handler) with `errorReason="Missing required fields"`

### Node 3 — Google Sheets: Set status to "Processing"

- **Type**: `n8n-nodes-base.googleSheets`
- **Operation**: Update Row
- **Sheet**: `Brands`
- **Row**: `{rowIndex}`
- **Fields to update**:
  - `Status` = `Processing`
  - `Run Date` = `{runTimestamp}`

### Node 4 — Step 1a: Scrape primary URL via Jina Reader

- **Type**: `n8n-nodes-base.httpRequest`
- **Method**: GET
- **URL**: `https://r.jina.ai/{URL}` (literal — Jina takes the target URL as a path suffix)
- **Headers**: `{ "Accept": "text/markdown" }`
- **Continue on fail**: yes (so we can fall back)
- **Output**: response body = markdown of the page, OR error
- **Expected length**: typically 500–10,000 characters of clean markdown

### Node 5 — Step 1b: Scrape secondary URL via Jina Reader (conditional)

- **Type**: `n8n-nodes-base.httpRequest`
- **Run condition**: only if `Secondary URL` is non-empty
- **Method**: GET
- **URL**: `https://r.jina.ai/{Secondary URL}`
- **Headers**: `{ "Accept": "text/markdown" }`
- **Continue on fail**: yes

### Node 6 — IF: Did Jina succeed?

- **Type**: `n8n-nodes-base.if`
- **Condition**: Node 4 output length > 200 characters AND no HTTP error
- **True branch**: continue to Node 9 (combine sales copy)
- **False branch**: continue to Node 7 (Firecrawl fallback)

### Node 7 — Firecrawl fallback (primary URL)

- **Type**: `n8n-nodes-base.httpRequest`
- **Method**: POST
- **URL**: `https://api.firecrawl.dev/v1/scrape`
- **Headers**:
  - `Authorization: Bearer {firecrawl-api-key}`
  - `Content-Type: application/json`
- **Body**:

```json
{
  "url": "{URL}",
  "formats": ["markdown"],
  "onlyMainContent": true,
  "waitFor": 2000
}
```

- **Continue on fail**: yes

### Node 8 — IF: Firecrawl also failed → use Sales Copy Override

- **Type**: `n8n-nodes-base.if`
- **Condition**: Node 7 returned valid markdown
- **True branch**: continue (use Firecrawl output)
- **False branch**:
  - IF `Sales Copy Override` is non-empty → use it as the sales copy
  - ELSE → route to Node 14 (error handler) with `errorReason="All scraping methods failed and no Sales Copy Override provided"`

### Node 9 — Function: Combine sales copy

- **Type**: `n8n-nodes-base.function`
- **Code**:

```javascript
const data = $input.first().json;
let salesCopy = '';

if (data.salesCopyOverride && data.salesCopyOverride.trim()) {
  salesCopy = `# Sales page copy — ${data.Brand}\n\n## Source: SALES_COPY (provided by operator)\n\n${data.salesCopyOverride}`;
} else {
  salesCopy = `# Sales page copy — ${data.Brand}\n\n## Primary: ${data.URL}\n\n${data.primaryScrape || '(no primary content)'}`;
  if (data.secondaryScrape) {
    salesCopy += `\n\n## Secondary: ${data['Secondary URL']}\n\n${data.secondaryScrape}`;
  }
}

return [{
  json: { ...data, salesCopy }
}];
```

### Node 10 — Read research methodology + templates from GitHub

- **Type**: `n8n-nodes-base.httpRequest` × 5 (one per file, can be parallelized via Split In Batches or 5 parallel branches)
- **Method**: GET
- **URL pattern**: `https://api.github.com/repos/jumpsquat/marketing-sops/contents/<path>?ref=main`
- **Headers**:
  - `Authorization: Bearer {github-marketing-sops-pat}`
  - `Accept: application/vnd.github.raw`
- **Files to fetch**:
  1. `sops/foundational-docs/reference/research-part-1.md`
  2. `sops/foundational-docs/reference/research-part-2.md`
  3. `sops/foundational-docs/reference/argument-structure-transcript.md`
  4. `sops/foundational-docs/templates/avatar-sheet.md`
  5. `sops/foundational-docs/templates/offer-brief.md`
- **Output**: raw markdown content of each file
- **On error**: any file missing → route to Node 14 (this should never happen in practice since the files are committed)

### Node 11 — Step 3: Perplexity deep research

- **Type**: `n8n-nodes-base.httpRequest`
- **Method**: POST
- **URL**: `https://api.perplexity.ai/chat/completions`
- **Headers**:
  - `Authorization: Bearer {perplexity-api-key}`
  - `Content-Type: application/json`
- **Body**:

```json
{
  "model": "sonar-pro",
  "messages": [
    {
      "role": "system",
      "content": "<PERPLEXITY_SYSTEM_PROMPT — see §6.1>"
    },
    {
      "role": "user",
      "content": "<PERPLEXITY_USER_PROMPT — see §6.1, with {Brand}, {Category}, {URL}, {salesCopy}, {researchPart1}, {researchPart2} interpolated>"
    }
  ],
  "max_tokens": 8000,
  "temperature": 0.3,
  "return_citations": true,
  "return_related_questions": false
}
```

- **Output**: research doc markdown in `choices[0].message.content`, sources in `citations`
- **Timeout**: 300 seconds (Perplexity Sonar Pro can take 1–3 min for deep queries)

### Node 12 — Function: Format research doc

- **Type**: `n8n-nodes-base.function`
- **Purpose**: take Perplexity's content + citations array, ensure final markdown has proper "Sources" section with all cited URLs
- **Code**:

```javascript
const data = $input.first().json;
const research = data.perplexityResponse.choices[0].message.content;
const citations = data.perplexityResponse.citations || [];

let researchDoc = `# Research Doc — ${data.Brand}\n\n`;
researchDoc += `Research conducted: ${data.runTimestamp}\n`;
researchDoc += `Product category: ${data.Category}\n`;
researchDoc += `Sales page analyzed: ${data.URL}\n\n`;
researchDoc += research;

// If Perplexity didn't include a Sources section, append one
if (!research.includes('## Sources') && citations.length > 0) {
  researchDoc += `\n\n## Sources\n\n`;
  citations.forEach((url, i) => {
    researchDoc += `- [${i + 1}] ${url}\n`;
  });
}

return [{ json: { ...data, researchDoc } }];
```

### Node 13 — Step 4: Claude fills Avatar Sheet

- **Type**: `n8n-nodes-base.httpRequest` (or n8n's Anthropic node if available)
- **Method**: POST
- **URL**: `https://api.anthropic.com/v1/messages`
- **Headers**:
  - `x-api-key: {anthropic-api-key}`
  - `anthropic-version: 2023-06-01`
  - `Content-Type: application/json`
- **Body**:

```json
{
  "model": "claude-sonnet-4-5-20250929",
  "max_tokens": 8000,
  "messages": [
    {
      "role": "user",
      "content": "<CLAUDE_AVATAR_PROMPT — see §6.2, with {Brand}, {Category}, {salesCopy}, {researchDoc}, {avatarTemplate} interpolated>"
    }
  ]
}
```

- **Output**: filled avatar markdown in `content[0].text`
- **Timeout**: 120 seconds

### Node 14 — Step 5: Claude fills Offer Brief

Same as Node 13 but with `<CLAUDE_OFFER_BRIEF_PROMPT — see §6.3>`. Inputs: `{Brand}`, `{Category}`, `{salesCopy}`, `{researchDoc}`, `{avatarDoc}` (output of Node 13), `{offerBriefTemplate}`.

### Node 15 — Step 7: Claude generates Necessary Beliefs

Same as Node 13 but with `<CLAUDE_BELIEFS_PROMPT — see §6.4>`. Inputs: `{Brand}`, `{researchDoc}`, `{avatarDoc}`, `{offerBriefDoc}`, `{argumentTranscript}`.

### Node 16 — Function: Build inputs.md metadata file

- **Type**: `n8n-nodes-base.function`
- **Code**:

```javascript
const data = $input.first().json;
const inputsMd = `# Inputs — ${data.Brand}

**Run timestamp (UTC)**: ${data.runTimestamp}
**Brand slug**: ${data.brandSlug}
**Workflow**: n8n (this run)

## Parsed fields

- **BRAND**: ${data.Brand}
- **URL**: ${data.URL}
- **SECONDARY_URL**: ${data['Secondary URL'] || '(none)'}
- **CATEGORY**: ${data.Category}
- **SALES_COPY_OVERRIDE**: ${data.salesCopyOverride ? 'provided (used as ground truth)' : 'not provided'}

## Cost summary

- Perplexity (sonar-pro): ~$${data.perplexityCost.toFixed(2)}
- Claude API (3 calls): ~$${data.claudeCost.toFixed(2)}
- Firecrawl: ${data.firecrawlUsed ? '$0.003 (1 scrape)' : '$0.00'}
- Total: ~$${data.totalCost.toFixed(2)}
`;
return [{ json: { ...data, inputsMd } }];
```

### Node 17 — Commit 6 files to new GitHub branch

- **Type**: `n8n-nodes-base.httpRequest` × multiple, or use n8n's GitHub node
- **Sequence** (use the GitHub Contents API):
  1. Get the SHA of `main`'s HEAD: `GET /repos/jumpsquat/marketing-sops/git/ref/heads/main`
  2. Create new branch: `POST /repos/jumpsquat/marketing-sops/git/refs` with body `{ "ref": "refs/heads/{branchName}", "sha": "<main-sha>" }`
  3. For each of the 6 files, `PUT /repos/jumpsquat/marketing-sops/contents/{outputFolder}/{filename}`:
     - `00-inputs.md` (content: `inputsMd`)
     - `01-sales-page.md` (content: `salesCopy`)
     - `02-research-doc.md` (content: `researchDoc`)
     - `03-avatar-doc.md` (content: `avatarDoc`)
     - `04-offer-brief.md` (content: `offerBriefDoc`)
     - `05-necessary-beliefs.md` (content: `beliefsDoc`)
     - Each PUT body: `{ "message": "Foundational docs build (n8n) — {Brand} — {YYYY-MM-DD}", "content": "<base64-encoded content>", "branch": "{branchName}" }`

- **Output**: branch URL `https://github.com/jumpsquat/marketing-sops/tree/{branchName}`

### Node 18 — Google Sheets: Mark row "Done"

- **Type**: `n8n-nodes-base.googleSheets`
- **Operation**: Update Row
- **Row**: `{rowIndex}`
- **Fields**:
  - `Status` = `Done`
  - `Branch URL` = `{branchURL}`
  - `Run Date` = `{runTimestamp}` (UTC)
  - `Notes` = `OK` (or any non-blocking warnings — e.g. "Used Sales Copy Override; primary URL 403'd")
  - `Cost` = `{totalCost}`

### Node 19 — Slack notification: Success

- **Type**: `n8n-nodes-base.slack` (or HTTP to webhook)
- **Channel**: `#foundational-docs-runs`
- **Message** (formatted Slack blocks):

```
✅ Foundational Docs build complete
Brand: {Brand}
Category: {Category}
Branch: <{branchURL}|{branchName}>
Run cost: ~${totalCost}
Notes: {Notes}
```

### Node 14alt / Node 20 — Error handler (referenced from any failure point)

- **Type**: `n8n-nodes-base.function` + Slack + Sheet update
- **Behavior**:
  1. Update sheet row: `Status="Error"`, `Notes="<errorReason>: <details>"`, `Run Date={now}`
  2. Slack post:

```
❌ Foundational Docs build FAILED
Brand: {Brand}
Failed at: {nodeName}
Reason: {errorReason}
Sheet row: {rowIndex}
```

  3. End workflow

---

## 6. The 4 LLM prompts (verbatim — paste these into nodes)

### 6.1 Perplexity Deep Research prompt (Node 11)

**System message**:

```
You are a senior copywriting research analyst working in the Agora-school direct-response tradition. Your job is to produce a research dossier on a specific brand's prospect — the kind of document an expert copywriter could turn into a sales page or ad campaign without doing any additional research.

Your research must be:
- Grounded in real, citable sources (Reddit threads, news articles, forums, review sites, YouTube comments, podcast transcripts) — not speculation.
- Centered on the prospect's actual lived experience, in their own vocabulary.
- Specific enough that the reader can quote verbatim phrases from your output and feel confident they'd resonate with the target prospect.

You will be given:
1. A research methodology (Research Part 1 + Part 2) describing HOW to research a market for direct-response copy.
2. The actual sales page copy for the brand.
3. Brand and category context.

Internalize the methodology, then apply it to this specific brand. Output a structured markdown research doc following the exact section schema provided in the user message.
```

**User message** (with interpolation):

```
# Brand to research

- BRAND: {Brand}
- CATEGORY: {Category}
- Sales page URL: {URL}

# The sales page copy for this brand

```
{salesCopy}
```

# Research methodology — Part 1

```
{researchPart1}
```

# Research methodology — Part 2

```
{researchPart2}
```

# Your task

Produce a research doc covering 6 sections in this exact order. Target length: 1,500–2,500 words total. Cite sources inline as markdown links wherever you make a factual claim or quote a verbatim phrase. Do not speculate without citing.

Output the doc as markdown with this exact structure:

## 1. The Prospect
[demographics, psychographics, identity, daily reality — 200–300 words]

## 2. Primary Pains
[3–5 key pains across functional/emotional/social dimensions, with verbatim phrases — 250–400 words]

## 3. Dream Outcomes & Current Solutions
[what they want; what they've tried and why those failed — 200–300 words]

## 4. Prospect Vocabulary
[10–15 verbatim phrases from Reddit/YouTube/reviews, each with a source URL]

## 5. Beliefs & Objections
[existing beliefs about the category + what stops them buying — 200–300 words]

## 6. Competitor Landscape
[top 3–5 alternatives, their positioning, their gaps that this brand can exploit — 200–300 words]

Do not include a separator headline like "# Research Doc" — start directly with `## 1. The Prospect`. The workflow will prepend the title.

Do not exceed 2,500 words. Stop when each section is adequately covered.
```

### 6.2 Claude — Fill Avatar Sheet (Node 13)

**User message** (single message, no system prompt needed):

```
You are filling out an Avatar Sheet template for a copywriting research project. Below you'll find:

1. The Avatar Sheet template (with placeholder fields like `[Specify ...]`)
2. The brand's sales page copy
3. A completed research doc on the prospect
4. The brand name and category

Your job: fill out every field of the template using the research and sales copy as evidence. Where the research is ambiguous, mark the field `[inferred: <one-sentence reasoning>]` rather than inventing specifics. Preserve the template's structure exactly — every section, every emoji header, every bullet style. Do not add or remove sections.

# Brand
- BRAND: {Brand}
- CATEGORY: {Category}

# Sales page copy
```
{salesCopy}
```

# Research doc
```
{researchDoc}
```

# Avatar Sheet template (fill this out)
```
{avatarTemplate}
```

# Output

Return only the completed avatar sheet markdown. Begin directly with the template's first heading. Do not include any preamble, explanation, or trailing notes.
```

### 6.3 Claude — Fill Offer Brief (Node 14)

**User message**:

```
You are filling out an Offer Brief template for a copywriting research project. You have access to:

1. The brand's sales page copy
2. The completed research doc on the prospect
3. The completed avatar sheet
4. The Offer Brief template (with placeholder fields)

Your job: fill out every field of the template using the upstream documents as evidence. For frameworks like Schwartz's Levels of Awareness and Stages of Sophistication, give a specific stage number with one-paragraph justification. For Big Idea, UMP, UMS, Discovery Story — write substantive content (not placeholders). Where any field can't be answered from the available evidence, mark `[inferred: <reasoning>]` rather than fabricating.

Preserve the template's structure exactly. Do not add or remove sections.

# Brand
- BRAND: {Brand}
- CATEGORY: {Category}

# Sales page copy
```
{salesCopy}
```

# Research doc
```
{researchDoc}
```

# Avatar sheet
```
{avatarDoc}
```

# Offer Brief template (fill this out)
```
{offerBriefTemplate}
```

# Output

Return only the completed offer brief markdown. Begin directly with the template's first heading. Do not include any preamble, explanation, or trailing notes.
```

### 6.4 Claude — Generate Necessary Beliefs (Node 15)

**User message**:

```
You are producing a Necessary Beliefs Doc — the core deliverable from the Agora-school copywriting tradition. The premise: marketing is the process of leading a prospect to a specific belief via a rock-solid logical + emotional argument, not via clever word choice. Your job is to identify the EXACT chain of beliefs the prospect must hold, in order, before they will purchase.

You have access to:

1. A transcript explaining the Agora "craft arguments not copy" framework (read this first; it grounds your work)
2. The completed research doc on the prospect
3. The completed avatar sheet
4. The completed offer brief

# Argument-structure framework (read this first)
```
{argumentTranscript}
```

# Brand context

- BRAND: {Brand}

# Research doc
```
{researchDoc}
```

# Avatar sheet
```
{avatarDoc}
```

# Offer brief
```
{offerBriefDoc}
```

# Your task

Produce no more than 6 "I believe that…" statements the prospect must hold, in sequence, before clicking buy. Each belief must:
- Be written in the prospect's voice ("I believe that..." — first person)
- Be a logical prerequisite for purchase (if they don't hold this, they won't buy)
- Depend on the previous beliefs (belief N requires beliefs 1..N-1)
- Be specific to this brand's offer, not generic platitudes

For each belief, include:
- **Why necessary**: 2–3 sentences on what role this belief plays in the chain — what problem it solves for the prospect, what would happen if they didn't hold it.
- **Shifting evidence**: 3–5 sentences on the specific arguments, framings, citations, and proof elements from the offer that move a non-believer toward this belief. Reference specific facts from the offer brief and research doc.

# Output format

Begin directly with `## Belief 1: I believe that…`. No preamble, no introductory paragraph, no closing notes. Use this exact structure:

## Belief 1: I believe that [statement]

**Why necessary**: …

**Shifting evidence**: …

## Belief 2: I believe that [statement]

[…and so on, up to 6]

Do not include the "# Necessary Beliefs Doc — {Brand}" title. The workflow will prepend it.
```

---

## 7. Error handling matrix

| Failure point | Behavior |
|---|---|
| Node 2 validation fails (missing required field) | Sheet row → `Status=Error`, Slack alert with field names. End. |
| Both URL scrapes fail (Jina + Firecrawl) AND no `Sales Copy Override` | Sheet row → `Status=Error`, `Notes="Both URLs unreachable, no Sales Copy Override provided"`. Slack alert. End. |
| One URL fetch fails but other succeeds OR override present | Continue. `Notes` field warns about partial input. |
| GitHub template/reference fetch fails (any of 5 files) | Sheet row → `Status=Error`, `Notes="Reference file missing: <path>"`. Slack alert with file path. End. (Should never happen — files are committed; this is a defensive check.) |
| Perplexity returns < 1000 chars or HTTP error | Retry once after 30s wait. If still fails, `Status=Error`, `Notes="Perplexity research failed"`. Slack alert. End. |
| Perplexity returns successful but research doc < 800 words | Continue but flag in `Notes="Research doc thin (<800 words) — review before using"`. Don't fail. |
| Claude API call fails (rate limit, 5xx) | Retry up to 3 times with exponential backoff (5s, 15s, 45s). If still fails, `Status=Error` with the failed step name. |
| GitHub branch creation fails (e.g., branch already exists from prior partial run) | Append `-retry-{HHMMSS}` to branch name and continue. Log warning in `Notes`. |
| GitHub file commit fails (any of 6 files) | Retry once. If persists, capture which files DID commit and which didn't in `Notes`. Mark `Status=Error`. Slack alert with branch URL so partial output is recoverable. |
| Slack notification fails | Log to n8n execution log only. Don't block sheet update — sheet IS the source of truth. |

---

## 8. Cost ledger

Per brand, typical case:

| Service | Calls | Per-call cost | Subtotal |
|---|---|---|---|
| Jina Reader | 1–2 | $0 | $0.00 |
| Firecrawl (only if Jina fails) | 0–1 | ~$0.003 | $0.00–0.003 |
| Perplexity Sonar Pro | 1 | $3–5 | $3.00–5.00 |
| Claude Sonnet (Avatar fill) | 1 | ~$0.10–0.20 | $0.10–0.20 |
| Claude Sonnet (Offer Brief fill) | 1 | ~$0.10–0.30 | $0.10–0.30 |
| Claude Sonnet (Beliefs) | 1 | ~$0.10–0.30 | $0.10–0.30 |
| GitHub API | ~10 | $0 | $0.00 |
| Google Sheets API | ~3 | $0 | $0.00 |
| Slack | 1 | $0 | $0.00 |
| **Total per brand** | | | **~$3.30–5.85** |

Monthly fixed: $9 (Firecrawl), $20 (Perplexity API minimum) = **$29/mo base**.

Break-even vs. manual VA execution at any volume (a VA's hour costs more than 5 brands' worth of API calls).

---

## 9. First-run test payload

Seed the spreadsheet with this row to test:

| Brand | URL | Secondary URL | Category | Sales Copy Override |
|---|---|---|---|---|
| Eternal Life Tribe | https://masterclass.samuelbleemd.com/fb_completed | https://www.skool.com/eternallifetribe/about | online holistic health / spiritual wellness community led by a medical doctor | (paste full sales copy here — see `active/foundational-docs/eternal-life-tribe-20260416/01-sales-page.md` in this repo for reference content) |

The reason for the override: both ELT URLs return 403 to server-side fetches (Cloudflare bot protection). Use the override for the first test so you're not debugging both the workflow AND the scraping fallback at the same time.

After the first run succeeds, you can leave Sales Copy Override empty for subsequent brands and let the Jina → Firecrawl chain handle it.

**Expected outcome of first run**:
- Branch `n8n/foundational-docs-eternal-life-tribe-<YYYYMMDD>` appears on `jumpsquat/marketing-sops`
- 6 files committed in `active/foundational-docs/eternal-life-tribe-<YYYYMMDD>/`
- Sheet row updated to `Status=Done`, `Branch URL` populated, `Cost` ~$4
- Slack notification in `#foundational-docs-runs`

The 6-file structure should match the Claude Code routine's first run (in branch `claude/foundational-docs-eternal-life-tribe-20260416`). Diff the two branches' outputs to validate the n8n version produces comparable quality.

---

## 10. Iteration notes

The Instructions prompt for the cloud routine version is a thin bootstrap that points to `sops/foundational-docs/prompt/routine-prompt.md` for the authoritative SOP. The n8n version inlines the prompts directly into nodes 11/13/14/15.

When iterating:
- **To change research scope or output structure** → edit the Perplexity prompt in Node 11
- **To change avatar/offer brief field treatment** → edit the Claude prompts in Nodes 13/14, OR update the templates in `sops/foundational-docs/templates/` (the workflow re-fetches them on every run, so a `git push` to `main` is enough)
- **To swap Perplexity for Tavily+Claude** (cost reduction) → replace Node 11 with a Tavily HTTP call followed by a Claude synthesis call. The downstream nodes don't need to change as long as `researchDoc` ends up populated in the same shape.
- **To swap Claude for OpenAI** → swap Nodes 13/14/15 to OpenAI's `chat/completions` endpoint with the same prompt content.

The 5 reference files in `sops/foundational-docs/{templates,reference}/` are the source of truth for the workflow's behavior. Keep them in sync between the cloud routine and the n8n workflow by treating that folder as read-only-ish — only edit those files when you want both runners to change behavior together.

---

## Cross-references

- **Cloud routine version of this same workflow**: `sops/foundational-docs/prompt/routine-prompt.md` (the authoritative SOP)
- **Cloud routine handoff**: `HANDOFF.md`
- **First successful cloud routine output (for diff comparison)**: branch `claude/foundational-docs-eternal-life-tribe-20260416`
- **Original human-facing SOPs (Desktop)**: `Foundational-Docs-SOP.html`, `Foundational-Docs-SOP-Tango.html`
