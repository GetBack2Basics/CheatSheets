# Agent Your Tax Docs

See https://notebooklm.google.com/notebook/d8559832-1faa-4eff-9d61-691e6bd13271/artifact/5657ea07-96d9-4ae7-83f0-1894dde9a79e?utm_source=nlm_web_share&utm_medium=google_oo&utm_campaign=art_share_1&utm_content=&utm_smc=nlm_web_share_google_oo_art_share_1_
and
https://notebooklm.google.com/notebook/d8559832-1faa-4eff-9d61-691e6bd13271/artifact/5657ea07-96d9-4ae7-83f0-1894dde9a79e?utm_source=nlm_web_share&utm_medium=google_oo&utm_campaign=art_share_1&utm_content=&utm_smc=nlm_web_share_google_oo_art_share_1_

How to point a free/local AI agent at your own tax backlog without giving up control
(or your privacy). You delegate the *consolidation*, not the *responsibility* — the
agent drafts, you own the sign-off.
You save money and time by handing your tax consultant what they want. You save on not having to do the drudge work as well!

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
- Prior-year consolidated ledgers when they exist. A verified draft workbook is
  usually more reliable than re-parsing prior-year bank PDFs, which are often
  incomplete or formatted differently year-to-year.

## 4. Judgment calls only YOU can make
Hand these to the agent as a short brief + historical defaults; let it ask when ambiguous.
- Business-use % on mixed expenses (vehicle, home, phone, internet).
- Personal vs business classification on tangled rows.
- Cost base of any disposed asset.
- Which financial year each transaction belongs to.
- Occupancy timeline for any property that changed from owner-occupied to rental,
  because the split rules change on the rental start date, not the financial year start.
- Primary point of contact for the tax engagement, especially when spouses are joint
  owners or co-signatories.

## 5. What to always re-check (where banks and bookkeepers drift)
- Bank debits vs the source loan docs — they often disagree. Trust the source document.
- "Rates" that are actually the interest-slice of a repayment mislabelled as an interest rate.
- Asset disposals buried as routine transfers — rebuild the cost base from the finance docs.
- Expenses living only in email, never in the bank.
- Estimates you couldn't source a statement for (FX, interest) — keep them flagged, not buried.
- **PDF text extraction:** many bank PDFs are scanned images, not text layers. If
  `pdfminer` / `PyMuPDF` text extraction returns empty or garbled output, fall back
  to OCR or vision-based inspection before assuming the document is blank.
- **Header drift across years:** precompute/report scripts should map by header name,
  not fixed column index. When a new column is inserted or removed, an index-based
  reader silently returns wrong totals instead of failing loudly.
- **Category name drift:** old ledgers use informal categories like
  `Other Personal Expense` or `Purchases`. Map them explicitly to the current schema;
  do not assume a prior year's category names match the current year.

## 6. Agent ops lessons from actual tax packs
- **Prefer normalization over rebuild** when a verified prior-year ledger exists.
  Re-parsing prior-year bank PDFs often yields 0 transactions because formats change
  or files are incomplete in the new working folder.
- **Internal transfers:** detect by description text (`TFR`, `TRANSFER TO/FROM`,
  `MOBILE`) rather than amount-pairing. Amount-pairing found only 2 pairs in
  FY25-26 testing; text matching found hundreds.
- **Windows save-path trap:** `os.path.join(BASE, "..", "Sagacity", "file.xlsx")`
  can raise `OSError: [Errno 22] Invalid argument` even when standalone
  `openpyxl.Workbook().save(path)` succeeds on the same path. Prefer absolute
  hardcoded paths or write-to-temp-then-copy.
- **Old P&L formulas are not portable.** Do not paste spreadsheet formulas from an
  old workbook into a new schema. Rebuild P&L totals from normalized transaction rows
  so the numbers tie back to the Transactions sheet.
- **Bundle income statements by payer and year.** ATO/pre-fill income statements
  often carry suffixes like `2526` that are not authoritative. Inspect the PDF body
  for payer name, TFN, period dates, and amounts before filing.
- **Ask once, record forever.** Occupation, co-signatory/spouse details, business address, and
  prior-year lodgement status should be captured in an `Explanatory_Notes` worksheet
  so next year starts faster.

## 7. The win
Have the agent build the whole ledger first, then ask you only the judgment calls a
human must own. And have it write down how it did the work — so next year starts
faster and more accurately. It isn't a chatbot you talk to. It's a colleague you delegate to.
