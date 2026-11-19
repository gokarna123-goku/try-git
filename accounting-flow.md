# Accounting flow

How Accounting works: who does what, how money moves, how every number is calculated.
Short on purpose. Code lives in `backend/src/products/accounting` and `frontend/apps/tenant/src/features/accounting`.

---

## 1. The one rule

**The journal is the only truth.** Every invoice, bill, payment, expense and voucher becomes a balanced journal entry (Debit = Credit).
Dashboard, reports, party balances and VAT are all *calculated from the journal* when you open them. Nothing is stored twice, so reports can never disagree with the books.

- Posted entries cannot be edited or deleted (database triggers). Fix = **cancel** (a reversing entry) or a **credit / debit note**.
- The database refuses an unbalanced entry, even if code has a bug.
- Amounts are NPR, exact decimals (never floats), rounded half-up to paisa.

---

## 2. Who does what

| Role | Can do |
|---|---|
| **Owner / Admin** | Everything: all documents, purchases, reports, VAT/TDS, banking, journal, settings, period lock, year-end, audit log, cost prices |
| **Member** (staff) | Daily billing: parties, items, sales documents, payments received, expenses, dashboard. **Cannot** see purchase cost/margin, reports, tax, settings, journal or lock periods |
| **Crown team** | No access to a workspace's books |
| **Demo visitor** | Own private copy of the demo books (per browser). Nothing reaches real data |

Permissions are checked on the server for every request; the UI only hides what you cannot do.
Plan limits and the Accounting subscription are checked before any of this.

---

## 3. Daily flow (what a user does)

```
Set up once ──► Parties & Items ──► Sell / Buy ──► Get paid / Pay ──► Bank ──► Reports & VAT ──► Close year
```

1. **Set up once** (Settings): business PAN, VAT registered or not, VAT number, date display (BS/AD), fiscal year, numbering prefixes, tax rates (verify TDS), opening balances.
2. **Parties** (customers, suppliers) and **Items** (goods with stock, or services).
3. **Sales**: Estimate → Sales order → Delivery challan → **Invoice** → Credit note (returns).
4. **Purchases**: Purchase order → **Bill** → Debit note. **Expenses** for things paid at once (rent, fuel…).
5. **Payments & bank**: receive money / pay money, split over documents; cheques; transfers; statement reconciliation.
6. **Books**: Journal vouchers, Day book, Trial balance, Recurring, Audit log.
7. **Reports & tax**: P&L, balance sheet, cash flow, ageing, VAT register, TDS, return summary.
8. **Year end**: close the fiscal year (Shrawan 1 – Ashadh end).

Sidebar has 9 entries; the pages inside a section are tabs at the top:
Dashboard · Sales · Purchases · Payments & bank · Parties · Items · Books · Reports & tax · Settings.

---

## 4. What each document posts

| Document | Journal entry | Stock |
|---|---|---|
| Estimate, Sales order, Delivery challan, Purchase order | **Nothing** (working papers) | none |
| **Sales invoice** | Dr Receivables · Cr Sales/Service income + Output VAT (± round-off) | Dr COGS / Cr Inventory at average cost |
| Credit note | Mirror of the invoice | stock back in |
| **Purchase bill** | Dr Purchases/Inventory/Expense + Input VAT · Cr Payables | stock in |
| Debit note | Mirror of the bill | stock out |
| **Expense** | As a bill; Cr Cash/Bank if paid now, else Payables | none |
| Receipt (money in) | Dr Cash/Bank (+ TDS Receivable) · Cr Receivables | none |
| Payment (money out) | Dr Payables · Cr Cash/Bank (+ TDS Payable) | none |
| Transfer | Dr one cash/bank · Cr another | none |
| Manual voucher | Whatever balanced lines you enter | none |

Status of a document: `DRAFT → POSTED → PARTIALLY_PAID → PAID`, or `CANCELLED`. Drafts can be edited and deleted; anything issued is locked.

---

## 5. How numbers are calculated

**Per line** (`lib/money.ts`)
```
gross    = qty × rate
discount = gross × discount%  +  discount amount
net      = gross − discount
```
**Price includes VAT?** The rate is divided by 1.13 first, so the VAT is carved out of the price.

**Invoice-level discount** is shared across lines in proportion to their net (last paisa fixed so parts add up exactly).

**Per document**
```
taxable   = Σ line net (after discounts)
VAT       = Σ round2(line taxable × 13%)     (taxable lines only; exempt/zero-rated = 0)
before    = taxable + VAT + other charges
round-off = round(before) − before            (if round-off is on)
TOTAL     = before + round-off
```
- VAT only if the business is **VAT registered**. A PAN-only business bills with no VAT, and the VAT field is disabled.
- VAT rate comes from the dated tax-rate table (13% today), not from code.
- **TDS** = taxable × rate of the chosen nature (service, rent, contract…). Rates are seeded but flagged **"needs verification"**; the owner confirms them in Settings → Tax rates.
- **Amount in words** is English and Nepali (lakh / crore style). Amounts show Nepali grouping: `12,34,567.50`.

**Dates**: stored as AD, shown as **BS** (Bikram Sambat). Calendar data covers up to BS 2084.
**Numbers**: `INV-0001`, `BILL-0001`… one series per document type per fiscal year; prefixes are editable.

---

## 6. Balances and reports (all derived from the journal)

| Figure | How it is worked out |
|---|---|
| **Party balance** | Σ(debit − credit) on Receivables/Payables lines carrying that party. `+` = they owe us, `−` = we owe them. Unallocated payment = advance |
| **Document paid / outstanding** | Σ payment allocations (+ credit notes applied); `outstanding = total − paid − credited` |
| **Receivables / Payables** | Σ outstanding of open invoices / bills + expenses on credit |
| **Ageing** | Outstanding by days past due: Not due, 1–30, 31–60, 61–90, 90+ |
| **Sales growth** | Sales of the period vs the previous period of the same length, `(now − before) ÷ before` |
| **Gross profit** | Income − Cost of sales |
| **Net profit** | Gross profit − Operating expenses (year-end closing entries left out) |
| **Margin %** | Profit ÷ Income |
| **Cash flow** (direct method) | Opening cash/bank + money in − money out, grouped by cause (receipts, payments, expenses, TDS deposit, transfers) |
| **Expenses by category** | Expense account totals in the period, top 6 + "Other" |
| **Balance sheet** | Assets, Liabilities, Equity as of a date. Equity includes profit not yet closed. Check: **Assets = Liabilities + Equity** |
| **Account balance** | Σ(debit − credit) of that account (sign flipped for liabilities/equity/income) |
| **Stock value** | Weighted-average cost: each purchase updates the average; sales go out at that average |
| **VAT payable** | Output VAT (sales − credit notes) − Input VAT (purchases − debit notes) |

The **dashboard** shows: KPIs with growth, P&L, cash flow, sales trend, expenses by category, receivables and payables ageing, balance sheet with account balances, top customers, recent documents, low stock. Each card links to its full report. Roles without report access see only the basic KPIs.

---

## 7. Nepal specifics

- **Fiscal year**: Shrawan 1 to Ashadh end. 12 BS-month periods; each month can be locked.
- **VAT 13%**, exempt, zero-rated. Tax invoice shows PAN and VAT numbers of seller and buyer.
- **VAT registers**: sales book and purchase book (credit/debit notes negative), monthly return summary.
- **TDS**: withheld on payments (posted to TDS Payable) and on receipts (TDS Receivable); deposits tracked until paid to IRD.
- **Payment modes**: cash, bank, cheque, eSewa, Khalti, Fonepay.
- **Cheques**: Pending → Deposited → Cleared / Bounced / Cancelled.
- **CBMS** (IRD billing): queue, retry and log are built; the live IRD gateway is **off** until the IRD specification is wired in.
- **Bill print**: browser print / save as PDF in 6 layouts (Classic IRD, Modern, Minimal, Compact A5, Thermal 80mm, Abbreviated).

---

## 8. Safety nets

| Control | What it does |
|---|---|
| Period lock | Locked month / closed year rejects every write |
| Year-end close | Zeroes income and expenses into Retained Earnings, locks the year, opens the next. No reopen; corrections go in the open year |
| Cancel | Posts a reversal, keeps the original visible |
| Audit log | Who did what, with before/after, for every change |
| Books check | Every 6 hours the worker re-checks that books balance and VAT matches the VAT accounts |
| Recurring | A schedule copies a template document on its due date and posts it (claimed once, so never twice) |
| Share link | Customer view of an invoice by random token; expires, can be revoked; stops working if the workspace is suspended |

---

## 9. Not covered yet

Fixed-asset register and depreciation schedule (post depreciation by manual voucher), payroll / SSF, income-tax return computation, live CBMS, inventory beyond weighted-average stock (batches, warehouses), multi-currency.
