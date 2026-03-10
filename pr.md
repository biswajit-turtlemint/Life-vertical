# 🤖 PR Review Bot — AI-Powered Code Review CLI
A lightweight CLI tool that reviews your code changes using **AI** and posts structured suggestions to **Slack** — with **zero org-level permissions** required.
---
## What This Does
```
You run a single command from your project repo
        ↓
CLI reads your git diff locally (your branch vs any base branch)
        ↓
Sends the diff to Groq AI (Llama 3.3 70B) for review
        ↓
AI analyzes for: 🐛 Bugs, 🔒 Security, ⚡ Performance, 🎨 Style issues
        ↓
Posts a formatted review to your Slack channel
        ↓
Team discusses and resolves suggestions
```
**No GitHub tokens. No org admin access. Completely free.**
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
├── index.js              # CLI entry point — handles commands & orchestration
├── lib/
│   ├── local-git.js      # Reads git diff locally using git commands
│   ├── ai-reviewer.js    # Sends diff to Groq/Gemini AI for analysis
│   └── slack.js          # Posts formatted results to Slack via webhook
├── .env                  # Your API keys (never commit this!)
├── .env.example          # Template showing required keys
├── .gitignore            # Ignores node_modules and .env
├── package.json          # Node.js dependencies
└── README.md             # This file
```
### How each file works:
- **`index.js`** — The main CLI. Parses your command (`review` or `test-slack`), validates config, orchestrates the pipeline: diff → AI → Slack.
- **`lib/local-git.js`** — Runs `git diff` between your current branch and the base branch. Parses the unified diff format into per-file changes. No GitHub API or token needed.
- **`lib/ai-reviewer.js`** — Takes the parsed diff and sends it to the AI (Groq) with a code review prompt. Returns categorized suggestions (bugs, security, performance, style).
- **`lib/slack.js`** — Formats the AI review into Slack Block Kit messages and posts via incoming webhook.
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
✔ Diff ready: your-branch → develop (X files, +N -N)
✔ AI review complete (llama-3.3-70b-versatile)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Review Results
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## Summary
[AI-generated summary of your changes]
## Risk Level: Low/Medium/High
## Issues Found
[Categorized suggestions]
✔ Review posted to Slack ✅
Done! 🎉
```
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

### Setup up your Github username

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
## Why Groq?
| Factor | Groq | Google Gemini | OpenAI |
|---|---|---|---|
| **Cost** | ✅ **Completely free** | ⚠️ Free tier, but quota can be 0 | ❌ $0.01-0.15 per review |
| **Speed** | ✅ ~2-5 seconds per review | ~5-10 seconds | ~10-15 seconds |
| **Signup** | ✅ Google account, instant | ✅ Google account | Needs billing setup |
| **Rate limit** | ✅ ~30 req/min | Varies, can be 0 | Depends on spend |
| **Model quality** | ✅ Llama 3.3 70B (excellent) | Gemini Flash (good) | GPT-4o (excellent) |
| **Credit card needed** | ✅ **No** | ✅ No | ❌ Yes (for API) |
**Groq** is the best choice because:
1. **Truly free** — no billing, no credit card, no hidden limits
2. **Blazing fast** — Groq uses custom LPU hardware, reviews complete in seconds
3. **High quality** — Llama 3.3 70B provides GPT-4-level code analysis
4. **Generous limits** — hundreds of reviews per day on the free tier
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
---
## Security Notes
- ⚠️ **Never commit `.env`** — it's in `.gitignore` by default
- 🔄 **Rotate keys** if they're ever exposed (Groq: [console.groq.com/keys](https://console.groq.com/keys))
- 🔒 Your code **never leaves your machine** via GitHub — diffs are read locally with `git diff`, only the diff text is sent to Groq's API for analysis
