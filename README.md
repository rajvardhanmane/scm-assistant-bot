# SCM Assistant — Supply Chain RAG Chatbot (Flowise)

A RAG chatbot built in Flowise Cloud that answers questions about the BQBYTE Technologies supplier network, using two knowledge sources — a 2,000-row supplier performance CSV and the Supplier Governance & Compliance Policy v3.2 PDF — plus a pre-computed analytics summary to handle aggregate questions.

## Live Chatbot

**Public URL:** https://cloud.flowiseai.com/chatbot/ca13b60c-42ea-45f5-bc08-a41f1100f04c

## Stack

| Component | Choice | Why |
|---|---|---|
| Platform | Flowise Cloud (free plan) | Hosted, public share link out of the box |
| LLM | Google Gemini 2.5 Flash (free via Google AI Studio API key) | Fast, free tier, strong grounded QA |
| Embeddings | HuggingFace Inference — sentence-transformers/all-MiniLM-L6-v2 (384-dim) | Free, no daily request cap that blocks a 2k-chunk upsert |
| Vector store | Pinecone serverless (free tier), index `bqbyte-scm`, cosine | Managed, zero infra, dimension-matched to MiniLM |
| Chain | Conversational Retrieval QA Chain | Chat-history-aware retrieval QA with custom response prompt |

## Architecture

```
Document Store (3 loaders):
  supplier_performance_data.csv  -> 2,000 chunks (1 per PO row)
  SupplyChain_Governance_Policy_v3.2.pdf -> 17 chunks (Recursive splitter 1200/200)
  supplier_summary.md            -> 77 chunks (Markdown splitter 1200/150)
            |  embed: all-MiniLM-L6-v2 -> Pinecone (2,094 vectors)
            v
   Document Store (Vector) retriever
            |
ChatGoogleGenerativeAI (gemini-2.5-flash, temp 0.4)
            |
Conversational Retrieval QA Chain  ->  Public chat UI
```

A custom Response Prompt instructs the model to answer only from retrieved context, cite policy section numbers, and prefer the pre-computed analytics summary for totals, counts and averages.

## Why a third file (supplier_summary.md)?

Vector retrieval returns the top-k most similar chunks — it cannot sum 2,000 PO rows or average a column. Questions like "which region has the highest total PO value" are analytics questions, not lookup questions. So I pre-aggregated the CSV into a supplier-level and network-level summary (region spend totals, category defect averages, rebate eligibility, SWL lists, one profile per supplier) and upserted it alongside the raw files. This is a standard pattern for making RAG work over tabular data, and it is what makes Q2, Q3 and Q5 below answerable.

## Chunk Configurations Tried

| Config | PDF splitter | Summary splitter | PDF chunks | Summary chunks | CSV chunks | Result |
|---|---|---|---|---|---|---|
| A (baseline) | None (1 chunk/page) | None | 8 | 1 | 2,000 | Broken for the summary: the entire 69,835-char file embedded as a single vector, so the embedding only represented the first slice of the document and most of its content (rebate lists, SWL lists, 116 supplier profiles) was invisible to retrieval |
| B (final) | Recursive Character, size 1200 / overlap 200 | Markdown, size 1200 / overlap 150 | 17 | 77 | 2,000 | Policy tables retrieved as coherent chunks with correct section citations; each summary chunk holds a complete section or 1–2 supplier profiles |

Total upserted: **2,094 vectors** (verified in Pinecone: Added 2094, Skipped 0).

## Sample Q&A (verbatim chatbot output)

**Q1. Which Tier-3 suppliers have an active disruption flag, and what response level applies per policy?**

> The following Tier-3 suppliers have an active disruption flag, and the corresponding response level applies per Policy Section 9:
>
> * Level 3 'Activate' (for High Risk suppliers with disruption flag): Archipelago PCB Corp, Bogota Pack Ltda, Bohai Electronics, Cerromax Mineria, DaNang Metal Works, Dravex Components India, Helios Pack Greece, Lima Polymer SA, Sahyadri Alloy Tech, Vistula Pack Sp, Yucatan Polymer Mfg
> * Level 2 'Manage' (for Medium Risk suppliers with disruption flag): Deltaforge Vietnam, Longhua Polymer Works, Maghreb Castworks, MeKong Pack Co, Plataforma Metales SA, Quetzal Textiles, Sibertek Molding, Varna Electronics EAD, Visayas Textile Co
> * Level 1 'Monitor' (for Low Risk suppliers with disruption flag): Buenos Aires Pack, Deccan Polymer Mfg, Orinoco Pack SAPI, Patagonia Alloys SRL

**Q2. Which suppliers qualify for the annual Volume Rebate Program and how many are there?**

> Based on the average performance across all 8 quarters, no Tier-1 supplier meets all three thresholds simultaneously for a full calendar year, which is required for the annual Volume Rebate Program (Policy Section 4.2). Therefore, no suppliers qualify for the program.

**Q3. Which region has the highest total PO value, and does it breach the concentration limit?**

> The region with the highest total PO value is APAC, with a total PO value of $131,620,356.14, representing 37.0% of the total network spend. This does NOT breach the concentration limit, as Policy Section 5.3 states that no single region may account for more than 45% of total annual procurement spend.

**Q4. Which suppliers are on Supplier Watch List (SWL) status and what does it restrict?**

> The following 24 suppliers are on Supplier Watch List (SWL) status because they scored below 60 in at least one audit period (Policy Section 3.4): Archipelago PCB Corp, Bogota Pack Ltda, Bohai Electronics, Buenos Aires Pack, Cerromax Mineria, DaNang Metal Works, Deccan Polymer Mfg, Deltaforge Vietnam, Dravex Components India, Helios Pack Greece, Lima Polymer SA, Longhua Polymer Works, Maghreb Castworks, MeKong Pack Co, Orinoco Pack SAPI, Patagonia Alloys SRL, Plataforma Metales SA, Quetzal Textiles, Sahyadri Alloy Tech, Sibertek Molding, Varna Electronics EAD, Visayas Textile Co, Vistula Pack Sp, Yucatan Polymer Mfg.
> SWL status restricts new Purchase Order (PO) issuance to 20% of the prior quarter volume (Policy Section 3.4).

**Q5. Which product category has the highest average defect rate and does it exceed the Tier-2 limit?**

> The product category with the highest average defect rate is Packaging Materials at 1.91%. This average defect rate does not exceed the Tier-2 maximum permissible defect rate of 2.50% (Policy Section 3.2).

### Note on the task's expected answers

All five answers above were independently verified against the provided CSV with pandas. Some figures in the task sheet's answer key (e.g. EMEA at $193.9M / 48.5% of a $399.5M total; 19 rebate-eligible suppliers) do not reproduce from the provided dataset under any aggregation I tested — the provided CSV totals $356.0M with APAC as the largest region at 37.0%, and no Tier-1 supplier sustains all three rebate thresholds on full-period averages (25 meet them in at least one quarter). The chatbot reports what the provided data actually contains. One parsing detail worth noting: the "NA" (North America) region value is silently converted to a missing value by default CSV readers (e.g. pandas), which can drop the region from aggregates entirely — handled here with explicit NA-value settings.

## Challenges Faced

- **Gemini embedding quota:** the first upsert with `gemini-embedding-001` froze at exactly 100 vectors — the model's free-tier daily request cap — leaving the store stuck in UPSERTING. Fixed by migrating embeddings to HuggingFace MiniLM and rebuilding the Pinecone index at 384 dimensions (it was originally created at 3072 to match Gemini; Pinecone dimensions are immutable after creation).
- **Dimension-0 upsert error:** an initial `PineconeBadRequestError: Vector dimension 0` was caused by the embedding call failing silently with `TASK_TYPE_UNSPECIFIED`; setting the task type to `RETRIEVAL_DOCUMENT` resolved it.
- **Chat model quota:** `gemini-2.0-flash` returned 429 with `limit: 0` (no free-tier quota on that model anymore); switched the chat node to `gemini-2.5-flash`.

## What I'd Improve

- MiniLM truncates input at 256 tokens (~1,000 chars), so my 1,200-char chunks are embedded from their first portion only. I would reduce chunk size to ~900 chars or use a longer-context embedding model.
- Replace the pure-RAG chain with an Agentflow that has a code/SQL tool, so aggregate questions are computed live from the CSV instead of pre-computed.
- Add a re-ranker (e.g. Cohere Rerank) for better retrieval precision on policy clauses.
- Add metadata filtering by source file, so policy questions search only the PDF and data questions search only the CSV/summary.
- Evaluate with a larger gold-standard question set and track retrieval hit-rate per chunk config.

## Repo Contents

```
scm_assistant.json     # exported Flowise chatflow
supplier_summary.md    # pre-computed analytics knowledge file
README.md
.gitignore             # excludes .env and key files
/screenshots/          # build steps: store configs, upsert, retrieval test, canvas, public chat
```
