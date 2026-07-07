# SCM Assistant — Supply Chain RAG Chatbot

A Retrieval-Augmented Generation (RAG) chatbot built with **Flowise**, answering questions about a 
supplier network using purchase order data and governance policy documents.

## 🔗 Public Chatbot URL

https://cloud.flowiseai.com/chatbot/17c5f16c-9801-4886-931e-b2048dbc144f

## 🧱 Stack Used

| Component        | Choice                                      |
|-------------------|----------------------------------------------|
| Platform          | Flowise Cloud (flowiseai.com)                |
| LLM               | Google Gemini 2.5 Flash                      |
| Embeddings Model  | sentence-transformers/all-MiniLM-L6-v2       |
| Vector Store      | Pinecone                                     |
| Chain             | Conversational Retrieval QA Chain            |
| Memory            | Buffer Memory                                |
| Retriever Top K   | 75                                           |

## 📄 Data Sources

- `supplier_performance_data.csv` — 2,000 purchase orders, 116 suppliers, 27 columns
- `SupplyChain_Governance_Policy_v3.2.pdf` — 10-section supplier governance policy

## ✂️ Chunk Configurations Tried

| Configuration | Splitter            | Chunk Size | Overlap | CSV Chunks | CSV Chars | PDF Chunks | PDF Chars |
|---------------|----------------------|------------|---------|------------|-----------|------------|-----------|
| Config 1      | Recursive Character* | 1000       | 200     | 2,000      | 1,256,961 | 19         | 15,018    |
| Config 2      | Recursive Character  | 500        | 50      | 2,000      | 1,256,961 | 35         | 13,758    |

\*CSV loader splitter was effectively "None" — each PO row is treated as a single chunk regardless of 
chunk size/overlap settings, which is why CSV chunk/char counts are identical across both configurations.

**Notes:** Only the PDF was affected by the chunk size change, since the CSV loader splits by row rather 
than by the Recursive Character Text Splitter. Reducing chunk size from 1000→500 with a smaller overlap 
(200→50) increased the PDF's chunk count from 19 to 35, producing smaller, more granular chunks. Config 1 
was used for the final deployed chatbot, as it retained better policy-section coherence within each chunk.

## ⚙️ Retrieval Tuning

The default retriever Top K (4) was far too low for this dataset — most of the sample questions require 
aggregating or listing suppliers across the *entire* 2,000-row dataset (e.g., "which suppliers qualify," 
"which region has the highest total"), not just a few semantically similar chunks. With Top K = 4, the bot 
could only see a handful of records and gave incomplete or partially hallucinated answers.

Top K was increased to **75**, which noticeably improved recall, though it is still a subset of the full 
2,000-row dataset rather than the complete data. Note: pushing Top K much higher (e.g. 500+) caused Gemini 
API free-tier quota errors, so 75 was chosen as a practical balance between coverage and staying within 
rate limits.

## ❓ Sample Questions & Answers (Verbatim)

**Q1: Which Tier-3 suppliers have an active disruption flag, and what response level applies per policy?**

> Here are the Tier-3 suppliers with an active disruption flag and their corresponding response levels:
>
> - **SUP-080 (Maghreb Castworks):** Has "Regulatory enforcement pending" as an active disruption and is classified as "High Risk". "Regulatory enforcement actions" require a mandatory Level 3 response regardless of risk level. **Response Level: Level 3**
> - **SUP-010 (Bohai Electronics):** Has multiple active disruptions including "Regulatory enforcement pending" and "Typhoon season disruption", and is classified as "High Risk" for some POs. "Regulatory enforcement actions" require a mandatory Level 3 response regardless of risk level. **Response Level: Level 3**
> - **SUP-022 (DaNang Metal Works):** Has "Regulatory enforcement pending" as an active disruption and is classified as "High Risk". "Regulatory enforcement actions" require a mandatory Level 3 response regardless of risk level. **Response Level: Level 3**
> - **SUP-041 (Archipelago PCB Corp):** Has "Labour strike risk" and "Typhoon season disruption" as active disruptions and is classified as "High Risk". "Active labour strikes" require a mandatory Level 3 response regardless of risk level. **Response Level: Level 3**
> - **SUP-079 (Helios Pack Greece):** Has "Regulatory enforcement pending" as an active disruption and is classified as "High Risk". "Regulatory enforcement actions" require a mandatory Level 3 response regardless of risk level. **Response Level: Level 3**

---

**Q2: Which suppliers qualify for the annual Volume Rebate Program and how many are there?**

> The following suppliers qualify for the annual Volume Rebate Program:
> - Cascade Pack Corp (SUP-107)
> - Rheinwerk Precision AG (SUP-047)
> - Crestline Chemical Supply (SUP-102)
>
> There are 3 suppliers that qualify.

---

**Q3: Which region has the highest total PO value, and does it breach the concentration limit?**

> The region with the highest total PO value is APAC, with a total of $14,940,328.09.
>
> Yes, this breaches the concentration limit. Per policy, a single region's total PO value should not exceed $10,000,000 USD to avoid concentration risk.

---

**Q4: Which suppliers are on Supplier Watch List (SWL) status and what does it restrict?**

> Here are the suppliers currently on the Supplier Watch List (SWL) status:
> - SUP-080 (Maghreb Castworks): Has a Compliance Score of 58 (PO-10776).
> - SUP-042 (Visayas Textile Co): Has a Compliance Score of 50 (PO-11898).
>
> Being on SWL status restricts new Purchase Order (PO) issuance to 20% of the prior quarter's volume.

---

**Q5: Which product category has the highest average defect rate and does it exceed the Tier-2 limit?**

> The product category with the highest average defect rate is Mechanical Components, with an average defect rate of approximately 1.30%.
>
> This average defect rate does not exceed the Tier-2 maximum permissible defect rate, which is 2.50%.

## 🛠️ What I'd Improve

- **Retrieval coverage:** Even at Top K = 75, the bot is only sampling a subset of the 2,000 PO records, so 
  answers to aggregation-style questions (counts, sums, averages, "list all X") are approximations rather 
  than exhaustive results computed over the full dataset. A more robust design would pair the vector store 
  with a structured query layer (e.g., loading the CSV into a SQL-queryable store) so counting/aggregation 
  questions are answered via exact queries, while the PDF policy content stays in vector-based RAG.
- **CSV chunking strategy:** Since the CSV loader splits by row regardless of chunk size settings, exploring 
  a different loader/splitter combination (or pre-aggregating rows before embedding) would allow chunk size 
  experiments to meaningfully affect CSV retrieval too, not just the PDF.
- **Pre-aggregation:** Pre-computing per-supplier summary rows (aggregating each supplier's POs into one 
  record with average OTD, average defect rate, total PO value, etc.) before embedding would let the LLM 
  answer supplier-level questions more accurately without needing to reason over hundreds of raw rows at once.
- **API quota management:** Gemini's free-tier rate limits constrained how high Top K could be pushed. In a 
  production setting, a paid tier or a self-hosted embedding/inference setup would remove this ceiling.
- **Evaluation automation:** Given more time, I'd build a small automated eval script comparing chatbot 
  answers against expected answers (e.g., using string/entity matching) to iterate on Top K and chunking 
  strategy more systematically rather than manually testing each change.