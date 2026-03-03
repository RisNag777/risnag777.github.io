---
layout: post
title: "Designing an Agentic Textbook TA"
---

## Introduction

When you ask an LLM a technical question, it answers from whatever it absorbed during training. Sometimes that's useful. Sometimes it's outdated. Sometimes it's just wrong. And most of the time, you have no idea *where* the answer came from.

If I'm reading a specific textbook and I have a question about the material, I don't want an answer stitched together from half-remembered internet patterns. I want a system that behaves the way a teacher or a TA would: open the book, find the relevant section, and reason from what's actually written there. For example, if I want to know about the Central Limit Theorem, I'd like the answer to come from page 368 of the textbook.

<img width="340" height="438" alt="image" src="https://github.com/user-attachments/assets/23e37905-0a97-4efa-bfed-11aed985eae7" />

This blog post walks through how I built such a system, a **Textbook Teaching Assistant** that answers questions by consulting a textbook at runtime, not its own training-time memory. I'll cover the core idea, the architecture, the agent loop, what worked, what didn't, and how to run it yourself.

---

## The Core Idea

Traditional "ask an LLM" setups treat the model as if it's the whole system: you send a question, you get an answer, and you hope it's right.

Here, the framing is different:

- **The textbook is the source of truth.**
- **The LLM is just the reasoning layer over that textbook.**
- **The system logic enforces that answers stay grounded in the book.**

Instead of asking "Can the LLM answer this?", the real question becomes:

> How do you turn an LLM into a reasoning system that consults a book before it speaks?

Concretely, the system:

1. Processes a PDF textbook into semantically coherent chunks with chapter and page metadata.
2. Embeds those chunks into a vector database (FAISS + sentence transformers).
3. Uses the LLM to **select relevant chapters**, given a question.
4. Uses **semantic search** over those chapters to find the most relevant chunks.
5. Asks the LLM to **synthesize an answer only from those chunks**.
6. Asks the LLM to **self-evaluate** whether the answer is complete and well-supported.

The result is an agent that behaves less like a chatbot and more like a TA flipping through a textbook.

---

## System Architecture

At a high level, the system is a knowledge pipeline wrapped around a reasoning engine:

| Component      | Role in the System                            |
|----------------|-----------------------------------------------|
| Textbook       | The source of truth (external knowledge)      |
| Retrieval      | Memory access mechanism (vector search)       |
| LLM            | Reasoning engine over retrieved context       |
| System logic   | Enforces grounding in the textbook            |

```html
<svg xmlns="http://www.w3.org/2000/svg" viewBox="-30 0 700 280" width="600" height="280">
  <defs>
    <style>
      .box { fill: #f8f9fa; stroke: #495057; stroke-width: 2; rx: 8; }
      .box-knowledge { fill: #e7f5ff; stroke: #1971c2; }
      .box-reasoning { fill: #fff3bf; stroke: #f59f00; }
      .box-control { fill: #d3f9d8; stroke: #2f9e44; }
      .arrow { stroke: #495057; stroke-width: 2; fill: none; marker-end: url(#arrowhead); }
      .label { font-family: system-ui, sans-serif; font-size: 14px; fill: #212529; }
      .title { font-family: system-ui, sans-serif; font-size: 12px; fill: #868e96; font-weight: 500; }
    </style>
    <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#495057"/>
    </marker>
  </defs>
  <!-- Textbook -->
  <rect x="20" y="40" width="120" height="60" class="box box-knowledge"/>
  <text x="80" y="65" class="label" text-anchor="middle">Textbook</text>
  <text x="80" y="85" class="title" text-anchor="middle">Source of truth</text>
  
  <!-- Vector DB -->
  <rect x="190" y="40" width="120" height="60" class="box box-knowledge"/>
  <text x="250" y="65" class="label" text-anchor="middle">Vector DB</text>
  <text x="250" y="85" class="title" text-anchor="middle">FAISS + embeddings</text>
  
  <!-- LLM -->
  <rect x="360" y="40" width="120" height="60" class="box box-reasoning"/>
  <text x="420" y="65" class="label" text-anchor="middle">LLM</text>
  <text x="420" y="85" class="title" text-anchor="middle">Reasoning layer</text>
  
  <!-- System Logic -->
  <rect x="540" y="40" width="80" height="60" class="box box-control"/>
  <text x="580" y="65" class="label" text-anchor="middle">Logic</text>
  <text x="580" y="85" class="title" text-anchor="middle">Grounding</text>
  
  <!-- Arrows -->
  <path d="M 140 70 L 190 70" class="arrow"/>
  <path d="M 310 70 L 360 70" class="arrow"/>
  <path d="M 480 70 L 540 70" class="arrow"/>
  
  <!-- Data flow labels -->
  <text x="165" y="55" class="title" text-anchor="middle">chunks</text>
  <text x="335" y="55" class="title" text-anchor="middle">context</text>
  <text x="510" y="55" class="title" text-anchor="middle">answer</text>
  
  <!-- Question input -->
  <rect x="360" y="160" width="120" height="40" class="box" stroke-dasharray="4"/>
  <text x="420" y="185" class="label" text-anchor="middle">Question</text>
  <path d="M 420 160 L 420 100" class="arrow" stroke-dasharray="4"/>
</svg>
```

The LLM is not an all-knowing brain here. It's a processor that operates **inside** a controlled knowledge system, over context that the system chooses and provides.

---

## Key Components

This project revolves around three main modules: PDF processing, vector search, and the agent.

### 1. PDF Processing & Chunking (`load_data.py`)

The goal of this stage is to turn a messy PDF into clean, searchable chunks.

- **PDF parsing**: Uses PyMuPDF (`fitz`) to:
  - Extract text per page.
  - Extract the table of contents (TOC) to identify chapters/sections.
- **Chunking**:
  - Splits text into paragraph-level chunks (~300 tokens).
  - Preserves metadata: `chapter_title`, `page_num`, `book`.
- **Output schema**: Each chunk looks like:

```json
{
  "chapter_title": "12.3 The Regression Equation",
  "page_num": 638,
  "chunk_text": "In this section, we define the regression equation...",
  "book": "Introductory Statistics 2e"
}
```

These chunks are saved as a JSON file (e.g., `textbook.pdf.json`).

### 2. Vector Database (`vector_db.py`)

Once we have structured chunks, we build a semantic search index.

- **Embeddings**: Sentence Transformer `all-MiniLM-L6-v2`.
- **Index**: FAISS `IndexFlatL2` for fast similarity search.
- **Core methods**:
  - `build_index()`: Load all textbook JSONs and build the index.
  - `query(question, top_k=5, chapter_keywords=None)`:
    - Embed the question.
    - Optionally filter chunks to specific chapters.
    - Return top-k results with metadata and distances.
  - `list_chapters()`: List all chapter titles in sorted order.

For *Introductory Statistics 2e*, this produces **827 searchable chunks**.

### 3. Textbook Agent (`agent.py`)

The `TextbookAgent` orchestrates the entire reasoning loop.

- `choose_chapters(question, available_chapters)`:
  - Asks the LLM to select chapter titles from the TOC that seem relevant.
  - Returns a list of chapter titles.
- `answer_question(question, top_k=5, model="gpt-4o-mini")`:
  1. Get available chapters from the vector DB.
  2. Use the LLM to pick the most relevant chapters.
  3. Query the vector DB with `chapter_keywords` to focus retrieval.
  4. Synthesize an answer from the retrieved chunks.
  5. Ask the LLM to evaluate if the answer is complete.
  6. Return `(answer, chunks)` (or `("", "")` if nothing suitable is found).
- `synthesize_answer(question, chunks)`:
  - Prompts the LLM to answer **only using the given chunks**.
- `evaluate_answer(question, answer)`:
  - Asks the LLM: "Is this answer COMPLETE and WELL-SUPPORTED? Return YES or NO."

Each chunk is labeled with `[Chapter: X, Page: Y]`, so answers are traceable back to specific pages.

---

## Pipeline Flow

At runtime, answering a question is not a single LLM call. It's a small agent loop:

```text
Question
  ↓
1. Interpret the question (chapter selection)
  ↓
2. Retrieve context (chapter-filtered vector search)
  ↓
3. Synthesize answer from chunks
  ↓
4. Evaluate answer (YES/NO)
  ↓
5. Stop or (in future) continue with refined search
```

```html
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 410" width="500" height="380">
  <defs>
    <style>
      .step { fill: #fff; stroke: #495057; stroke-width: 2; rx: 6; }
      .step-decision { fill: #fff3bf; stroke: #f59f00; }
      .arrow { stroke: #495057; stroke-width: 2; fill: none; marker-end: url(#arrowhead); }
      .label { font-family: system-ui, sans-serif; font-size: 13px; fill: #212529; }
      .small { font-family: system-ui, sans-serif; font-size: 11px; fill: #868e96; }
    </style>
    <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#495057"/>
    </marker>
  </defs>
  <!-- 1. Interpret -->
  <rect x="150" y="20" width="220" height="50" class="step"/>
  <text x="260" y="45" class="label" text-anchor="middle">1. Interpret the question</text>
  <text x="260" y="62" class="small" text-anchor="middle">LLM selects relevant chapters from TOC</text>
  
  <!-- 2. Retrieve -->
  <rect x="150" y="100" width="220" height="50" class="step"/>
  <text x="260" y="125" class="label" text-anchor="middle">2. Retrieve context</text>
  <text x="260" y="142" class="small" text-anchor="middle">Vector search with chapter filter (top_k=5)</text>
  
  <!-- 3. Synthesize -->
  <rect x="150" y="180" width="220" height="50" class="step"/>
  <text x="260" y="205" class="label" text-anchor="middle">3. Synthesize answer</text>
  <text x="260" y="222" class="small" text-anchor="middle">LLM answers from chunks only</text>
  
  <!-- 4. Evaluate -->
  <rect x="150" y="260" width="220" height="50" class="step step-decision"/>
  <text x="260" y="285" class="label" text-anchor="middle">4. Evaluate answer</text>
  <text x="260" y="302" class="small" text-anchor="middle">Self-check: complete & grounded?</text>
  
  <!-- 5. Stop or continue -->
  <rect x="150" y="340" width="220" height="50" class="step"/>
  <text x="260" y="365" class="label" text-anchor="middle">5. Stop or continue</text>
  <text x="260" y="382" class="small" text-anchor="middle">YES => return | NO => refine & repeat</text>
  
  <!-- Arrows -->
  <path d="M 250 70 L 250 100" class="arrow"/>
  <path d="M 250 150 L 250 180" class="arrow"/>
  <path d="M 250 230 L 250 260" class="arrow"/>
  <path d="M 250 310 L 250 340" class="arrow"/>
  
  <!-- Loop back arrow (NO path) -->
  <path d="M 150 285 L 80 285 L 80 125 L 150 125" class="arrow" stroke="#e03131" stroke-dasharray="4"/>
  <text x="100" y="200" class="small" fill="#e03131">NO</text>
</svg>
```

**Current implementation detail**: If the evaluation step returns `NO`, the agent stops and reports that it couldn't confidently answer. A future iteration could refine the search and loop back.

---

## Detailed Pipeline Steps

Putting it all together, one question goes through these stages:

1. **Parse and index the textbook (offline, once per book)**  
   - Extract text and TOC via PyMuPDF.  
   - Chunk text into ~300-token segments with chapter and page metadata.  
   - Embed chunks with `SentenceTransformer(all-MiniLM-L6-v2)`.  
   - Build a FAISS `IndexFlatL2` index.

2. **Interpret the question (chapter selection)**  
   - Call the LLM with the question and a list of chapter titles.  
   - Ask it to return a JSON array of chapters likely to be relevant.

3. **Retrieve context (vector search)**  
   - Query the vector DB with:
     - The question embedding.
     - `chapter_keywords` equal to the selected chapters.
   - Return top `k` chunks ranked by L2 distance.

4. **Synthesize the answer**  
   - Concatenate retrieved chunks with `[Chapter: X, Page: Y]` headers.
   - Prompt the LLM to:
     - Answer the question.
     - Use **only** the provided textbook content (but still do its best if incomplete).

5. **Self-evaluate**  
   - Ask the LLM: is this answer **complete and well-supported** by the context?  
   - Force a one-word response: `YES` or `NO`.

6. **Return or (future) refine**  
   - If `YES`: return the answer plus the list of chunks.  
   - If `NO`: return an empty answer or a fallback; a future version will refine the search and loop back.

---

## Technical Deep Dive

### Chunking & TOC Alignment

The quality of retrieval depends heavily on:

- **TOC extraction**: If the PDF has a clean TOC, chapter boundaries are meaningful; if not, retrieval degrades.
- **Chunk size**: ~300 tokens strikes a balance:
  - Long enough to contain coherent ideas.
  - Short enough to combine multiple chunks in a single LLM context window.
- **Metadata**: Attaching `chapter_title` and `page_num` to each chunk allows:
  - Chapter-level filtering before vector search.
  - Page-level traceability in the final answer.

### Vector Search Implementation

The vector database:

- Uses `all-MiniLM-L6-v2` for embeddings (a strong speed/quality trade-off).
- Indexes all chunks via FAISS `IndexFlatL2`.
- Filters by chapter **before** scoring:
  - This drastically reduces the number of candidates.
  - It also makes retrieval semantically sharper, since we're not competing across the entire book.

### Synthesis Prompt

The synthesis step makes the LLM behave like a TA reading the book, not like a general chatbot:

```python
prompt = f"""
You are a helpful teaching assistant. Use the following textbook
content to answer the question below.
Answer only based on the textbook content, but do your best even if
the answer is not explicitly stated.

Textbook content:
{context}

Question: {question}
"""
```

Every chunk in `context` is prefixed with something like:

```text
[Chapter: 12.3 The Regression Equation, Page: 638]
...
```

This helps both the model and the user understand where each part of the answer comes from.

### Self-Evaluation Prompt

The self-check is intentionally simple and binary:

```text
Is this answer COMPLETE and WELL-SUPPORTED by the textbook content?

Respond with ONLY ONE WORD: YES or NO.
```

This turns the LLM into a cheap, probabilistic quality gate. It doesn't guarantee correctness, but it's a useful extra layer that often flags incomplete answers.

### Architectural Shift: LLM as Reasoning Layer

The central design decision is:

- **Knowledge** lives in:
  - The textbook.
  - The chunked JSON.
  - The vector index.
- **Reasoning** lives in:
  - The LLM calls (chapter selection, synthesis, evaluation).
- **Control** lives in:
  - The `TextbookAgent` logic (what to call when, and how to interpret results).

This explicit separation makes the system more debuggable and controllable than a single "ask the LLM" call.

---

## What Worked Well

- **Grounded answers**:  
  Because all answers are derived from retrieved textbook chunks, it's much easier to spot and correct issues. You can literally see which pages the model used.

- **Chapter-based filtering**:  
  Using the LLM to select chapters before vector search significantly improves retrieval quality, especially for books with many chapters.

- **Simple, composable prompts**:  
  Splitting the agent loop into:
  - Chapter selection
  - Synthesis
  - Evaluation  
  made prompts shorter, clearer, and easier to iterate on.

- **Traceability**:  
  Including `[Chapter, Page]` in both the chunks and the synthesized answer gives the system a "show your work" property, similar to how a human TA would answer.

---

## What Didn't Work (and What I Learned)

- **One-shot retrieval isn't always enough**  
  A single retrieval pass sometimes misses important context, especially for:
  - Multi-part questions.
  - Concepts that are defined in one chapter and used in another.  
  The current system stops after one retrieval; a future version will loop, refine, and re-retrieve.

- **Over-reliance on TOC quality**  
  When the PDF's TOC is noisy or missing, chapter selection becomes brittle.
  - Lesson: the system should be able to fall back to alternative grouping strategies (e.g., purely page-based or embedding-based clustering).

- **Prompt-only grounding is probabilistic**  
  Despite clear instructions, the model can still sometimes blend in prior training knowledge.
  - Lesson: prompts significantly reduce hallucinations but do not mathematically eliminate them; stronger constraints would require different tooling (e.g., tool-usage constraints, retrieval-only answer modes, etc.).

- **Single-textbook assumption**  
  The current implementation assumes "one textbook at a time".
  - Extending to multiple books introduces new questions: how to handle conflicting explanations, multiple editions, and much larger indexes.

---

## Limitations

### Retrieval quality depends on chunking and TOC

- If the TOC is poor or text extraction is messy (e.g., PDFs with broken paragraphs), chunks can become:
  - Too fragmented.
  - Misaligned with chapters.
- This hurts both chapter selection and vector search.

### Approximate vector search

- FAISS + sentence transformers work well in practice, but:
  - Some subtle or cross-chapter questions still get slightly off-target chunks.
  - The self-evaluation step can only flag some of these issues.

### Multiple LLM calls per question

- Each question triggers several calls:
  - Chapter selection.
  - Synthesis.
  - Evaluation.
- That's fine for a classroom or personal tool, but it's slower and more expensive than a bare single-shot LLM call.

### Prompt-based grounding only

- Grounding is enforced via instructions, not hard constraints.
- Under adversarial or tricky prompts, the model can still hallucinate.

### Single-textbook scope

- Currently tuned for one statistics textbook and one embedding space.
- Multiple textbooks would require:
  - Cross-book disambiguation.
  - Smarter indexing and retrieval strategies.

---

## Code Highlights

### Agent Orchestration

The `TextbookAgent` glues together chapter selection, retrieval, synthesis, and evaluation:

```python
# From agent.py: chapter selection narrows the search space
chapters, consulted_chapters = self.find_chapters(
    question, available_chapters, consulted_chapters
)
results = self.db.query(
    question,
    top_k=5,
    chapter_keywords=chapters
)

answer = self.synthesize_answer(question, all_chunks)
verdict = self.evaluate_answer(question, answer)  # "YES" or "NO"
```

The design goal is for each step to be understandable and testable in isolation.

### Synthesis Prompt (Grounded Answering)

```python
prompt = f"""
You are a helpful teaching assistant. Use the following textbook
content to answer the question below.
Answer only based on the textbook content, but do your best even if
the answer is not explicitly stated.

Textbook content:
{context}

Question: {question}
"""
```

---

## Tutorial 1: Setting Up the Project Locally

### Prerequisites

- Python 3.8+
- OpenAI API key
- Git

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd autonomous-ta
```

### Step 2: Create a Virtual Environment

```bash
python -m venv venv
# Windows:
venv\Scripts\activate
# macOS/Linux:
# source venv/bin/activate
```

### Step 3: Install Dependencies

```bash
pip install -r requirements.txt
```

This includes:

- `openai` – OpenAI API client
- `sentence-transformers` – Text embeddings
- `faiss-cpu` – Vector similarity search
- `PyMuPDF` (`fitz`) – PDF parsing
- `numpy`, `dotenv`, etc.

### Step 4: Configure Environment Variables

Create a `.env` file in the project root:

```bash
OPENAI_API_KEY=your_api_key_here
```

---

## Tutorial 2: Using the Textbook TA

### Step 1: Prepare Your Textbook

- Place your PDF textbook in `data/raw/`.
- The system will process any PDFs found there.

### Step 2: Parse the Textbook

```python
from src.autonomous_ta.load_data import parse_book

parse_book()
```

This will:

- Extract text from all pages.
- Extract the TOC.
- Chunk the text into ~300-token segments.
- Save the processed JSON file (e.g., `data/processed/textbook.pdf.json`).

### Step 3: Initialize the Agent

```python
from src.autonomous_ta.agent import TextbookAgent

agent = TextbookAgent(model="gpt-4o-mini")
```

The agent will:

- Load the processed textbook data.
- Build the vector index.
- Get ready for queries.

### Step 4: Ask Questions

```python
question = "What is regression and why is it useful? What are the disadvantages of using regression?"
answer, chunks = agent.answer_question(question, top_k=5)

print(answer)
for c in chunks:
    print(c["chapter"], c["page"])
```

You can also use the CLI:

```bash
python cli.py "Explain the central limit theorem" --model gpt-4o-mini --top-k 10 --verbose
```

---

## Performance & Behavior (Qualitative)

In practice, for a textbook like *Introductory Statistics 2e*:

- **Index build time**: A one-time cost (seconds to minutes, depending on size).
- **Per-question latency**:
  - Vector search: fast (milliseconds).
  - LLM calls: dominate latency (a few seconds each).
- **Behavior**:
  - For well-defined textbook questions (e.g., "What is a confidence interval?"), the system retrieves the right sections and produces grounded, page-cited answers.
  - For vague or cross-cutting questions, the self-evaluation step often flags incomplete answers, which is preferable to confident hallucinations.

---

## Future Improvements

### Architectural Enhancements

- **Multi-textbook support**:
  - Shared embedding space across multiple books.
  - Source-aware retrieval and conflict handling.
- **Iterative agent loop**:
  - When evaluation returns `NO`, refine the question or chapter selection and try again.
- **Smarter retrieval**:
  - Combine chapter filters with learned rerankers.
  - Use hybrid search (sparse + dense) for better coverage.

### Reliability & UX

- **Stronger grounding mechanisms**:
  - Tooling that enforces retrieval-only answers more strictly than prompts alone.
- **Caching and indexing strategies**:
  - Cache embeddings and index shards for faster startup.
- **Better TOC handling**:
  - Fallback strategies when TOC is missing or malformed.

---

## Conclusion

Building this Textbook TA changed how I think about LLMs.

Their real strength isn't in "knowing everything." It's in **reasoning over information** when you give them the right structure and constraints. When you move knowledge out of the model and into a system that manages memory, retrieval, and control flow, you get behavior that feels more like a careful TA and less like a guessing chatbot.

The intelligence of this system doesn't live in the model or the database alone. It emerges from how structured memory (the textbook + vector DB) and probabilistic reasoning (the LLM) are combined under explicit control.

---

## Implementation at a Glance

| Layer           | Technology                              |
|-----------------|-----------------------------------------|
| PDF parsing     | PyMuPDF (`fitz`), TOC extraction        |
| Chunking        | Paragraph-based, ~300 tokens, chapters  |
| Embeddings      | SentenceTransformer `all-MiniLM-L6-v2`  |
| Vector index    | FAISS `IndexFlatL2`                     |
| LLM             | GPT-4o-mini (OpenAI API)               |
| Agent logic     | `TextbookAgent` (Python)               |

---

## Resources

- **GitHub Repository**: `https://github.com/RisNag777/autonomous-ta`
- **OpenAI API Docs**: `https://platform.openai.com/docs`
- **Sentence Transformers**: `https://www.sbert.net/`
- **FAISS**: `https://github.com/facebookresearch/faiss`
- **PyMuPDF Docs**: `https://pymupdf.readthedocs.io/`

*This project is a small step toward building more reliable, grounded AI systems where models reason over explicit knowledge, instead of pretending to be the knowledge themselves.*

