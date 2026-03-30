# Ostras — Ticket Validation Workflow (V1)

Automated receipt validation pipeline built in N8N for the Ostras app. Users submit supermarket receipt photos and get points awarded based on the purchase. Supports near-expiry product submissions for bonus points.

---

## What it does

Every time a user submits a ticket in the app, Supabase fires a webhook to N8N. The workflow validates the submission, extracts data using Gemini AI, applies business rules, and logs the result to Google Sheets.

**Flow 1 — Standard receipt (submission_type = 1)**
- User uploads a receipt photo
- Gemini reads the receipt and extracts store name, date, items, total
- Validates: supermarket only, receipt within 24h of purchase, Spain only
- Result: 2 points if approved

**Flow 2 — Near-expiry product (submission_type = 2)**
- User uploads receipt + barcode photo + expiry date photo
- Gemini reads all three images
- Validates: same as Flow 1, plus product must be on the receipt and expiry within 5 days of purchase
- Result: 1–5 points based on days to expiry (same day = 5 pts, 4 days away = 1 pt)

---

## Pipeline

```
Webhook Trigger
  → Check Required Images
  → Images Valid?
      ✗ → Prepare Output Row → Write To Google Sheets
      ✓ → Gemini Receipt Extraction
            → Normalize And Parse Receipt
            → Extraction Error?
                ✓ → Prepare Output Row → Write To Google Sheets
                ✗ → Confidence OK?
                      ✗ → Reject - Low Confidence → Prepare Output Row
                      ✓ → Is Flow 2?
                            ✗ → Flow 1 Logic → Prepare Output Row
                            ✓ → Extract Barcode + Extract Expiry (parallel)
                                  → Merge Product Data
                                  → GTIN Lookup OpenFoodFacts
                                  → GTIN Lookup UPCItemDB
                                  → Flow 2 Logic → Prepare Output Row
```

---

## Setup

### 1. Prerequisites
- N8N instance (self-hosted or cloud)
- Google Gemini API key — [get one here](https://aistudio.google.com/app/apikey)
- Google Sheets with the output sheet set up
- Supabase project with a `tickets` table

### 2. Configure environment

```bash
cp .env.example .env
# Fill in your GEMINI_API_KEY and GOOGLE_SHEET_ID
```

### 3. Import the workflow

1. Open N8N
2. Go to **Workflows → Import**
3. Upload `ostras_v1_workflow.json`
4. In each Gemini HTTP node, replace `YOUR_GEMINI_API_KEY` in the URL with your actual key
5. In the `Write To Google Sheets` node, replace `YOUR_GOOGLE_SHEET_ID` with your sheet ID and connect your Google Sheets credential
6. Save and activate the workflow

### 4. Set up Google Sheets

Create a sheet with this exact header row (18 columns):

```
timestamp | ticket_id | user_id | firebase_user_id | submission_type | status | rejection_reason | rejection_message | points_awarded | flow | store_name | receipt_date | hours_since_purchase | days_to_expiry | matched_product | gemini_confidence | receipt_url | processing_stage
```

### 5. Configure Supabase webhook

In Supabase → Database → Webhooks:
- **Table:** `tickets`
- **Event:** `UPDATE` (important — not INSERT, so all image URLs are populated)
- **Filter:** `barcode_url IS NOT NULL` (for Flow 2) or configure per submission type
- **URL:** `https://your-n8n-instance/webhook/ostras-v1-webhook`

---

## Output columns

| Column | Description |
|--------|-------------|
| `timestamp` | When the workflow processed this ticket |
| `ticket_id` | Supabase ticket ID |
| `user_id` | Supabase user UUID |
| `firebase_user_id` | Firebase auth UID |
| `submission_type` | 1 = receipt only, 2 = near-expiry product |
| `status` | `approved` or `rejected` |
| `rejection_reason` | Code (e.g. `OUTSIDE_24H`, `INVALID_STORE`) |
| `rejection_message` | Spanish user-facing message |
| `points_awarded` | 0 if rejected, 2 for Flow 1, 1–5 for Flow 2 |
| `flow` | 1 or 2 |
| `store_name` | Store name extracted by Gemini |
| `receipt_date` | Date on the receipt (YYYY-MM-DD) |
| `hours_since_purchase` | Hours between receipt date and submission |
| `days_to_expiry` | Flow 2 only — days between purchase and expiry |
| `matched_product` | Flow 2 only — product name matched on receipt |
| `gemini_confidence` | Gemini's confidence score for the extraction |
| `receipt_url` | Supabase storage URL for the receipt image |
| `processing_stage` | Where in the pipeline the decision was made |

---

## Rejection codes

| Code | Meaning |
|------|---------|
| `MISSING_RECEIPT` | No receipt image uploaded |
| `MISSING_BARCODE` | Flow 2 — no barcode image |
| `MISSING_EXPIRY` | Flow 2 — no expiry date image |
| `RECEIPT_READ_ERROR` | Gemini could not read the receipt |
| `LOW_CONFIDENCE` | Gemini confidence below 75% |
| `INCOMPLETE_TICKET` | Missing store name, date, or total |
| `INVALID_STORE` | Not a supermarket (restaurant, pharmacy, etc.) |
| `OUTSIDE_24H` | Receipt submitted more than 24h after purchase |
| `FUTURE_DATE` | Receipt date is in the future |
| `ALREADY_EXPIRED` | Product was already expired at time of purchase |
| `EXPIRY_OUT_OF_RANGE` | Product expires 5+ days after purchase |
| `PRODUCT_NOT_ON_RECEIPT` | Barcode product not found on the receipt |

---

## Tech stack

- **N8N** — workflow automation
- **Google Gemini 2.5 Flash** — receipt and product image extraction
- **Supabase** — database and webhook trigger
- **Google Sheets** — output logging
- **OpenFoodFacts API** — GTIN/barcode product lookup
- **UPCItemDB API** — fallback barcode lookup

---

## Current status

Testing mode — outputs to Google Sheets only. No Supabase writes, no app updates, no WhatsApp notifications.

**Before going live:**
- [ ] Change Supabase webhook from INSERT to UPDATE
- [ ] Add fraud checks (image quality, screenshot detection, duplicate hash)
- [ ] Add velocity limiting
- [ ] Connect Supabase write nodes to update ticket status
- [ ] Add WhatsApp/notification integration
