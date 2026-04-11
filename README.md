# n8n Automation Workflows

A collection of production automation workflows built with n8n during my internship at a Barcelona-based startup. The work focused on receipt validation, AI-powered data extraction, and backend pipeline design.

This repo contains workflow exports and setup notes. Internal business logic, pipeline diagrams, and company-specific documentation are not published here.

---

## What these workflows do

Each workflow handles a different part of the data pipeline:

- Webhook-triggered processing from Supabase database events
- Image analysis using Gemini Vision API (receipt reading, product identification)
- Multi-source API lookups (OpenFoodFacts, UPCItemDB)
- Structured output logging to Google Sheets
- Validation logic with rejection codes and user-facing messages in Spanish

The pipelines run in near-real-time and were tested against live user submissions during the internship.

---

## Tech stack

- n8n (self-hosted workflow automation)
- Google Gemini 2.5 Flash (vision-based data extraction)
- Supabase (PostgreSQL database + webhook triggers)
- Google Sheets (output logging)
- OpenFoodFacts API + UPCItemDB API (product data lookup)
- Python (data normalization and preprocessing scripts)

---

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- Google Gemini API key
- Supabase project with webhook configuration
- Google Sheets with the required output schema

### Import a workflow

1. Open n8n and go to Workflows → Import
2. Upload the relevant `.json` file from this repo
3. Replace placeholder values (API keys, sheet IDs, webhook URLs) with your own
4. Connect your credentials in n8n
5. Activate the workflow

Refer to the inline node descriptions inside each workflow JSON for field-level documentation.

---

## Notes

These workflows were built for a specific production environment. If you are adapting them for a different project, you will need to adjust the Supabase schema, webhook filters, and output column structure to match your setup.

The Gemini prompts inside each HTTP node are tuned for Spanish-language receipts and supermarket data. They may need adjustment for other languages or retail formats.

---

## Related projects

- [BookFinder NLP](https://github.com/Saiaditya26122006/NLP_Recommender_project) — TF-IDF + Sentence-BERT recommender using Supabase as backend
- [Smart Camera](https://github.com/Saiaditya26122006/Smart_Camera) — Flask + Gemini Vision API deployed on AWS Elastic Beanstalk
- [Student Club Hub](https://github.com/Saiaditya26122006/Student-Club-Hub) — Full-stack platform with PostgreSQL, Flask, React, and Docker
