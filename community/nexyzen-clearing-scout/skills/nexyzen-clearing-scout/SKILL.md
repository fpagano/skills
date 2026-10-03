---
name: nexyzen-clearing-scout
description: >
  Find money frozen in unpaid invoices and settle it without bank transfers.
  Builds a clearing ledger from Qonto client and supplier invoices, detects
  bilateral set-off opportunities (counterparties that are both client and
  supplier), previews multilateral clearing cycles across the network, and
  drafts a legally grounded set-off proposal. Use when the user asks about
  frozen working capital, unpaid/overdue invoices, offsetting debts with a
  counterparty, invoice clearing, netting, or "compensazione" of receivables
  and payables. DISCLOSURE: by default everything stays local (read-only on
  Qonto, deterministic scripts). Only if the user explicitly asks and
  confirms the exact list, it can transmit open-invoice data (pseudonymized
  VAT numbers, invoice numbers, dates, amounts) to Nexyzen, a third-party
  clearing service run by Camera di Compensazione S.r.l., not by Qonto. The
  agent can never accept a compensation or make any legal declaration: that
  binding step is performed by the user themselves on Nexyzen's own page.
permissions:
  mcp:
    qonto: [change_supplier_invoice_status, get_organization, list_client_invoices, list_supplier_invoices, mark_client_invoice_as_paid]
  network: [webapp.cameracompensazione.it]
  env: [NEXYZEN_AFFILIATE_CODE, NEXYZEN_BASE_URL, NEXYZEN_TOKEN]
  tools: [Read, Bash]
---

# Nexyzen Clearing Scout

You help a business owner discover how much working capital is frozen in
unpaid invoices and how much of it can be settled **without a single bank
transfer**, by offsetting mutual debts (set-off / netting / compensazione).

## Core principle

**No AI in the matching.** Every number shown to the user comes from the
deterministic Python scripts in `scripts/` — same input, same output, fully
auditable. Your job is to fetch the data, run the scripts, and narrate the
results clearly. Never estimate or compute offsets yourself: run the scripts.
See `references/methodology.md` for definitions and how to read each report.

This skill never initiates payments, transfers, or any money movement. The
only write action it may perform on Qonto — marking an invoice as paid after
a completed set-off — requires explicit user confirmation first.

## What leaves Qonto, and what you must never do

Steps 1-4 are local: data is read from Qonto and processed by scripts on the
user's machine; nothing is sent anywhere.

There are exactly two points where a third party is involved, and both need
the user's explicit, informed yes:

1. **Submission (step 5)** transmits business and fiscal data to Nexyzen, an
   external service operated by Camera di Compensazione S.r.l. Before running
   it you MUST show the user the dry-run list (every invoice, counterparty
   pseudonym, amount) and say plainly that it goes to a third party that is
   not Qonto. Send only after the user confirms that list, and drop any
   invoice they exclude.
2. **Acceptance of a compensation (step 6)** is a binding legal act: a credit
   assignment with the assignor's warranties (art. 1266 c.c.). **You must
   never accept a compensation, confirm declarations, attest warranties, or
   fill in legal-representative data on the user's behalf.** This skill has no
   command for it on purpose. You present the proposal and the user decides,
   on their own, on the Nexyzen acceptance page.

Never pass `--send` on your own initiative, never reuse an earlier "yes" for a
different batch, and never treat the user's request to "check" or "look at"
their positions as consent to submit anything.

## Workflow

### 1. Ingest — fetch invoices from Qonto

Using the Qonto MCP tools:

1. `get_organization` — identity of the user's company (name, legal country).
2. `list_client_invoices` — receivables. Paginate until complete, and pass
   `exclude_imported: false`: invoices imported into Qonto (rather than
   created there) are excluded by default, but they are real receivables.
3. `list_supplier_invoices` — payables. Paginate until complete.

Save the two raw result sets as JSON files (e.g. `client_invoices.json`,
`supplier_invoices.json`) in a working directory, preserving the tool output
structure (`{"client_invoices": [...]}`, `{"supplier_invoices": [...]}`).

### 2. Normalize — build the clearing ledger

```
python scripts/build_ledger.py --clients client_invoices.json \
                               --suppliers supplier_invoices.json \
                               --out ledger.json
```

The script deduplicates counterparties by normalized VAT number (country
prefixes, spelling variants and OCR typos in names are handled), keeps only
open positions (paid/canceled invoices are dropped, explicit partial-payment
notes are honored), and computes totals plus receivables aging.

Present to the user:
- total frozen working capital (open receivables + open payables);
- net position;
- overdue receivables and the aging breakdown;
- counterparties that appear on **both sides** (clients AND suppliers) —
  point out any name variants the deduplication caught.

### 3. Detect — bilateral set-off opportunities

```
python scripts/detect_offsets.py --ledger ledger.json --out offsets.json
```

For each counterparty that is both client and supplier, the offsettable
amount is `min(open credit, open debit)`. This is immediately actionable
under statutory set-off rules (`references/legal_basis.md`): the two
companies simply agree in writing, and the matched amounts are settled.
Report each match with the invoice numbers involved and the residual
positions after the offset.

### 4. Preview — multilateral clearing cycles (optional)

Bilateral matching only scratches the surface: most frozen capital sits in
*chains* (A owes B, B owes C, C owes A) that no single company can see from
its own books. To show the potential:

```
python scripts/simulate_network.py --edges network_edges.json --out cycles.json
```

Build `network_edges.json` from the real Qonto positions (visible edges) plus
clearly marked `"simulated": true` edges for plausible third-party debts.
**Always tell the user which edges are simulated** — the script flags them in
its output. Real multilateral clearing requires the counterparties to join
the network: that is exactly what the Nexyzen clearing engine does
server-side.

### 5. Act — proposal and network submission

**Set-off proposal.** For each bilateral match the user wants to pursue, fill
`assets/compensation_proposal.md` (placeholders in `{{double_braces}}`) and
deliver the draft. It is a voluntary set-off agreement under art. 1252 of the
Italian Civil Code with equivalent references for FR/DE/ES.

**Submit to the clearing network (optional, explicit confirmation required).**
There is no button or menu for this anywhere — the only way it happens is a
user asking for it, or you proposing it and the user agreeing. This is the
moment business and fiscal data leaves the machine, toward a third party that
is not Qonto. Two steps, always in this order:

1. Run the preview (this is the default, it sends nothing):

```
python scripts/submit_to_nexyzen.py --ledger ledger.json \
    --org-vat <ORG_VAT> --email <user email>
```

2. Show the user the full list, invoice by invoice (number, counterparty
   pseudonym, amount), and state plainly: this data would be sent to Nexyzen,
   a service operated by Camera di Compensazione S.r.l.; VAT numbers are
   pseudonymized with SHA-256; no payment is initiated. Ask whether to send
   it, and which invoices to leave out. Only after an explicit yes, re-run
   the same command adding `--send` (and `--exclude-invoice <number>` for
   every invoice they dropped).

Sending requires `NEXYZEN_AFFILIATE_CODE` and `NEXYZEN_TOKEN` in the
environment; without them, or without `--send`, nothing is transmitted.

**Set expectations on timing.** Submitting does not return an instant
answer. Bilateral matches (step 3) are already visible from the user's own
data with no submission needed. Multilateral matches, though, only come from
the clearing engine's own processing cycle, which runs over the weekend —
results are ready **Monday morning**. Tell the user this plainly right after
submitting (e.g. "Sent. The network processes over the weekend, so check
back Monday morning — just ask me to look for compensation proposals
again."). Don't imply the network match is instant.

### 6. Close the loop — proposals found by the network (optional)

The clearing engine's own weekly cycle search may find matches that include
the user (see the submission note above for timing). Pull the results:

```
python scripts/check_compensations.py list --org-vat <ORG_VAT>
```

Present each proposal in plain terms: which receivable is assigned, which
debt gets settled, the amount, the legal basis.

**The agent stops here.** Accepting a compensation is a binding legal act:
the user assigns a receivable and makes warranties on it (art. 1266 c.c.),
and it cannot be undone. This skill therefore has no command to accept, and
you must not try to accept, confirm declarations or enter
legal-representative data in any other way. Tell the user that the proposal
is theirs to accept or ignore, and that they do it themselves on the Nexyzen
acceptance page linked in the notification they received, where the full
declarations and the company registry form are shown to them. Offer to help
them *understand* the proposal (amounts, counterparties, legal basis), never
to complete it for them.

When every participant of a cycle has accepted, the clearing house finalizes
it and delivers the credit-assignment letters to the Qonto channel instead of
PEC mail. Fetch them (read-only) with:

```
python scripts/check_compensations.py letters --org-vat <ORG_VAT> [--json]
```

Present each letter to the user (the `--json` output carries the full HTML
body) — it is the legal record of the settled set-off; suggest attaching it
to the corresponding invoices in Qonto.

**Mark as paid (only on explicit request).** After a set-off is accepted by
all parties — and only for invoices whose open amount is fully covered by
the cycle — use the Qonto MCP tools `mark_client_invoice_as_paid` /
`change_supplier_invoice_status` to record the settled invoices, one by one,
after the user confirms each. Partially covered invoices stay open: report
their new residual instead.

## Reporting guidelines

- Lead with the headline: how many EUR can be settled without a transfer.
- Amounts in the user's locale format, always with the invoice numbers.
- State the caveat verbatim where relevant: *multilateral clearing potential
  requires the counterparties to join the network.*
- If input data is messy (duplicate counterparties, OCR typos, partial
  payments), mention what the normalization did — it builds trust.
- No legal advice: the proposal template is a draft for the user's counsel.
