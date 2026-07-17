# Agent Your Tax Docs — a non-specific cheat sheet

See https://notebooklm.google.com/notebook/d8559832-1faa-4eff-9d61-691e6bd13271/artifact/5657ea07-96d9-4ae7-83f0-1894dde9a79e?utm_source=nlm_web_share&utm_medium=google_oo&utm_campaign=art_share_1&utm_content=&utm_smc=nlm_web_share_google_oo_art_share_1_
and
https://notebooklm.google.com/notebook/d8559832-1faa-4eff-9d61-691e6bd13271/artifact/5657ea07-96d9-4ae7-83f0-1894dde9a79e?utm_source=nlm_web_share&utm_medium=google_oo&utm_campaign=art_share_1&utm_content=&utm_smc=nlm_web_share_google_oo_art_share_1_

How to point a free/local AI agent at your own tax backlog without giving up control
(or your privacy). You delegate the *consolidation*, not the *responsibility* — the
agent drafts, you own the sign-off.

## 1. What you're actually doing
You're hiring a very fast, very cheap analyst that works from your source documents.
The bottleneck was never ability. It was the cost of a human's time. Agents flip that
economics. The scarce skill becomes: set up access, write a clear brief, check the output.

## 2. What to grant the agent (least privilege)
- Read-only access to a cloud folder holding your statements. No write, no send, no delete.
- Read-only email access (to pull receipt emails) — only if you pay for subs by card.
- Run it on a personal machine that's sandboxed, not a shared/prod box.
- Don't hand over anything you don't need categorized. Redact what's irrelevant.

## 3. Source material to point it at
- Every bank/card statement as PDF — all accounts, not just the main one. The
  stray accounts are usually where the interesting stuff hides.
- Any manual spreadsheets a prior bookkeeper kept — especially ones where personal
  and business expenses are tangled together.
- Email receipts for subscriptions/tools that never show up in a statement.
- Loan schedules + payout quotes for any financed equipment.
- Anything about property use changing (home office → rental), since the
  cost-split rules change the moment the use changes.

## 4. Judgment calls only YOU can make
Hand these to the agent as a short brief + historical defaults; let it ask when ambiguous.
- Business-use % on mixed expenses (vehicle, home, phone, internet).
- Personal vs business classification on tangled rows.
- Cost base of any disposed asset.
- Which financial year each transaction belongs to.

## 5. What to always re-check (where banks and bookkeepers drift)
- Bank debits vs the source loan docs — they often disagree. Trust the source document.
- "Rates" that are actually the interest-slice of a repayment mislabelled as an interest rate.
- Asset disposals buried as routine transfers — rebuild the cost base from the finance docs.
- Expenses living only in email, never in the bank.
- Estimates you couldn't source a statement for (FX, interest) — keep them flagged, not buried.

## 6. The win
Have the agent build the whole ledger first, then ask you only the judgment calls a
human must own. And have it write down how it did the work — so next year starts
faster and more accurate. It isn't a chatbot you talk to. It's a colleague you delegate to.

---
*Non-specific by design. Drop this into NotebookLM to generate an explainer if you like.*
