<div align="center">

# 🩺 MED-RAG: Medical Transcript Assistant

### AI-Powered Clinical Knowledge Retrieval Using RAG with Gemini & ChromaDB

[![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![LangChain](https://img.shields.io/badge/LangChain-0.2+-1C3C3C?style=for-the-badge&logo=langchain&logoColor=white)](https://langchain.com)
[![Google Gemini](https://img.shields.io/badge/Gemini_2.5_Flash-4285F4?style=for-the-badge&logo=google&logoColor=white)](https://ai.google.dev/)
[![ChromaDB](https://img.shields.io/badge/ChromaDB-Vector_Store-FF6F00?style=for-the-badge)](https://www.trychroma.com/)
[![Gradio](https://img.shields.io/badge/Gradio-UI-F97316?style=for-the-badge&logo=gradio&logoColor=white)](https://gradio.app/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

*Ask questions about medical transcriptions. Get cited answers. See evidence.*

---

</div>

## 📋 Overview

**MED-RAG** is a Retrieval-Augmented Generation (RAG) system that enables natural-language querying over a corpus of **4,999 real medical transcription records** spanning **40 clinical specialties**. The system retrieves semantically relevant clinical context from a ChromaDB vector store and generates citation-backed answers using Google's **Gemini 2.5 Flash** LLM — ensuring every response is grounded in the source data with explicit `[Source: Row X]` citations.

The project features a **production-ready Gradio web interface** with a modern dark-themed UI, out-of-scope query rejection, and batch evaluation across 33 diverse clinical queries.

---

## 🎯 Problem Statement

Medical professionals and researchers frequently need to search through large volumes of unstructured clinical transcription data to find relevant information about procedures, symptoms, medications, and treatment plans. Traditional keyword-based search fails to capture semantic meaning, and manually reviewing thousands of transcriptions is prohibitively time-consuming.

**MED-RAG solves this by:**
- Converting unstructured medical transcriptions into a searchable vector knowledge base
- Enabling natural-language clinical queries with context-aware answers
- Enforcing source citation to maintain clinical traceability
- Rejecting out-of-scope questions to prevent hallucination on non-medical topics

---

## 📊 Dataset

| Property | Details |
|---|---|
| **Name** | MTSamples (Medical Transcription Samples) |
| **Source** | [MTSamples.com](https://www.mtsamples.com/) via [Kaggle](https://www.kaggle.com/datasets/tboyle10/medicaltranscriptions) |
| **Records** | 4,999 medical transcriptions |
| **Specialties** | 40 clinical specialties |
| **Key Columns** | `medical_specialty`, `description`, `transcription` |
| **Subset Used** | 1,500 rows (configurable via `USE_SUBSET` / `MAX_ROWS`) |

### Specialties Include:
Orthopedic, Cardiovascular, Gastroenterology, Neurology, Urology, General Medicine, Ophthalmology, Radiology, Psychiatry, Dermatology, ENT, Obstetrics/Gynecology, and more.

---

## 🔬 Methodology

### RAG Architecture

The system implements a **Retrieval-Augmented Generation** pipeline that decouples knowledge retrieval from language generation:

1. **Offline Indexing Phase**: Medical transcriptions are chunked, embedded, and stored in a persistent vector database.
2. **Online Query Phase**: User questions trigger semantic retrieval of the top-K most relevant chunks, which are injected as context into the LLM prompt.

### Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Embedding Model** | `text-embedding-004` | Google's latest embedding model with strong semantic understanding |
| **LLM** | `gemini-2.5-flash` | Fast inference, strong instruction following, cost-effective |
| **Vector Store** | ChromaDB (persistent) | Lightweight, local, supports persistence to avoid re-embedding |
| **Chunk Size** | 1,000 characters | Balances context richness with embedding API call efficiency |
| **Chunk Overlap** | 100 characters | Preserves cross-chunk semantic continuity |
| **Top-K Retrieval** | K=3 | Provides sufficient context without overwhelming the LLM |
| **Temperature** | 0.3 | Low temperature for factual, deterministic medical responses |

---

## ⚙️ Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                    OFFLINE INDEXING                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│   📄 MTSamples CSV (4,999 records)                      │
│          │                                               │
│          ▼                                               │
│   🔧 Data Loading (Pandas + LangChain Document)         │
│     • Combines: specialty + description + transcription  │
│     • Adds row-level metadata for citation tracking      │
│          │                                               │
│          ▼                                               │
│   ✂️ Text Chunking (RecursiveCharacterTextSplitter)      │
│     • chunk_size=1000, chunk_overlap=100                 │
│     • Separators: ["\n\n", "\n", ". ", " ", ""]          │
│          │                                               │
│          ▼                                               │
│   🧠 Vector Embedding (text-embedding-004)              │
│          │                                               │
│          ▼                                               │
│   💾 ChromaDB Persistent Vector Store                    │
│                                                          │
├─────────────────────────────────────────────────────────┤
│                    ONLINE QUERYING                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│   👤 User Query (natural language)                       │
│          │                                               │
│          ▼                                               │
│   🔍 Semantic Retrieval (Top-K=3 similarity search)     │
│          │                                               │
│          ▼                                               │
│   📝 Prompt Assembly (Citation-enforcing system prompt)  │
│     • Context injected from retrieved documents          │
│     • System prompt enforces [Source: Row X] citations   │
│          │                                               │
│          ▼                                               │
│   🤖 Gemini 2.5 Flash (temperature=0.3)                 │
│          │                                               │
│          ▼                                               │
│   ✅ Cited Medical Answer + Evidence Display             │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Pipeline Stage Details

| Stage | Component | Description |
|---|---|---|
| **Data Collection** | Pandas `read_csv` | Loads MTSamples CSV; supports subset mode for experimentation |
| **Data Preprocessing** | Custom Document Builder | Concatenates `medical_specialty`, `description`, and `transcription` into structured text per row |
| **Text Chunking** | `RecursiveCharacterTextSplitter` | Splits documents into 1000-character chunks with 100-character overlap using hierarchical separators |
| **Embedding** | `GoogleGenerativeAIEmbeddings` | Generates dense vector representations using Google's `text-embedding-004` model |
| **Vector Storage** | `Chroma.from_documents()` | Stores embeddings in a persistent ChromaDB instance for reuse across sessions |
| **Retrieval** | `vectorstore.as_retriever(k=3)` | Performs cosine similarity search returning the 3 most relevant chunks |
| **Generation** | `ChatGoogleGenerativeAI` | Gemini 2.5 Flash generates answers constrained to retrieved context with mandatory citations |
| **Interface** | Gradio Blocks | Dark-themed web UI with examples, markdown output, and disclaimer |

---

## 📈 Results

### Batch Evaluation Summary

The system was evaluated across **33 diverse queries** spanning five categories:

| Category | Queries | Description |
|---|---|---|
| **Procedure-Specific** | 10 | Surgical techniques, anesthesia, findings, complications |
| **Symptoms / History** | 10 | Presenting symptoms, exam findings, patient history |
| **Management / Plan** | 5 | Postoperative plans, medications, follow-up instructions |
| **Out-of-Scope** | 5 | General knowledge questions (stethoscope inventor, stock prices, etc.) |
| **High-Level Dataset** | 3 | Cross-patient aggregation and comparison queries |

### Key Findings

| Metric | Result |
|---|---|
| **Successful Responses** | 33/33 (100% completion rate) |
| **Citation Compliance** | All in-scope answers included `[Source: Row X]` citations |
| **Out-of-Scope Rejection** | System correctly declined to answer non-medical queries using context-only constraint |
| **Retrieval Relevance** | Top-3 retrieved chunks consistently matched the clinical domain of each query |

> For detailed per-query results and analysis, see [results.md](results.md).

---

## 🛠️ Technologies Used

### Programming Languages
| Language | Usage |
|---|---|
| Python 3.10+ | Core development language |

### Frameworks & Libraries
| Technology | Version | Purpose |
|---|---|---|
| LangChain | 0.2+ | RAG orchestration, chains, document processing |
| LangChain Community | 0.2+ | CSV loaders, ChromaDB integration |
| LangChain Classic | — | `create_stuff_documents_chain`, `create_retrieval_chain` |
| Google Generative AI | — | Gemini 2.5 Flash LLM + text-embedding-004 |
| ChromaDB | 0.4+ | Persistent vector store |
| Gradio | 4.0+ | Web interface |
| Pandas | 2.0+ | Data loading and manipulation |

### Tools & Platforms
| Tool | Purpose |
|---|---|
| Google Colab (T4 GPU) | Development and execution environment |
| Google AI Studio | API key management for Gemini |

---

## 🚀 Installation

### Prerequisites
- Python 3.10+
- Google Gemini API key ([Get one here](https://aistudio.google.com/apikey))

### Setup

```bash
# Clone the repository
git clone https://github.com/yourusername/rag-medical-assistant.git
cd rag-medical-assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### API Key Configuration

```bash
# Option 1: Environment variable
export GOOGLE_API_KEY="your-api-key-here"

# Option 2: The app will prompt you at runtime
```

---

## 💻 Usage

### Option 1: Run the Notebook (Recommended for First Time)

1. Upload `MedicalRag.ipynb` to [Google Colab](https://colab.research.google.com/)
2. Upload the `mtsamples.csv` dataset to `/content/`
3. Run all cells sequentially
4. Access the Gradio interface via the generated public URL

### Option 2: Run Locally

```bash
# Ensure your API key is set, then run the notebook
jupyter notebook MedicalRag.ipynb
```

### Example Queries

```
"What were the presenting symptoms in patients with acute appendicitis?"
"Which medications are prescribed for hypertension within the dataset?"
"What postoperative care instructions are commonly given after tonsillectomy?"
"Describe the incision type documented for carpal tunnel release."
```

---

## 📁 Project Structure

```
rag-medical-assistant/
│
├── README.md                 # Project overview, setup, and usage instructions
├── requirements.txt          # Python dependencies with version constraints
├── MedicalRag.ipynb          # Main notebook: data loading → RAG pipeline → Gradio UI → evaluation
├── report.md                 # Detailed technical report covering methodology and analysis
├── results.md                # Comprehensive evaluation results with per-query analysis
└── assets/                   # Architecture diagrams, screenshots, and visual assets
    └── rag_pipeline_architecture.png
```

| File | Description |
|---|---|
| `README.md` | Complete project documentation with pipeline details, results, and setup guide |
| `requirements.txt` | All Python packages with pinned versions for reproducibility |
| `MedicalRag.ipynb` | End-to-end implementation: data ingestion, embedding, RAG chain, Gradio UI, and batch evaluation |
| `report.md` | In-depth technical report covering architecture decisions, methodology, and limitations |
| `results.md` | Evaluation results across 33 queries with category-level analysis and findings |
| `assets/` | Visual assets including pipeline architecture diagram |

---

## 🔮 Future Work

- [ ] **Hybrid Retrieval**: Combine dense vector search with BM25 sparse retrieval for improved recall
- [ ] **Reranking**: Add a cross-encoder reranking step (e.g., `ms-marco-MiniLM-L-12-v2`) to improve precision
- [ ] **Multi-turn Conversation**: Implement chat history and follow-up question support
- [ ] **Evaluation Framework**: Add automated metrics (RAGAS, faithfulness, answer relevancy scores)
- [ ] **Full Dataset**: Scale from 1,500 to all 4,999 records with batch embedding
- [ ] **Streaming Responses**: Enable token-by-token streaming for better UX
- [ ] **Deployment**: Deploy to Hugging Face Spaces or Google Cloud Run for persistent hosting
- [ ] **Fine-tuned Embeddings**: Train domain-specific medical embeddings for improved retrieval
- [ ] **Multi-modal Support**: Incorporate medical imaging data alongside transcriptions
- [ ] **HIPAA Compliance**: Add data anonymization and access control for clinical deployment

---

## 📜 License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

---

## ⚠️ Disclaimer

> This is an **academic research project**. It is **not** intended for real clinical use. Always consult qualified healthcare professionals for medical advice. The system's answers are constrained to the provided dataset and should not be treated as medical recommendations.

---

<div align="center">

**Built with ❤️ using LangChain, Google Gemini, ChromaDB & Gradio**

**GitHub Topics:** `rag` · `medical-ai` · `langchain` · `gemini` · `chromadb` · `nlp` · `retrieval-augmented-generation` · `healthcare` · `gradio` · `vector-search`

</div>
