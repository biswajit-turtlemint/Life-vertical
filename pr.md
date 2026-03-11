# 🤖 PR Review Bot — AI-Powered Code Review CLI
A lightweight CLI tool that reviews your code changes using **AI** and posts structured suggestions to **Slack** — with **zero org-level permissions** required.
---
## What This Does
```
You run a single command from your project repo
        ↓
CLI reads your git diff locally (your branch vs any base branch)
        ↓
Enriches diff with full file content for cross-method analysis
        ↓
Sends EACH file individually to Groq AI (Llama 3.3 70B) with a 21-rule checklist
        ↓
AI performs deep line-by-line analysis per file checking:
   🔴 Null Safety  🔒 Security  ⚠️ Error Handling
   🧹 Dead Code    ♻️ DRY       🔄 Data Consistency  📐 API Design
        ↓
Per-file reviews are aggregated into a final report with cross-file analysis
        ↓
Posts a formatted, section-separated review to your Slack channel
        ↓
Team discusses and resolves suggestions
```
**No GitHub tokens. No org admin access. Completely free.**
---
## How the Multi-Pass Review Works
Unlike a simple "send all diffs at once" approach, this bot uses a **file-by-file multi-pass strategy** to work within Groq's free tier token limits while maximizing review depth:
### Phase 1: Per-File Deep Review
```
For each changed file:
  1. Extract the diff patch with line numbers (L1, L2, ...)
  2. Load the full file content via `git show HEAD:<filename>`
  3. Send BOTH to the AI with the 21-rule checklist prompt
  4. Store the per-file review result
  5. Wait 15 seconds (Groq free tier: 12K tokens/minute rolling window)
  6. If rate-limited (429), auto-retry after the wait time specified by Groq
```
Each file is reviewed **independently** with its full context, so the AI can:
- Compare null-check patterns across methods within the same class
- Detect duplicated logic between methods in the same file
- Trace variable assignments line-by-line for dead code detection
- Cross-reference imports against actual usage
### Phase 2: Aggregation & Cross-File Analysis
```
After all files are reviewed:
  1. Combine all per-file reviews into one prompt
  2. Send to AI with aggregation instructions
  3. AI deduplicates, adds severity ratings, and finds cross-file issues:
     - Same object null-checked in File A but not in File B
     - Same builder pattern duplicated across different service files
     - Inconsistent error handling (one file logs, another swallows)
  4. Final report is formatted and posted to Slack
```
### Why Multi-Pass?
| Approach | Token Usage | Depth | Cross-File Analysis |
|----------|------------|-------|---------------------|
| Single pass (all files) | ❌ Exceeds Groq free tier limit | Shallow — AI skims | ❌ None |
| **Multi-pass (file by file)** | ✅ Fits within 12K TPM | **Deep — line by line** | ✅ In aggregation phase |
---
## The AI Review Prompt (21 Rules)
The bot uses a structured **21-rule checklist** that the AI must check against every line of code. Each rule has an ID (R1–R21) for traceability in the output.
### 🔴 Null Safety (R1–R4)
| Rule | What It Catches |
|------|----------------|
| **R1** | Method calls on objects that could be null (from DB lookups, service calls, `.get()`) |
| **R2** | Method chain NPEs like `a.getB().getC()` where any intermediate could be null |
| **R3** | `.trim()`, `.toLowerCase()`, `.length()` etc. called directly on a String parameter that could be null |
| **R4** | Inconsistent null checks — method A checks null but method B in the same class doesn't |
### 🔒 Security (R5–R9)
| Rule | What It Catches |
|------|----------------|
| **R5** | Hardcoded secrets, API keys, passwords in source code (patterns like `sk_`, `api_key`, long alphanumeric constants) |
| **R6** | Log injection — user input concatenated with `+` into log statements instead of `{}` placeholders |
| **R7** | Sensitive data logged via `.toString()` on Maps, DTOs, request objects containing payment/card data |
| **R8** | Stack traces, `e.getMessage()`, `e.getStackTrace()` exposed in HTTP responses |
| **R9** | Missing input validation — `Enum.valueOf()` without try-catch, unvalidated controller params |
### ⚠️ Error Handling (R10–R12)
| Rule | What It Catches |
|------|----------------|
| **R10** | Empty catch blocks — exception swallowed silently |
| **R11** | Catch block that catches Exception but doesn't log the exception object |
| **R12** | Generic `catch(Exception)` when only specific checked exceptions are thrown |
### 🧹 Dead Code (R13–R16)
| Rule | What It Catches |
|------|----------------|
| **R13** | Variable assigned then immediately overwritten without being read (dead computation / variable shadowing) |
| **R14** | Unused imports — import present but class never referenced in the code |
| **R15** | Unused private methods or fields |
| **R16** | Unreachable code after return/throw/break |
### ♻️ DRY / Reusability (R17–R18)
| Rule | What It Catches |
|------|----------------|
| **R17** | Duplicated logic across methods — URL construction, builder patterns, string assembly |
| **R18** | Private helper methods that exist but are never called |
### 🔄 Data Consistency (R19)
| Rule | What It Catches |
|------|----------------|
| **R19** | Multiple DB writes (save + update) without `@Transactional`, risking partial failures |
### 📐 API Design (R20–R21)
| Rule | What It Catches |
|------|----------------|
| **R20** | Raw types — `ResponseEntity` without type parameter |
| **R21** | `@GetMapping` on methods that modify state |
---
## Prerequisites
| Requirement | Details |
|---|---|
| **Node.js** | v16+ (check: `node -v`). Recommended: v18+ |
| **Git** | Must be installed (check: `git --version`) |
| **Groq API Key** | Free — see setup below |
| **Slack Webhook URL** | Free — see setup below |
| **A cloned Git repo** | The project you want to review must be cloned locally |
---
## Project Structure
```
prBot/
├── index.js              # CLI entry point — orchestrates multi-pass review pipeline
├── lib/
│   ├── local-git.js      # Reads git diff locally + provides full file content
│   ├── ai-reviewer.js    # Multi-pass AI reviewer (per-file → aggregation)
│   └── slack.js          # Formats & posts results to Slack (markdown → mrkdwn)
├── .env                  # Your API keys (never commit this!)
├── .env.example          # Template showing required keys
├── .gitignore            # Ignores node_modules and .env
├── package.json          # Node.js dependencies
└── README.md             # This file
```
### How each file works:
- **`index.js`** — The main CLI. Parses your command (`review` or `test-slack`), validates config, orchestrates the pipeline: diff → enrich with full file content → file-by-file AI review → aggregate → Slack. Shows per-file progress in the terminal.
- **`lib/local-git.js`** — Runs `git diff` between your current branch and the base branch. Parses the unified diff format into per-file changes with line numbers. Also provides full file content via `git show` for cross-method analysis. No GitHub API or token needed.
- **`lib/ai-reviewer.js`** — The core review engine. Reviews each file individually against the 21-rule checklist, with full file content for context. Includes:
  - `PER_FILE_SYSTEM_PROMPT` — The 21-rule checklist prompt
  - `AGGREGATION_SYSTEM_PROMPT` — Cross-file analysis and deduplication
  - Auto-retry with backoff for Groq 429 rate limit errors
  - 15-second delays between API calls for Groq free tier
  - Progress callback for terminal UI updates
- **`lib/slack.js`** — Converts AI markdown output to Slack Block Kit format. Handles:
  - Markdown → Slack mrkdwn conversion (`##` → bold, `**` → `*`)
  - Section-based layout with dividers between issue categories
  - Splits long sections at bullet boundaries (3000 char Slack limit)
  - Multi-message support if review exceeds 50-block Slack limit
---
## Setup Guide
### Step 1: Install Dependencies
```bash
cd ~/companyProjects/prBot
npm install --registry https://registry.npmjs.org
```
---
### Step 2: Get Your Groq API Key (Free)
1. Go to **[console.groq.com](https://console.groq.com)**
2. Click **"Sign Up"** — sign in with your Google account
3. Once logged in, go to **[console.groq.com/keys](https://console.groq.com/keys)**
4. Click **"Create API Key"**
5. Give it a name like `pr-review-bot`
6. **Copy the key** — it starts with `gsk_...`
7. Save it somewhere safe (you won't see it again)
---
### Step 3: Get Your Slack Webhook URL (Free)
1. Go to **[api.slack.com/apps](https://api.slack.com/apps)**
2. Click **"Create New App"** → Choose **"From scratch"**
3. Fill in:
   - **App Name:** `PR Review Bot`
   - **Workspace:** Select your company's Slack workspace
4. Click **"Create App"**
5. In the left sidebar, click **"Incoming Webhooks"**
6. Toggle the switch to **ON** (top right)
7. Scroll down → Click **"Add New Webhook to Workspace"**
8. **Select the channel** where reviews should be posted (e.g., `#pr-reviews`, `#dev-team`)
9. Click **"Allow"**
10. You'll see a **Webhook URL** — it looks like:
    ```
    https://hooks.slack.com/services/T02XXXXX/B06XXXXX/xxxxxxxxxxxxxxxxxxx
    ```
11. Click **"Copy"**
> 💡 **Note:** Creating a Slack app only needs workspace membership, not admin access. If your workspace requires admin approval, ask your admin — it's just an incoming webhook (posts messages only, no data access).
---
### Step 4: Configure the `.env` File
```bash
cd ~/companyProjects/prBot
cp .env.example .env
nano .env
```
Paste your keys:
```env
GROQ_API_KEY=gsk_your_groq_key_here
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```
Save: `Ctrl+X` → `Y` → `Enter`
---
## Testing
### Test 1: Verify Slack Connection
```bash
node ~/companyProjects/prBot/index.js test-slack
```
**Expected output:**
```
✔ Test message sent to Slack! Check your channel.
```
You should see a "✅ PR Review Bot is connected!" message in your Slack channel.
### Test 2: Review a PR
Go into any of your project repos and run:
```bash
cd ~/companyProjects/your-project
node ~/companyProjects/prBot/index.js review --base develop
```
**Expected output:**
```
🔍 PR Review CLI
✔ Diff ready: your-branch → develop (5 files, +202 -149)
⠋ Reviewing with groq (file 1/5: PaymentLinkController.java)...
⠋ Reviewing with groq (file 2/5: PaymentLinkServiceImpl.java)...
⠋ Reviewing with groq (file 3/5: PaymentCallbackServiceImpl.java)...
⠋ Reviewing with groq (file 4/5: PaymentCallbackController.java)...
⠋ Reviewing with groq (file 5/5: PaymentRedirectionServiceImpl.java)...
⠋ Aggregating 5 file reviews...
✔ AI review complete — 5 files reviewed (llama-3.3-70b-versatile)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Review Results
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## Summary
[AI-generated summary with severity ratings]
## Risk Level: Medium
## Issues Found
### 🔴 Null Safety Issues
- 🔴 [PaymentCallbackController.java:67]: [R1] `orderContext` — ...
### 🔒 Security Issues
- 🔴 [PaymentLinkServiceImpl.java:43]: [R5] `PAYMENT_GATEWAY_API_KEY` — ...
[... categorized by R# rules ...]
✔ Review posted to Slack ✅
Done! 🎉
```
> ⏱️ **Timing:** Expect ~2-3 minutes for a 5-file review on Groq free tier (15s cooldown between files to respect rate limits).
---
## Usage
### Basic Commands
```bash
# Review current branch against develop (default)
node ~/companyProjects/prBot/index.js review
# Review against any base branch
node ~/companyProjects/prBot/index.js review --base main
node ~/companyProjects/prBot/index.js review --base centralbank-develop
node ~/companyProjects/prBot/index.js review --base feature/payment-module
node ~/companyProjects/prBot/index.js review --base release/v2.1
# Review without posting to Slack (terminal only)
node ~/companyProjects/prBot/index.js review --base develop --no-slack
# Review a repo from a different directory
node ~/companyProjects/prBot/index.js review --base develop --repo ~/companyProjects/other-project
# Focus on specific areas only
node ~/companyProjects/prBot/index.js review --base develop --focus bugs,security
```
### Set up your Git username
```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@company.com"
```
### All CLI Options
```
pr-review review [options]
Options:
  -b, --base <branch>     Base branch to compare against (default: "develop")
  -r, --repo <path>       Path to git repo (default: current directory)
  --no-slack              Skip posting to Slack, print to terminal only
  --model <model>         AI model to use (default: llama-3.3-70b-versatile)
  --focus <areas>         Comma-separated focus areas (default: bugs,security,performance,style)
pr-review test-slack       Test your Slack webhook connection
```
---
## Architecture: Review Pipeline
```
┌──────────────────────────────────────────────────────────────┐
│                        index.js                              │
│                    (CLI Orchestrator)                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Parse CLI args (--base, --repo, --focus)                 │
│  2. Validate env vars (GROQ_API_KEY, SLACK_WEBHOOK_URL)      │
│  3. Call local-git.js → get diff                             │
│  4. Call ai-reviewer.js → enrich + review + aggregate        │
│  5. Call slack.js → format + post                            │
│                                                              │
└──────────┬──────────────────┬──────────────────┬─────────────┘
           │                  │                  │
     ┌─────▼─────┐    ┌──────▼──────┐    ┌──────▼──────┐
     │ local-git  │    │ ai-reviewer │    │   slack     │
     │            │    │             │    │             │
     │ git diff   │    │ Per-file:   │    │ markdown →  │
     │ git show   │    │  R1-R21     │    │ Slack mrkdwn│
     │ git log    │    │  checklist  │    │             │
     │            │    │             │    │ Section     │
     │ Returns:   │    │ Aggregation:│    │ splitting   │
     │ - patches  │    │  Cross-file │    │             │
     │ - stats    │    │  Dedup      │    │ Multi-msg   │
     │ - metadata │    │  Severity   │    │ support     │
     └────────────┘    └─────────────┘    └─────────────┘
```
---
## Why Groq?
| Factor | Groq | Google Gemini | OpenAI |
|---|---|---|---|
| **Cost** | ✅ **Completely free** | ⚠️ Free tier, but quota can be 0 | ❌ $0.01-0.15 per review |
| **Speed** | ✅ ~2-5 seconds per file | ~5-10 seconds | ~10-15 seconds |
| **Signup** | ✅ Google account, instant | ✅ Google account | Needs billing setup |
| **Rate limit** | ✅ 12K TPM (handled by multi-pass) | Varies, can be 0 | Depends on spend |
| **Model quality** | ✅ Llama 3.3 70B (excellent) | Gemini Flash (good) | GPT-4o (excellent) |
| **Credit card needed** | ✅ **No** | ✅ No | ❌ Yes (for API) |
**Groq** is the best choice because:
1. **Truly free** — no billing, no credit card, no hidden limits
2. **Blazing fast** — Groq uses custom LPU hardware, reviews complete in seconds
3. **High quality** — Llama 3.3 70B provides GPT-4-level code analysis
4. **Multi-pass friendly** — 12K TPM limit is handled gracefully by file-by-file review + auto-retry
5. **No org permissions needed** — just your personal account
---
## Troubleshooting
| Error | Fix |
|---|---|
| `fetch is not defined` | You're on Node <18. The tool handles this with `node-fetch`, just ensure deps are installed |
| `Not a git repository` | Run the command from inside a git repo, or use `--repo /path/to/repo` |
| `Base branch not found` | Run `git fetch origin <branch>` first |
| `GROQ_API_KEY not set` | Make sure `.env` exists in `~/companyProjects/prBot/` with your key |
| `Slack webhook failed` | Verify the webhook URL in `.env` and test with `node index.js test-slack` |
| `No changes found` | Commit your changes first (`git add . && git commit`) |
| `Rate limit (429)` | Auto-handled with retry. If persistent, wait 60s and try again |
| `Request too large (413)` | Too many changed files. Try reviewing fewer files with `git diff` filtering |
---
## Security Notes
- ⚠️ **Never commit `.env`** — it's in `.gitignore` by default
- 🔄 **Rotate keys** if they're ever exposed (Groq: [console.groq.com/keys](https://console.groq.com/keys))
- 🔒 Your code **never leaves your machine** via GitHub — diffs are read locally with `git diff`, only the diff text is sent to Groq's API for analysis
