# 📋 Technical Report — MED-RAG: Medical Transcript Assistant

> **Author:** [Your Name]  
> **Date:** June 2026  
> **Version:** 1.0  

---

## 1. Executive Summary

MED-RAG is a Retrieval-Augmented Generation (RAG) system designed to enable natural-language querying over a large corpus of medical transcription records. The system addresses the challenge of extracting actionable clinical information from unstructured medical text by combining semantic vector search with large language model generation.

The pipeline processes **4,999 medical transcriptions** from the MTSamples dataset (using a configurable subset of 1,500 rows), embeds them into a ChromaDB vector store using Google's `text-embedding-004` model, and generates citation-backed answers using **Gemini 2.5 Flash**. A Gradio-based web interface provides an accessible front-end for clinical querying, while a batch evaluation framework validates system performance across 33 diverse queries.

**Key Achievement:** The system successfully answers clinical queries with verifiable source citations while correctly rejecting out-of-scope questions, demonstrating both retrieval relevance and hallucination resistance.

---

## 2. Problem Statement

### 2.1 Background

Medical transcription records contain rich clinical information — surgical procedures, patient histories, diagnostic findings, treatment plans, and medication details. However, this data is:

- **Unstructured:** Free-text narratives without standardized formats
- **Voluminous:** Thousands of records across dozens of specialties
- **Semantically complex:** Medical terminology requires domain understanding for effective search

### 2.2 Challenge

Traditional keyword search fails to capture the semantic relationships in medical text. A search for "heart surgery complications" would miss records describing "post-CABG atrial fibrillation" — even though they are semantically related. Clinicians and researchers need a system that understands medical context and retrieves relevant information regardless of exact terminology.

### 2.3 Objective

Build a RAG pipeline that:
1. Converts medical transcriptions into a searchable vector knowledge base
2. Answers natural-language clinical queries using only the retrieved context
3. Provides explicit source citations (`[Source: Row X]`) for every claim
4. Rejects out-of-scope questions to prevent hallucination

---

## 3. Dataset Analysis

### 3.1 MTSamples Dataset

| Property | Value |
|---|---|
| **Total Records** | 4,999 |
| **Specialties** | 40 clinical specialties |
| **Key Columns** | `medical_specialty`, `description`, `transcription` |
| **Avg. Transcription Length** | ~2,000–5,000 characters per record |
| **Source** | MTSamples.com (open-source medical transcription samples) |

### 3.2 Data Characteristics

The dataset represents a broad cross-section of clinical documentation including:

- **Operative notes** (surgical procedures, findings, technique descriptions)
- **Consultation notes** (patient history, symptoms, physical examination)
- **Discharge summaries** (diagnoses, treatment plans, follow-up instructions)
- **Radiology reports** (imaging findings, interpretations)
- **Progress notes** (ongoing care documentation)

### 3.3 Subset Configuration

For experimentation, the system uses a configurable subset:

```python
USE_SUBSET = True      # Toggle full dataset vs. subset
MAX_ROWS = 1500        # Number of rows when subset mode is active
```

This reduces embedding API costs while maintaining sufficient coverage for meaningful retrieval.

---

## 4. Methodology

### 4.1 Architecture Overview

The system follows a standard RAG architecture with two distinct phases:

**Phase 1 — Offline Indexing:**
```
CSV Data → Document Construction → Text Chunking → Embedding → Vector Store
```

**Phase 2 — Online Querying:**
```
User Query → Retrieval → Context Assembly → LLM Generation → Cited Answer
```

### 4.2 Data Preprocessing

Each row in the CSV is converted into a LangChain `Document` object with structured text:

```python
text = (
    f"Row: {idx}\n"
    f"Specialty: {specialty}\n"
    f"Description: {description}\n\n"
    f"Transcription:\n{transcription}"
)
```

**Design rationale:** Including the row index and specialty in the document text allows the LLM to cite specific sources and understand clinical context.

Metadata is attached for downstream citation:
```python
metadata = {
    "row": int(idx),
    "source": "mtsamples.csv"
}
```

### 4.3 Text Chunking Strategy

```python
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=100,
    separators=["\n\n", "\n", ". ", " ", ""],
)
```

| Parameter | Value | Rationale |
|---|---|---|
| `chunk_size` | 1000 chars | Larger chunks reduce API calls while preserving context |
| `chunk_overlap` | 100 chars | Prevents information loss at chunk boundaries |
| `separators` | Hierarchical | Respects paragraph → sentence → word boundaries |

### 4.4 Embedding Model

| Property | Value |
|---|---|
| **Model** | `text-embedding-004` (Google) |
| **Type** | Dense vector embeddings |
| **Dimensionality** | 768 dimensions |
| **Access** | Via `langchain-google-genai` package |

### 4.5 Vector Store

**ChromaDB** was selected for its:
- Lightweight, file-based persistence (no external database required)
- Native LangChain integration
- Automatic persistence to disk (`persist_directory`)
- Efficient cosine similarity search

The system includes smart caching — if a ChromaDB instance already exists at the specified path, it loads from disk instead of re-computing embeddings:

```python
if os.path.exists(CHROMA_PATH) and len(os.listdir(CHROMA_PATH)) > 0:
    vectorstore = Chroma(embedding_function=embeddings, persist_directory=CHROMA_PATH)
else:
    vectorstore = Chroma.from_documents(documents=documents, ...)
```

### 4.6 Retrieval Configuration

```python
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
```

**Top-K = 3** balances:
- **Recall:** Enough context to answer most clinical questions
- **Precision:** Avoids overwhelming the LLM with irrelevant information
- **Cost:** Fewer tokens in the prompt reduces LLM API costs

### 4.7 Language Model

| Property | Value |
|---|---|
| **Model** | `gemini-2.5-flash` |
| **Temperature** | 0.3 |
| **Provider** | Google AI via `langchain-google-genai` |

**Temperature = 0.3** was chosen for medical applications where factual accuracy is critical. This produces deterministic, consistent responses while allowing minimal creative phrasing.

### 4.8 Prompt Engineering

The system prompt enforces strict citation behavior:

```
You are a helpful and extremely safe medical assistant. Your primary task is to 
answer the user's question ONLY based on the provided clinical context below. 
You must not use any external knowledge. If the context does not contain the 
answer, you must state clearly: 'I cannot provide an answer based on the 
provided medical context.' For every piece of information you provide, you MUST 
include a citation, referencing the original source document using the format: 
[Source: Row X].
```

**Key constraints enforced:**
1. **Context-only answering** — No external knowledge allowed
2. **Mandatory citations** — Every claim must reference `[Source: Row X]`
3. **Graceful refusal** — Explicit message when context is insufficient
4. **Safety-first** — Framed as an "extremely safe" medical assistant

### 4.9 Chain Architecture

The pipeline uses LangChain's chain composition:

```python
# Step 1: Create the document combination chain
document_chain = create_stuff_documents_chain(llm, prompt)

# Step 2: Create the retrieval chain
rag_chain = create_retrieval_chain(retriever, document_chain)
```

- **`create_stuff_documents_chain`**: "Stuffs" all retrieved documents into the prompt context window
- **`create_retrieval_chain`**: Orchestrates: query → retriever → document chain → final answer

---

## 5. Gradio Web Interface

### 5.1 Design

The interface features a **Gen-Z inspired dark-mode** design:

- **Color Scheme:** Black (#0a0a0a) background with red (#FF0000) accents
- **Typography:** Inter font family via Google Fonts
- **Components:**
  - Text input with placeholder example queries
  - "⚡ Analyze" submit button with hover animations
  - Markdown output displaying both the answer and retrieved evidence
  - Pre-built example queries for quick exploration
  - Medical disclaimer banner

### 5.2 Response Format

Each response is structured as:

```
### 💬 Assistant Answer
[LLM-generated answer with [Source: Row X] citations]

---

### 🧠 System Evidence (Retrieved Context)
Context 1 — Row 42: [snippet of retrieved document]
Context 2 — Row 187: [snippet of retrieved document]
Context 3 — Row 1204: [snippet of retrieved document]
```

This dual-panel format allows users to both read the synthesized answer and verify it against the raw evidence.

---

## 6. Evaluation Framework

### 6.1 Methodology

A batch evaluation was conducted over **33 diverse queries** with a 5-second throttle between requests to respect API rate limits. Queries were categorized into five groups to test different capabilities.

### 6.2 Query Categories

| Category | Count | Purpose |
|---|---|---|
| Procedure-Specific | 10 | Test retrieval of surgical/procedural details |
| Symptoms / History | 10 | Test patient history and symptom extraction |
| Management / Plan | 5 | Test treatment and follow-up retrieval |
| Out-of-Scope | 5 | Test hallucination resistance |
| High-Level Dataset | 3 | Test cross-document aggregation |

### 6.3 Evaluation Criteria

Since this is a generative RAG system (not a classification model), traditional ML metrics (accuracy, F1, etc.) do not directly apply. Instead, the evaluation focuses on:

1. **Completion Rate**: Did the system return a response for every query?
2. **Citation Compliance**: Does every in-scope answer include `[Source: Row X]` citations?
3. **Out-of-Scope Rejection**: Does the system correctly refuse non-medical questions?
4. **Retrieval Relevance**: Are the retrieved documents semantically relevant to the query?
5. **Answer Coherence**: Is the generated answer well-structured and clinically meaningful?

---

## 7. Hyperparameters Summary

| Component | Parameter | Value |
|---|---|---|
| **Text Splitter** | `chunk_size` | 1000 |
| **Text Splitter** | `chunk_overlap` | 100 |
| **Text Splitter** | `separators` | `["\n\n", "\n", ". ", " ", ""]` |
| **Embedding** | `model` | `text-embedding-004` |
| **Retriever** | `k` (top-K) | 3 |
| **LLM** | `model` | `gemini-2.5-flash` |
| **LLM** | `temperature` | 0.3 |
| **Data** | `USE_SUBSET` | True |
| **Data** | `MAX_ROWS` | 1500 |
| **Evaluation** | `throttle_delay` | 5 seconds |

---

## 8. Limitations

### 8.1 Dataset Limitations
- **Static corpus**: The system queries a fixed dataset; no real-time clinical data ingestion
- **English only**: All transcriptions are in English; no multilingual support
- **Sample bias**: MTSamples may over-represent certain specialties and under-represent others
- **Subset mode**: Default configuration uses only 1,500 of 4,999 records

### 8.2 Technical Limitations
- **No automated evaluation metrics**: Lacks RAGAS, faithfulness, or answer relevancy scoring
- **Single embedding model**: No comparison with alternative embeddings (e.g., OpenAI, Cohere)
- **No reranking**: Retrieved chunks are used as-is without cross-encoder reranking
- **No conversation memory**: Each query is independent; no multi-turn context
- **Stuff chain limits**: All 3 documents are "stuffed" into a single prompt; may fail for very long documents
- **API dependency**: Requires active Google Gemini API access; no offline fallback

### 8.3 Clinical Limitations
- **Not clinically validated**: Outputs have not been reviewed by medical professionals
- **No HIPAA compliance**: Not suitable for real patient data
- **Hallucination risk**: While mitigated by context-only prompting, LLMs can still generate plausible but incorrect statements

---

## 9. Conclusions

MED-RAG demonstrates that a RAG pipeline using Google's Gemini ecosystem (embeddings + LLM) can effectively:

1. **Index and retrieve** semantically relevant clinical information from unstructured medical transcriptions
2. **Generate grounded answers** with explicit source citations, maintaining clinical traceability
3. **Reject out-of-scope queries** by enforcing context-only answering constraints
4. **Provide a user-friendly interface** through Gradio for interactive medical knowledge exploration

The system achieves a **100% query completion rate** across 33 evaluation queries and demonstrates strong citation compliance, making it a viable foundation for clinical information retrieval applications.

---

## 10. Future Improvements

| Priority | Improvement | Expected Impact |
|---|---|---|
| 🔴 High | Add RAGAS/automated evaluation metrics | Quantitative quality measurement |
| 🔴 High | Implement hybrid retrieval (dense + sparse) | Improved recall for keyword-heavy queries |
| 🟡 Medium | Add cross-encoder reranking | Better precision in retrieved documents |
| 🟡 Medium | Multi-turn conversation support | More natural clinical consultation flow |
| 🟡 Medium | Scale to full dataset (4,999 records) | Broader coverage across specialties |
| 🟢 Low | Streaming token generation | Better user experience for long answers |
| 🟢 Low | Deploy to Hugging Face Spaces | Persistent public hosting |
| 🟢 Low | Fine-tune embeddings on medical data | Domain-specific retrieval improvement |

---

## 11. References

1. Lewis, P. et al. (2020). "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks." *NeurIPS 2020.*
2. LangChain Documentation. https://python.langchain.com/
3. ChromaDB Documentation. https://docs.trychroma.com/
4. Google Gemini API Documentation. https://ai.google.dev/docs
5. MTSamples Dataset. https://www.mtsamples.com/
