# VESSL — an ERP for a 10-person importer, built from scratch

**What it is:** the operations system [King Universal](https://kinguniversal.com) (a Raleigh, NC importer of licensed consumer products) runs on every day — quotes, sales and purchase orders, freight, customs cost, product testing and compliance, and a client portal. Live at `orders.vessl.io` since summer 2026. One developer: me.

**Why it exists:** the company ran on spreadsheets and email. Quotes lived in Excel, orders were re-typed three times, freight requests were copied by hand to each forwarder, and nobody could answer "what did we quote this client last time?" without a search through inboxes. VESSL replaced that, one workflow at a time, usually the week someone asked.

**Stack:** Next.js 14 (App Router) · Supabase (Postgres, Auth, Storage, RLS) · Vercel (hosting, cron) · Resend (transactional email) · Anthropic API (document extraction) · ExcelJS / print-to-PDF for documents.

**Code:** [`riley944/VESSL-erp`](https://github.com/riley944/VESSL-erp) — my commits are under `mdill97`. This page is a reader's guide to it.

---

## What it does

| Area | What the team can do |
|---|---|
| **Quotes** | Loren builds factory quotes with tiered pricing, per-size grids, HTS codes with duty rates, plate fees and carton specs; exports a client-facing price sheet (DDP only — freight and duty never shown to clients); imports a factory's PDF quotation straight into the form. |
| **Sales orders** | Kristy enters client POs (or imports the client's PDF), sizes break out one row per size, deposit terms print on the order confirmation PDF. |
| **Purchase orders** | Factory POs generated from quotes, with a read-only Production Board whose columns are the order statuses; SO and PO statuses stay in sync by trigger. |
| **Freight** | RFQs go to several contacts at each forwarder from one screen; bids come back by spreadsheet import or by hand; a landed-cost panel turns each bid into freight + MPF + HMF + duty + disbursement + insurance per unit; a Monday digest chases forwarders who haven't answered. |
| **Testing & compliance** | Test reports, materials, CPSC rules library linked to products, eFiling dates, a weekly email of products missing data. |
| **Everything else** | Companies and contacts, products catalogue, inventory, samples/PLM, analytics with exports, per-user access control, a client portal on a separate app. |

Everyone at the company uses it. Most features started as an email from one of them and were live within the week.

---

## Where to look in the code

The repo is a monolith with one big client-rendered page and a set of server routes — I say more about that below. These are the files I'd point a reviewer at first:

- **[`lib/landedCost.js`](https://github.com/riley944/VESSL-erp/blob/main/lib/landedCost.js)** — a pure function: lines + freight + rates → the customs cost breakdown. Rates (MPF with CBP's annual min/max, HMF, disbursement, insurance uplift) live in a settings table with CHECK constraints so Kristy can change them without a deploy. An "all-in" forwarder bid short-circuits the fees. This matched Kristy's spreadsheet to the cent on the first real bid; the one correction (goods value is the fee *base*, not part of the total) shipped the same afternoon she pointed it out.
- **[`lib/orderExtract.js`](https://github.com/riley944/VESSL-erp/blob/main/lib/orderExtract.js)** and **[`lib/pdfExtract.js`](https://github.com/riley944/VESSL-erp/blob/main/lib/pdfExtract.js)** — the PDF import pipeline. A client PO goes to a private storage bucket, then to Claude with a flat structured-output schema (flat because the grammar compiler has a size limit — learned the hard way). The model returns fields, lines one row per size, and *warnings tagged with the fields they're about*, so the form can colour a box amber. Client matching tries vendor number → exact name → same words in any order → buyer's email domain, and never auto-creates a company: a PDF's spelling of a name is not evidence. Three printout rows with per-size item codes collapse into one order line with S/M/L quantities.
- **[`app/api/orders/extract/route.js`](https://github.com/riley944/VESSL-erp/blob/main/app/api/orders/extract/route.js)** — the thin route over that helper; the quote variant shares it and ships behind an env flag so it could go to production hidden until its owner had tested it.
- **[`app/api/rfq/digest/route.js`](https://github.com/riley944/VESSL-erp/blob/main/app/api/rfq/digest/route.js)** — the Monday cron. The service role gets *no* table grants; it can execute exactly one `SECURITY DEFINER` function that returns the digest rows. Dry-run and only-me switches behind the secret.
- **[`lib/workbook.js`](https://github.com/riley944/VESSL-erp/blob/main/lib/workbook.js)** / **[`lib/clientSheet.js`](https://github.com/riley944/VESSL-erp/blob/main/lib/clientSheet.js)** — the Excel documents clients and forwarders receive, built cell by cell.
- **[`sql/README.md`](https://github.com/riley944/VESSL-erp/blob/main/sql/README.md)** — the index of the numbered database scripts and the convention behind them (next section).

---

## How I work on it

**Database changes are scripts, not clicks.** Every schema or data change is a numbered SQL file with a guard, a probe, before/after verification queries, and a rehearsal run that ends in `rollback;` before the real one ends in `commit;`. Most have a reverse script. A preflight checker catches the usual mistakes before the editor does. Supabase's MCP connection from the editor is read-only on purpose; I run the scripts by hand.

**Releases are deliberate.** Commits accumulate locally and are hand-tested on throwaway records (a `ZZTEST` client exists for exactly this) before anything is pushed. When one feature is ready and another isn't, I cherry-pick a release branch and push that to `main`, so unfinished work never reaches production. Every release has a named rollback target and I confirm the built SHA on Vercel before calling it live.

**Users first, then code.** Kristy, Loren, Riley and Monica tell me what's wrong with their day; I turn that into a scoped change, show it to them on localhost, and ship. The landed-cost calculator, the multi-contact RFQ, the PO import and the all-in bid flag all went from request to production in the same week in October.

**AI is a second pair of hands, not the brain.** I build with Claude Code daily: it writes most of the code against a spec I write, I read the diff, test by hand, and decide what ships. The things it can't do — know that a PO number should be filed the way the accountant files it rather than the way the PDF prints it, or that "quote date" means *created* date here — are the job. The PDF import is the same idea turned into a product feature.

**Security debt gets paid when found.** Replaced a browser-callable email edge function (an open relay) with a server route; found and fixed a table whose only RLS policy locked out everyone but one user; new buckets are private by default; API keys live only in Vercel and a gitignored `.env.local`.

---

## What I'd do differently

- **TypeScript and smaller files.** The app is plain JavaScript and the main page file is enormous. It was the fastest way to get a one-person system in front of users; it's now the biggest tax on changing it. The contract between the extraction route and the form that applies its result is exactly where end-to-end types would have caught drift.
- **Tests on the pure parts.** `landedCost` and the extraction shaping are pure functions with no test file. They're the first place I'd add one.
- **Backups and audit from day one.** The production database ran without backups for months; status history on orders only arrived after an accidental click showed we couldn't tell what a PO's status had been.

---

## By the numbers

- ~790 commits in the repo since June 2026, 565 under my own name (an early stretch went in under a shared workstation identity)
- 130+ numbered database scripts, each with its own verification
- 1 developer; every release with a named rollback target

---

## Screenshots

*(test data — ZZTEST client)*

<!-- quotes-form.png · sales-order-import.png · landed-cost.png · production-board.png -->

---

Matt Dillon · Raleigh, NC · [github.com/mdill97](https://github.com/mdill97)
