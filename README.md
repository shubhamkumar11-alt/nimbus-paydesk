# Nimbus PayDesk

Nimbus PayDesk is an automated Accounts Payable (AP) invoice processing pipeline built primarily as an **n8n workflow**, with a small companion **FastAPI** service for programmatic invoice intake.

## What it does

An invoice arrives as an email attachment (PDF) → the pipeline extracts the invoice fields using an LLM → applies policy rules to decide the payment path → routes high-value / low-confidence / GST-invalid invoices to a human approver by email → logs everything to a ticket database → and updates payment status once a decision is made (or a wait period times out).

### Pipeline stages

1. **Ingestion** — a Gmail trigger watches an inbox for new invoice emails and downloads the PDF attachment.
2. **Extraction** (`2. Extractor - Parse Invoice Fields`) — an LLM (Groq) reads the invoice text and extracts structured fields: vendor, GSTIN, invoice number, PO number, invoice date, amount (INR), and a confidence score.
3. **Ticket logging** (`6. Ticket Log - Insert New Ticket`) — the extracted invoice is upserted into a SQLite `tickets` table.
4. **Duplicate check** (`5. Duplicate Check - Query Tickets`) — checks whether a matching ticket (same vendor, invoice date, and amount) already exists.
5. **Policy check** (`3. Policy Checker - Route by Amount/Confidence`) — rules-based routing:
   - Low LLM confidence → exception path
   - Invalid GSTIN format → exception path
   - Duplicate found → exception path
   - Amount over ₹50,000 → exception path (human approval required)
   - Otherwise → auto-approved, ready to pay
6. **Exception routing / human approval** (`4. Exception Router - Email Approval Request`) — for anything that needs a human decision, an approval email is sent with approve/reject links; the workflow then waits (`Wait for Human Stamp`, up to 2 days) for a webhook response.
7. **Status resolution** — depending on the outcome (approved / rejected / timed out with no response), the ticket's `payment_status` is set accordingly and the ticket log is updated.

### GSTIN validation

GSTIN format is validated with the standard 15-character pattern:
`^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z][1-9A-Z]Z[0-9A-Z]$`

## Repository structure

```
nimbus_paydesk/
├── app/                 # Companion FastAPI service (invoice intake webhook)
│   ├── main.py
│   ├── api/
│   ├── core/
│   └── services/
├── docs/                # Design documentation
│   └── Nimbus_PayDesk_Capstone_Design.docx
├── policy/              # Policy rule notes
│   └── nimbus_policy_rules.md
├── workflow/            # n8n workflow export (JSON)
│   └── Nimbus_PayDesk.json
├── requirements.txt
└── README.md
```

## Running the FastAPI service

```bash
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

The service exposes:
- `GET /` — health check
- `POST /webhooks/inbound` — accepts a raw invoice payload (`ticket_id`, `raw_text`) and initializes an invoice ticket

## Running the n8n workflow

1. Import `workflow/Nimbus_PayDesk.json` into an n8n instance.
2. Configure credentials: a Gmail account (for both the trigger and sending approval emails), an SQLite database connection, and a Groq (or compatible) LLM credential.
3. Activate the workflow.

## Known limitations / future work

- **Duplicate-check ordering**: the duplicate check currently runs *after* the new ticket is inserted (`6. Ticket Log - Insert New Ticket → 5. Duplicate Check`), rather than before. This means every incoming invoice is logged regardless of duplicate status; the duplicate flag is informational rather than preventative. A future revision should re-order this so duplicate detection gates the insert.
- **Duplicate detection is exact-match only**: it compares vendor name, invoice date, and amount as plain equality — it does not do fuzzy/semantic matching (e.g. via embeddings/RAG) across near-duplicate invoices (OCR variations, re-issued invoices, minor formatting differences in vendor names, etc.). A RAG-based semantic duplicate/PO-matching layer is a natural next step.
- Attachment handling depends on the Gmail Trigger's "Download Attachments" option being enabled with a consistent attachment field prefix.

## Project background

This project was built and hardened as part of a capstone submission: the original workflow had several reliability issues (ambiguous timeout vs. rejection status, a validation warning from a misconfigured expression field, and a routing rule vulnerable to non-numeric invoice amounts) which were identified and fixed, after which the workflow was activated for production use and verified end-to-end with real test invoices sent through Gmail.
