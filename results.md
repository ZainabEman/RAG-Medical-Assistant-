# 📊 Evaluation Results — MED-RAG: Medical Transcript Assistant

> **Evaluation Date:** June 2026  
> **Total Queries:** 33  
> **LLM:** Gemini 2.5 Flash (temperature=0.3)  
> **Retriever:** ChromaDB (top-K=3)  
> **Throttle:** 5 seconds between queries  

---

## 1. Results Summary

| Metric | Value |
|---|---|
| **Total Queries Evaluated** | 33 |
| **Successful Responses** | 33/33 (100%) |
| **Errors / Failures** | 0 |
| **Citation Compliance** | ✅ All in-scope answers included `[Source: Row X]` citations |
| **Out-of-Scope Rejection Rate** | ✅ 5/5 (100%) — correctly refused non-medical queries |

---

## 2. Category Breakdown

### 2.1 Procedure-Specific Queries (10/10 ✅)

These queries test the system's ability to retrieve detailed surgical and procedural information.

| # | Query | Status | Notes |
|---|---|---|---|
| 1 | Which technique was used for laparoscopic cholecystectomy in the dataset? | ✅ | Retrieved relevant surgical technique descriptions with citations |
| 2 | Describe the incision type documented for carpal tunnel release. | ✅ | Correctly identified incision types from orthopedic transcriptions |
| 3 | What type of anesthesia is most commonly mentioned for colonoscopy procedures? | ✅ | Retrieved anesthesia-related documentation |
| 4 | Summarize the MRI lumbar spine findings in any available transcription. | ✅ | Pulled relevant radiology report content |
| 5 | How are patients positioned for total knee arthroplasty according to the notes? | ✅ | Retrieved positioning details from operative notes |
| 6 | Which sutures are used to close fascia in hernia repair operations? | ✅ | Identified specific suture types from surgical notes |
| 7 | Summarize the typical intraoperative findings during cystoscopy. | ✅ | Retrieved urology procedure findings |
| 8 | What complications, if any, are reported during cataract surgeries? | ✅ | Correctly retrieved ophthalmology complication data |
| 9 | What is the estimated blood loss reported for lumbar fusion cases? | ✅ | Retrieved EBL data from orthopedic/neurosurgery notes |
| 10 | Which specific medication is injected during epidural steroid injections? | ✅ | Identified specific medications from pain management transcriptions |

**Strengths:** The system excels at retrieving specific procedural details — surgical techniques, medications, anatomical descriptions — due to the rich vocabulary in operative notes aligning well with the embedding model's semantic understanding.

---

### 2.2 Symptoms / History Queries (10/10 ✅)

These queries test retrieval of patient symptoms, physical exam findings, and clinical history.

| # | Query | Status | Notes |
|---|---|---|---|
| 11 | What presenting symptoms are associated with allergic rhinitis in the dataset? | ✅ | Retrieved ENT symptom descriptions |
| 12 | Describe the documented history of COPD patients in these transcriptions. | ✅ | Pulled pulmonology patient history |
| 13 | Which symptoms are reported for acute appendicitis cases? | ✅ | Retrieved classic appendicitis presentations |
| 14 | What vital signs are recorded for patients presenting with chest pain? | ✅ | Identified vital sign documentation patterns |
| 15 | What physical exam findings appear in otitis media descriptions? | ✅ | Retrieved ENT examination findings |
| 16 | What previous surgeries are mentioned for patients with hip fractures? | ✅ | Pulled surgical history from orthopedic notes |
| 17 | Which allergies are listed for patients with skin abscesses? | ✅ | Retrieved allergy documentation from dermatology records |
| 18 | Is any family history of cardiovascular disease reported in the notes? | ✅ | Found family history sections in consultation notes |
| 19 | What neurological symptoms are mentioned for patients with headaches? | ✅ | Retrieved neurology symptom documentation |
| 20 | Summarize the social history for any patient evaluated for depression. | ✅ | Pulled psychiatry social history data |

**Strengths:** The system handles both specific (e.g., "vital signs for chest pain") and broader (e.g., "history of COPD patients") symptom queries effectively. The chunking strategy preserves enough context for meaningful retrieval.

---

### 2.3 Management / Plan Queries (5/5 ✅)

These queries test retrieval of treatment plans, medications, and follow-up instructions.

| # | Query | Status | Notes |
|---|---|---|---|
| 21 | What postoperative plan is described for tonsillectomy patients? | ✅ | Retrieved post-op care documentation |
| 22 | Which medications are prescribed for hypertension within the dataset? | ✅ | Identified hypertension medications from multiple sources |
| 23 | What type of physical therapy is recommended after rotator cuff repair? | ✅ | Retrieved rehabilitation protocols |
| 24 | What diet recommendations are given to patients with diabetes mellitus? | ✅ | Pulled dietary guidance from endocrinology records |
| 25 | What follow-up or wound care instructions are documented? | ✅ | Retrieved wound care and follow-up instructions |

**Strengths:** Treatment and management queries perform well because plan/assessment sections in medical transcriptions tend to be clearly structured and distinct from other sections.

---

### 2.4 Out-of-Scope Queries (5/5 ✅ — Correctly Rejected)

These queries test the system's ability to refuse non-medical questions and avoid hallucination.

| # | Query | Expected Behavior | Actual Result |
|---|---|---|---|
| 26 | Who invented the stethoscope? | ❌ Refuse (general knowledge) | ✅ Correctly stated it cannot answer from the medical context |
| 27 | What is the capital city of France? | ❌ Refuse (geography) | ✅ Correctly refused — not in clinical dataset |
| 28 | What is the current stock price of Pfizer? | ❌ Refuse (financial data) | ✅ Correctly refused — no financial data in context |
| 29 | Summarize the storyline of the movie 'The Doctor'. | ❌ Refuse (entertainment) | ✅ Correctly refused — not in medical transcriptions |
| 30 | How do I bake a basic vanilla cake? | ❌ Refuse (cooking) | ✅ Correctly refused — no cooking instructions in dataset |

**Key Finding:** The citation-enforcing system prompt is highly effective at preventing hallucination. By requiring the LLM to ground every statement in `[Source: Row X]` citations and explicitly instructing it to refuse when context is insufficient, the system achieves **100% out-of-scope rejection** — a critical property for medical AI applications.

---

### 2.5 High-Level Dataset Queries (3/3 ✅)

These queries test the system's ability to aggregate or compare information across multiple documents.

| # | Query | Status | Notes |
|---|---|---|---|
| 31 | Compare the anesthesia types used across orthopedic surgeries in the dataset. | ✅ | Aggregated anesthesia information from multiple orthopedic records |
| 32 | List common cardiovascular risk factors mentioned across different patients. | ✅ | Compiled risk factors from cardiology and general medicine records |
| 33 | What appear to be the most frequent postoperative diagnoses in these transcriptions? | ✅ | Summarized common post-op diagnoses based on retrieved context |

**Limitations:** With K=3 retrieval, high-level aggregation queries only see 3 documents, limiting the breadth of cross-document analysis. These answers are based on a small sample rather than a true dataset-wide aggregation.

---

## 3. Qualitative Analysis

### 3.1 Retrieval Quality

| Aspect | Assessment |
|---|---|
| **Semantic Relevance** | 🟢 Strong — retrieved documents consistently match query intent |
| **Specialty Alignment** | 🟢 Strong — orthopedic queries retrieve orthopedic docs, etc. |
| **Snippet Informativeness** | 🟢 Good — 1000-char chunks provide sufficient context |
| **Edge Cases** | 🟡 Moderate — very broad queries may retrieve tangentially related docs |

### 3.2 Answer Quality

| Aspect | Assessment |
|---|---|
| **Citation Compliance** | 🟢 Excellent — consistent `[Source: Row X]` format |
| **Factual Grounding** | 🟢 Strong — answers clearly derived from retrieved context |
| **Clinical Coherence** | 🟢 Good — well-structured, medically appropriate language |
| **Refusal Behavior** | 🟢 Excellent — clean, explicit refusal for out-of-scope queries |
| **Verbosity** | 🟡 Moderate — some answers could be more concise |

### 3.3 System Performance

| Aspect | Value |
|---|---|
| **Avg. Response Time** | ~3–8 seconds per query (including retrieval + generation) |
| **API Throttle** | 5 seconds between batch evaluation queries |
| **Total Evaluation Time** | ~5–8 minutes for 33 queries |
| **Error Rate** | 0% (no API failures or exceptions) |

---

## 4. Model Strengths

1. **Citation Enforcement**: The prompt engineering ensures every claim is traceable to a specific source document, critical for medical applications.

2. **Out-of-Scope Rejection**: 100% rejection rate for non-medical queries demonstrates strong hallucination prevention.

3. **Cross-Specialty Coverage**: The system handles queries across diverse medical specialties (orthopedic, cardiology, neurology, dermatology, etc.) with consistent quality.

4. **Persistent Vector Store**: ChromaDB persistence eliminates redundant embedding computation, reducing API costs and startup time on subsequent runs.

5. **Structured Evidence Display**: The dual-panel output (answer + evidence) allows users to verify claims against raw source material.

---

## 5. Model Weaknesses

1. **Limited Aggregation Depth**: With K=3 retrieval, dataset-wide summarization queries are limited to information from only 3 documents.

2. **No Quantitative Evaluation Metrics**: Lacks automated scoring (RAGAS faithfulness, answer relevancy, context precision) — evaluation is qualitative only.

3. **Single Retrieval Strategy**: No hybrid search (dense + sparse) or reranking, which could improve precision for keyword-heavy medical queries.

4. **Subset Limitation**: Default 1,500-row subset may miss relevant records in the remaining 3,499 rows.

5. **No Conversation Memory**: Each query is independent; follow-up questions cannot reference previous answers.

---

## 6. Comparison with Baseline

| Approach | Pros | Cons |
|---|---|---|
| **Keyword Search (baseline)** | Fast, simple | Misses semantic relationships, no NLU |
| **LLM-only (no RAG)** | Understands semantics | Hallucination risk, no source grounding |
| **MED-RAG (this system)** | Semantic search + grounded generation + citations | API-dependent, limited K retrieval |

---

## 7. Recommendations for Production

1. **Scale to full dataset** (4,999 records) for comprehensive coverage
2. **Add RAGAS evaluation** for automated quality metrics
3. **Implement reranking** with a cross-encoder model
4. **Enable streaming** for better UX on long answers
5. **Add user feedback collection** to continuously improve retrieval quality
6. **Deploy on Hugging Face Spaces** for persistent, shareable hosting

---

## 8. Conclusion

The MED-RAG system demonstrates strong performance across all evaluation categories, achieving **100% query completion**, **100% citation compliance** for in-scope queries, and **100% out-of-scope rejection**. The combination of Google's embedding model, ChromaDB vector search, and Gemini 2.5 Flash with citation-enforcing prompts creates a reliable, traceable medical information retrieval system.

While the system lacks formal quantitative evaluation metrics (a key area for future work), the qualitative assessment across 33 diverse queries confirms its viability as a foundation for clinical knowledge retrieval applications.
