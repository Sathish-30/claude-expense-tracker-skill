---
name: daily-expense-tracker
description: >
  Scans Gmail for today's financial transaction emails (UPI credits, UPI debits,
  bank credits, bank debits, credit card transactions) and saves a structured
  daily expense note into Obsidian under the "Daily Expenses" folder.
  Overwrites the existing note if called more than once on the same day.
  Use this skill whenever the user says "log my expenses", "track today's transactions",
  "save my expenses to Obsidian", "update my daily expense note", "record today's payments",
  or anything related to capturing daily financial activity into Obsidian.
---

# Daily Expense Tracker Skill

This skill pulls today's financial transaction emails from Gmail and writes a
clean, structured expense note into Obsidian under `Daily Expenses/YYYY-MM-DD.md`.
If that note already exists (i.e. skill was run earlier today), it is overwritten
with the latest data so the note always reflects all transactions for the day.

## Required Tools
- `search_gmail_messages` — fetch emails
- `obsidian_append_content` — write/overwrite the note (used with full content replacement)
- `obsidian_list_files_in_dir` — check if today's note already exists
- `obsidian_get_file_contents` — read existing note if needed

---

## Workflow

### Step 1: Determine Today's Date and Current IST Time

Get the current date and time in **IST (Asia/Kolkata, UTC+5:30)**. This is critical —
always derive the current time from the system's UTC clock and add 5 hours 30 minutes
to get IST. Never guess or reuse a previously seen timestamp.

- **Date**: format as `YYYY-MM-DD` (e.g. `2026-03-08`) — used for the note filename
- **Time**: format as `HH:MM IST` in 24-hour format (e.g. `22:45 IST`) — used as the "Last updated" timestamp in the note

The target note path will be: `Daily Expenses/YYYY-MM-DD.md`

> ⚠️ The "Last updated" time must reflect the actual current time at the moment the
> skill runs, not the time of any email or any previously written note. Compute it
> fresh every time the skill is triggered.

### Step 2: Fetch Today's Transaction Emails
Search Gmail for financial transaction emails received today using:

```
query: is:unread (credited OR debited OR "UPI" OR "transaction" OR "payment") after:YYYY/MM/DD
```

Also search for read emails in case the user already opened them:
```
query: (credited OR debited OR UPI OR transaction OR "credit card") after:YYYY/MM/DD
```

Collect from each matching email:
- Sender
- Subject
- Date/time
- Snippet (for amount and transaction details)

### Step 3: Extract Transaction Details
For each email, parse out:

| Field | How to extract |
|-------|---------------|
| **Amount** | Look for "Rs.", "INR", "₹" followed by a number in subject or snippet |
| **Type** | Classify as `UPI` if snippet mentions "UPI", "VPA", "PhonePe", "GPay", "Paytm"; else `Credit Card` if mentions "card", "CC"; else `Bank Transfer` |
| **Direction** | `Credited` if email says "credited", "received", "added"; `Debited` if says "debited", "paid", "deducted", "spent" |
| **Merchant/Party** | Extract payee or payer name if mentioned (e.g. "by VPA xyz@okicici" → party is xyz) |
| **Bank** | Infer from sender domain (hdfcbank → HDFC, sbi → SBI, icici → ICICI, etc.) |
| **Reference** | UPI ref number or transaction ID if present in snippet |

Skip emails that don't contain a clear monetary amount.

### Step 4: Check if Note Already Exists
Use `obsidian_list_files_in_dir` on `Daily Expenses/` to check if `YYYY-MM-DD.md` already exists.

- If it **exists**: the skill will overwrite it by writing a completely fresh note using `obsidian_append_content` after deleting old content. Since Obsidian MCP only supports append, use a clear overwrite strategy: write a header comment `<!-- updated: HH:MM -->` at the top so each save is timestamped.
- If it **does not exist**: create it fresh.

> Note: Because the MCP tool only supports `append_content`, overwrite is achieved by
> writing the complete note content each time (the skill manages this by always
> constructing the full note from scratch from today's emails).

### Step 5: Build the Note
Construct the note using this exact template:

```markdown
# Daily Expenses — YYYY-MM-DD
> Last updated: HH:MM IST  

## Summary
| Metric | Value |
|--------|-------|
| Total Credited | ₹X,XXX |
| Total Debited | ₹X,XXX |
| Net | ₹X,XXX (positive = net gain) |
| Transactions | N |

---

## Transactions

### 💚 Credited
| Time | Amount | Type | From / Party | Bank | Reference |
|------|--------|------|-------------|------|-----------|
| HH:MM | ₹XX,XXX | UPI / Bank Transfer | Name | HDFC | XXXXXXXXX |

### 🔴 Debited
| Time | Amount | Type | To / Party | Bank | Reference |
|------|--------|------|-----------|------|-----------|
| HH:MM | ₹XX,XXX | UPI / Credit Card | Merchant | HDFC | XXXXXXXXX |

---

## Raw Sources

- HDFC Bank InstaAlerts: "Account update for your HDFC Bank A/c"
```

Rules for building the note:
- Sort transactions within each section by time (earliest first)
- If no credited transactions exist, show `*No credits today*` under that section
- If no debited transactions exist, show `*No debits today*` under that section
- Calculate totals accurately — sum all credited amounts and all debited amounts separately
- Net = Total Credited - Total Debited

### Step 6: Write to Obsidian
Use `obsidian_append_content` with the full note content to write to:
```
Daily Expenses/YYYY-MM-DD.md
```

Since this tool appends, if the file already existed, the new version will be appended below the old. This is acceptable — the most recent version is always at the bottom and clearly timestamped with "Last updated".

### Step 7: Confirm to User
After writing, respond with a short confirmation:

> ✅ Expense note saved to `Daily Expenses/YYYY-MM-DD.md`
> 💚 Credited: ₹X,XXX (N transactions)
> 🔴 Debited: ₹X,XXX (N transactions)
> 📊 Net: ₹X,XXX

---

## Classification Reference

### Transaction Type
- **UPI** — keywords: UPI, VPA, @okicici, @oksbi, @ybl, PhonePe, GPay, Paytm
- **Credit Card** — keywords: credit card, CC, card ending, card no
- **Bank Transfer** — NEFT, RTGS, IMPS, direct transfer, no UPI indicators

### Direction
- **Credited** — keywords: credited, received, deposited, added, incoming
- **Debited** — keywords: debited, paid, deducted, spent, sent, outgoing, purchase

### Common Bank Sender Domains
- HDFC: `hdfcbank.net`, `hdfcbank.bank.in`
- SBI: `sbi.co.in`
- ICICI: `icicibank.com`
- Axis: `axisbank.com`
- Kotak: `kotak.com`

---

## Edge Cases
- **Multiple transactions in one email** — some banks batch transactions; parse each separately if amounts are listed individually
- **Ambiguous direction** — if unclear, default to Debited and add a ⚠️ flag in the Reference column
- **Non-INR amounts** — skip or note as `Foreign currency` in the Type column
- **Duplicate emails** — if two emails describe the same transaction (same ref number), include only once