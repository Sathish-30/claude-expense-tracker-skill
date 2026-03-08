# 🧾 claude-daily-expense-tracker-skill

A **Claude Skill** that automatically scans your Gmail for today's financial transaction emails — UPI credits, UPI debits, bank alerts, and credit card transactions — and saves a clean, structured daily expense note directly into your Obsidian vault under `Daily Expenses/YYYY-MM-DD.md`.

Just say **"log my expenses"** or **"track today's transactions"** and Claude does the rest.

---

## ✨ Features

- 📧 Scans Gmail for today's bank/UPI transaction emails automatically
- 💚 Separates **credited** and **debited** transactions into clean tables
- 🏦 Detects transaction type — **UPI**, **Credit Card**, or **Bank Transfer**
- 📓 Saves a structured daily note into your **Obsidian vault**
- 🔁 Call it twice in a day — it overwrites the note with the latest data
- 🏷️ Captures amount, party name, bank, UPI reference number, and time

---

## 🛠️ Prerequisites

### 1. Obsidian Setup

You need two things on the Obsidian side:

**a. Install the Local REST API community plugin**

This plugin exposes your Obsidian vault over a local HTTP server, which the Docker MCP server connects to.

1. Open Obsidian → `Settings` → `Community Plugins` → `Browse`
2. Search for **Local REST API** and install it
3. Enable the plugin and note the **API Key** shown in its settings
4. Make sure the plugin is running (you should see a green indicator in the status bar)

> 📖 Plugin repo: [coddingtonbear/obsidian-local-rest-api](https://github.com/coddingtonbear/obsidian-local-rest-api)

**b. Create the `Daily Expenses` folder in your vault**

Inside Obsidian, create a folder called exactly `Daily Expenses` at the root of your vault. The skill will write one note per day into this folder (`YYYY-MM-DD.md`).

---

### 2. Docker Desktop + Obsidian MCP Server

The skill uses Docker's MCP toolkit to connect Claude Desktop to your Obsidian vault.

**a. Install Docker Desktop**

Download and install Docker Desktop from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/).

**b. Enable the Obsidian MCP server in Docker Desktop**

1. Open **Docker Desktop**
2. Go to the **MCP Toolkit** section (available in Docker Desktop 4.x+)
3. Find the **Obsidian** MCP server and click **Enable**
4. When prompted, provide:
   - Your **Obsidian vault path** (the folder on your machine)
   - The **API Key** from the Local REST API plugin

Docker will run the MCP server as a container that bridges Claude Desktop to your vault.

---

### 3. Connect Claude Desktop

**a. Connect Docker MCP to Claude Desktop**

1. Open **Claude Desktop** → `Settings` → `Developer` → `MCP Servers`
2. Add the Docker MCP server endpoint (Docker Desktop provides the exact config snippet — copy it from the MCP Toolkit panel)
3. Restart Claude Desktop

**b. Connect Gmail to Claude Desktop**

1. In Claude Desktop, go to `Settings` → `Connectors`
2. Find **Gmail** and click **Connect**
3. Authorize Claude to access your Gmail account via Google OAuth

**c. Verify everything is connected**

You should see both connectors active in Claude Desktop. Run these quick tests to confirm:

**Test Gmail:**
```
"Show me my last 3 emails"
```
Claude should fetch and display recent emails from your inbox.

**Test Obsidian:**
```
"List the files in my Obsidian vault"
```
Claude should respond with the files and folders in your vault root.

**Test both together:**
```
"Check my Obsidian vault and show me what's inside the Daily Expenses folder"
```
Claude should list any notes already in that folder (or say it's empty).

---

### 4. Install the Skill

**Option A — Download the `.skill` file (recommended)**

1. Go to the [Releases](../../releases) page and download `daily-expense-tracker.skill`
2. In Claude Desktop, go to `Settings` → `Skills`
3. Click **Install Skill** and select the downloaded `.skill` file

**Option B — Clone and add manually**

```bash
git clone https://github.com/your-username/claude-daily-expense-tracker-skill.git
```

Then copy the `SKILL.md` file:

1. In Claude Desktop, go to `Settings` → `Skills` → **Add Skill**
2. Point it to the `daily-expense-tracker/SKILL.md` file from the cloned repo

---

## 🚀 Usage

Once everything is set up, simply tell Claude:

```
"Log my expenses"
```
```
"Track today's transactions"
```
```
"Save my expenses to Obsidian"
```
```
"Update my daily expense note"
```

Claude will scan your Gmail, extract all financial transactions for today, and save a note like this to `Daily Expenses/2026-03-08.md`:

```markdown
# Daily Expenses — 2026-03-08
> Last updated: 16:30 IST

## Summary
| Metric          | Value        |
|-----------------|--------------|
| Total Credited  | ₹15,000.00   |
| Total Debited   | ₹51.00       |
| Net             | ₹14,949.00 (net gain) |
| Transactions    | 3            |

## Transactions

### 💚 Credited
| Time  | Amount      | Type | From / Party                        | Bank | Reference    |
|-------|-------------|------|-------------------------------------|------|--------------|
| 16:17 | ₹15,000.00  | UPI  | xxxxxx | HDFC | 643308835684 |

### 🔴 Debited
| Time  | Amount  | Type | To / Party                                    | Bank | Reference    |
|-------|---------|------|-----------------------------------------------|------|--------------|
| 09:11 | ₹49.00  | UPI  | yyyyyyyy | HDFC | 643304284063 |
| 11:23 | ₹2.00   | UPI  | zzzzzzzz | HDFC | 643318313515 |
```

> If you run the skill more than once in a day, it will append a fresh updated note below the previous one, timestamped so you always know which version is latest.

---

## 🏦 Supported Banks

The skill recognises transaction emails from all major Indian banks:

| Bank | Sender Domain |
|------|--------------|
| HDFC | `hdfcbank.net`, `hdfcbank.bank.in` |
| SBI | `sbi.co.in` |
| ICICI | `icicibank.com` |
| Axis | `axisbank.com` |
| Kotak | `kotak.com` |

UPI apps (PhonePe, GPay, Paytm) are also detected automatically via VPA keywords in the email body.

---

## 📁 Repo Structure

```
claude-daily-expense-tracker-skill/
├── SKILL.md # The skill definition
├── daily-expense-tracker.skill  # Packaged skill file (installable)
└── README.md
```

---

## 📄 License

MIT License — feel free to use, modify, and distribute.