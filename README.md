# 🩸 THE 10-K FILES

### *200 pages. One number. And an AI that whispers: "I could not find it."*

```
 ╔══════════════════════════════════════════════════════════╗
 ║  INCIDENT REPORT #10-K                                   ║
 ║  STATUS ........ 🔴 UNRESOLVED                           ║
 ║  LOCATION ...... Retrieval layer (suspected)             ║
 ║  VICTIMS ....... 5 of 7 questions                        ║
 ║  SURVIVORS ..... 2                                       ║
 ╚══════════════════════════════════════════════════════════╝
```

---

## 🕯️ PROLOGUE

Every public company has to file a 10-K.

Two hundred pages of legal language, tables, footnotes, and numbers. Somewhere inside it, on exactly **one page**, sits the answer you need.

Most AI systems don't look for that page. They **guess**. And a confident wrong number in finance is worse than silence.

This project is a different kind of creature. It searches the filing, finds the evidence, answers with the **source page attached**, and when it can't find the answer, it **admits it**.

Then it turns around and **audits itself**, page by page, to find out exactly where it died.

Built on [FinanceBench](https://github.com/patronus-ai/financebench).

---

## 🕸️ THE RITUAL (Pipeline)

```
   📄  10-K PDF
        │
        ▼
   🔪  Cut into pages, then chunks
       (every fragment remembers its filing + page number)
        │
        ▼
   🧠  Embed the fragments            BAAI/bge-small-en-v1.5
        │
        ▼
   🗄️   Bury them in an index          FAISS
        │
        ▼
   🔦  Summon the top-k suspects       (retrieval)
        │
        ▼
   🎙️  Interrogate the witness         gpt-oss-120b on Groq
        │
        ▼
   🧾  Answer  +  the page it came from
```

| Stage | Weapon of choice |
|---|---|
| PDF parsing | PyMuPDF |
| Embeddings | `BAAI/bge-small-en-v1.5` |
| Vector search | FAISS (inner product, normalized) |
| LLM | `openai/gpt-oss-120b` via Groq |
| Interface | Gradio |

**The one rule the witness must obey:** speak only from the evidence in front of you. If it isn't there, say so.

---

## 🗂️ THE EVIDENCE LOCKER (Dataset)

**7 questions. 7
## 🔪 THE AUTOPSY (Results)

### Stage 1 · Did the retriever find the right page?

| Where the detective searched | Hit@1 | Hit@3 | Hit@5 | MRR |
|---|---|---|---|---|
| Inside the correct filing | 0.429 | 0.429 | 0.571 | **0.464** |
| Across all 7 filings | 0.000 | 0.429 | 0.429 | 0.190 |

> 🩸 **Finding #1.** Tell the detective *which building to search* and MRR jumps from **0.19 to 0.46**. Without that hint, the correct page was **never ranked first**. Not once.

### Stage 2 · Does chunk size matter?

| Chunk size | Overlap | Chunks | Hit@1 | Hit@3 | Hit@5 | Hit@10 | MRR |
|---|---|---|---|---|---|---|---|
| 500 | 75 | 10,245 | 0.429 | 0.429 | 0.571 | 0.571 | 0.464 |
| 1000 | 150 | 5,407 | 0.429 | 0.429 | 0.571 | 0.571 | 0.464 |
| 1500 | 200 | 3,728 | 0.429 | 0.429 | 0.571 | **0.714** | 0.475 |

> 🩸 **Finding #2.** Bigger chunks help a little at Hit@10, but the difference is tiny. Chunk size is **not** the killer.

### Stage 3 · The final verdict

Chunk size 1000 · top-5 · graded by an LLM judge against the reference answer.

| Metric | Result |
|---|---|
| Answer accuracy | **2 / 7** (28.6%) |
| Right page retrieved | **4 / 7** (57.1%) |

---

## 👁️ THE TWIST

Look closely at these two rows.

| What happened | Questions | Answered correctly |
|---|---|---|
| ☠️ The right page was **never found** | 3 | **0 / 3** |
| 🕯️ The right page was found | 4 | **2 / 4** |

In all 3 cases where the evidence was missing, the AI did **not** invent an answer.

It said:

> ***"I could not find it."***

No hallucination. No confident lie. Just silence where the evidence should have been.

**The language model was never the monster. The retriever was.**

And when the right page *did* surface, the witness delivered:

- 🧮 FY2021 unadjusted EBITDA: $1,196M + $636M = **$1,832M** ✅
- 🧮 Best Buy 3-year average net profit margin: **2.8%** ✅

---

## ⚰️ THE GRAVEYARD (Failure Analysis)

Five questions didn't make it out. Here's how each one died.

| Victim | Cause of death |
|---|---|
| 💀 AmEx card member retention (2022) | Retriever never found the page |
| 💀 Best Buy FY2019 inventories | Retriever never found the page |
| 💀 Best Buy gross margin consistency | Retriever never found the page |
| 💀 AWW working capital (FY2022) | Evidence spans **2 pages**. Only 1 was retrieved, so current assets went missing |
| 💀 AWW cash dividends (FY2020) | The right page was found, but the number wasn't inside the retrieved chunks |

Per-question results: [`outputs/answer_eval_results.csv`](outputs/answer_eval_results.csv)

## 🔥 THE EXORCISM (Next Steps)

How I'd drive the demon out of the retrieval layer:

- 🔤 **Hybrid search (BM25 + embeddings).** Exact figures like "5,409" are keyword problems, and dense search is bad at them.
- 🎯 **Cross-encoder reranker.** Re-score the top candidates before the LLM ever sees them.
- 📊 **Table-aware parsing.** Financial answers live in tables, and plain text extraction flattens them into noise.
- 📚 **Full 150-question FinanceBench.** Seven questions can't support strong conclusions.

---

## ⚡ SUMMON IT YOURSELF

1. Open `financebench_rag.ipynb` in **Google Colab** (GPU runtime recommended).
2. Add a free [Groq](https://console.groq.com) API key as a Colab Secret named `GROQ_API_KEY` (notebook access **ON**).
3. Upload `data/financebench_data.csv` when prompted and run the cells in order. The 10-K PDFs are pulled automatically from the FinanceBench repo.

```
📁 financebench-rag-qa
 ├── financebench_rag.ipynb     the full investigation (Colab)
 ├── data/                      questions, answers, evidence
 ├── outputs/                   evaluation results (CSV)
 ├── screenshots/               results and demo
 └── requirements.txt
```

<sub>🕯️ *Case status: still open. The detective is honest, but it can't find every page yet. Pull requests welcome. Enter at your own risk.*</sub>
