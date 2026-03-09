---
name: daily-expense-tracker
description: >
  Scans Gmail for today's financial transaction emails (UPI credits, UPI debits,
  bank credits, bank debits, credit card transactions) and saves a structured
  daily expense note into Obsidian under the "Daily Expenses" folder.
  Always appends exactly ONE entry per invocation. Never deletes the file.
  Use this skill whenever the user says "log my expenses", "track today's transactions",
  "save my expenses to Obsidian", "update my daily expense note", "record today's payments",
  or anything related to capturing daily financial activity into Obsidian.
---

# Daily Expense Tracker Skill

This skill pulls today's financial transaction emails from Gmail and writes a
clean, structured expense note into Obsidian under `Daily Expenses/YYYY-MM-DD.md`.

> ⚠️ CRITICAL RULES — must be followed strictly every invocation:
> 1. **ONE append only** — call `obsidian_append_content` exactly once per skill run. Never call it twice.
> 2. **Never delete** the Obsidian file. Do not call `obsidian_delete_file` under any circumstance.
> 3. Collect ALL emails first, then build the note, then write once.

## Required Tools
- `search_gmail_messages` — fetch emails
- `obsidian_append_content` — write the note (called exactly ONCE per run)
- `obsidian_list_files_in_dir` — check if today's note already exists

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

### Step 2: Fetch ALL of Today's Transaction Emails

Run both searches below and collect results from BOTH before doing anything else.

Search 1 — unread emails:
```
query: is:unread (credited OR debited OR "UPI" OR "transaction" OR "payment") after:YYYY/MM/DD
```

Search 2 — all emails (including already-read ones):
```
query: (credited OR debited OR UPI OR transaction OR "credit card") after:YYYY/MM/DD
```

Collect from each matching email:
- Sender
- Subject
- Date/time
- Snippet (for amount and transaction details)

Deduplicate by message ID — if the same email appears in both searches, count it once.

### Step 3: Extract Transaction Details

#### ⚠️ Confirmed vs Pending — CRITICAL RULE
**Only include transactions that have ALREADY settled in the bank account.**

**EXCLUDE (skip entirely) — Pending/Initiated transactions:**
- IRCTC cancellation emails saying "Please collect Rs. X from agent" — this means the refund is *initiated*, not received
- Any email with phrases like: "refund initiated", "refund will be processed", "collect from agent", "will be credited within X days", "processing your refund"
- Payment gateway confirmations of *outgoing* refund initiation
- Ticket booking cancellation notifications without a bank credit confirmation

**INCLUDE (confirmed) — Settled transactions:**
- Bank debit alerts (e.g. HDFC InstaAlert: "Rs. X debited from account") — money is gone, confirmed
- Bank credit alerts (e.g. "Rs. X credited to your account XXXX") — money is received, confirmed
- UPI debit/credit alerts from the bank itself (not from a merchant)
- Credit card transaction alerts

**Rule of thumb**: If the email comes from the *bank's own alert system* (e.g. `alerts@hdfcbank.bank.in`), it's confirmed. If it comes from a *merchant or service* (e.g. IRCTC, Swiggy, Amazon), treat it as a notification only — verify it's a confirmed bank-side event before including.

For each confirmed email, parse out:

| Field | How to extract |
|-------|---------------|
| **Amount** | Look for "Rs.", "INR", "₹" followed by a number in subject or snippet |
| **Type** | Classify as `UPI` if snippet mentions "UPI", "VPA", "PhonePe", "GPay", "Paytm"; else `Credit Card` if mentions "card", "CC"; else `Bank Transfer` |
| **Direction** | `Credited` if email says "credited", "received", "added"; `Debited` if says "debited", "paid", "deducted", "spent" |
| **Merchant/Party** | Extract payee or payer name if mentioned (e.g. "by VPA xyz@okicici" → party is xyz) |
| **Bank** | Infer from sender domain (hdfcbank → HDFC, sbi → SBI, icici → ICICI, etc.) |
| **Reference** | UPI ref number or transaction ID if present in snippet |

Skip emails that don't contain a clear monetary amount.
Skip emails that represent pending/initiated transactions per the rules above.

### Step 4: Build the Complete Note

Construct the full note content in memory using this exact template:

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
<!-- Email subjects used as source for this note -->
- HDFC Bank InstaAlerts: "Account update for your HDFC Bank A/c"
```

Rules for building the note:
- Sort transactions within each section by time (earliest first)
- If no credited transactions exist, show `*No credits today*` under that section
- If no debited transactions exist, show `*No debits today*` under that section
- Calculate totals accurately — sum all credited amounts and all debited amounts separately
- Net = Total Credited - Total Debited

### Step 5: Write to Obsidian — EXACTLY ONCE

Call `obsidian_append_content` **once and only once** with the full note content to:
```
Daily Expenses/YYYY-MM-DD.md
```

> ⚠️ Do NOT call `obsidian_append_content` a second time for any reason — not to fix,
> not to retry, not to add a separator. One call. Done.

### Step 6: Confirm to User
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
- **IRCTC / merchant refund emails** — phrases like "Please collect Rs. X from agent" or "refund initiated" mean the money has NOT been received yet. **Do NOT include these as credits.** Only include a refund when the bank itself sends a credit alert.
- **Pending refunds** — if you skip any transactions due to pending status, mention them briefly in the confirmation message to the user so they know what was excluded and why.