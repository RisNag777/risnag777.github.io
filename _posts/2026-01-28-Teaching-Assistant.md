---
layout: post
title: "Designing an Agentic Textbook TA"
---

When you ask an LLM a technical question, it answers from whatever it absorbed during training. Sometimes that's useful. Sometimes it's outdated. Sometimes it's just wrong. And most of the time, you have no idea *where* the answer came from.

This lack of control bothered me. If I'm reading a specific textbook and I have a question about the material, I don't want an answer stitched together from half-remembered internet patterns. I want a system that behaves the way a teacher or a TA would - open the book, find the relevant section, and reason from what's actually written there. For example, if I want to know about the Central Limit Theorem, I would like the answer to come from page 368 of the textbook!

<img width="340" height="438" alt="image" src="https://github.com/user-attachments/assets/23e37905-0a97-4efa-bfed-11aed985eae7" />

So I set out to build exactly that. A system where the model doesn't answer from memory, but from a textbook provided at runtime. However, the problem here is not about making the model smarter, it's about giving the model a structured way to access knowledge and constraining it to use that knowledge as its source of truth. Instead of asking, "Can the LLM answer this?", the question becomes "How do you turn an LLM into a reasoning system that consults a book before it speaks?"

## The Knowledge System.

At its core, the problem looks like this:

| Component | Role in the System |
|-----------|--------------------|
| Textbook | The source of truth (external knowledge) |
| Retrieval system | Memory access mechanism |
| LLM | Reasoning engine operating over retrieved context |
| System logic | Enforces that answers stay grounded in the textbook |

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 600 280" width="600" height="280">
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
  <rect x="180" y="40" width="120" height="60" class="box box-knowledge"/>
  <text x="240" y="65" class="label" text-anchor="middle">Vector DB</text>
  <text x="240" y="85" class="title" text-anchor="middle">FAISS + embeddings</text>
  
  <!-- LLM -->
  <rect x="340" y="40" width="120" height="60" class="box box-reasoning"/>
  <text x="400" y="65" class="label" text-anchor="middle">LLM</text>
  <text x="400" y="85" class="title" text-anchor="middle">Reasoning layer</text>
  
  <!-- System Logic -->
  <rect x="500" y="40" width="80" height="60" class="box box-control"/>
  <text x="540" y="65" class="label" text-anchor="middle">Logic</text>
  <text x="540" y="85" class="title" text-anchor="middle">Grounding</text>
  
  <!-- Arrows -->
  <path d="M 140 70 L 180 70" class="arrow"/>
  <path d="M 300 70 L 340 70" class="arrow"/>
  <path d="M 460 70 L 500 70" class="arrow"/>
  
  <!-- Data flow labels -->
  <text x="160" y="55" class="title" text-anchor="middle">chunks</text>
  <text x="320" y="55" class="title" text-anchor="middle">context</text>
  <text x="480" y="55" class="title" text-anchor="middle">answer</text>
  
  <!-- Question input -->
  <rect x="340" y="160" width="120" height="40" class="box" stroke-dasharray="4"/>
  <text x="400" y="185" class="label" text-anchor="middle">Question</text>
  <path d="M 400 160 L 400 100" class="arrow" stroke-dasharray="4"/>
</svg>

Instead of treating the LLM as a giant, all-knowing brain, this architecture treats it as something very different - a processor that reasons over information it is given. The knowledge itself lives outside the model, stored in a structured form that the system can search and control.

The pipeline starts with the textbook. The PDF is split into chunks, converted into embeddings, and stored in a vector database. That database acts as a searchable memory. When a question comes in, the system doesn't immediately ask the model to answer. Instead, it first asks, "What parts of the book are relevant to this question?"

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 550 200" width="550" height="200">
  <defs>
    <style>
      .stage { fill: #f8f9fa; stroke: #495057; stroke-width: 2; rx: 8; }
      .arrow { stroke: #495057; stroke-width: 2; fill: none; marker-end: url(#arrowhead); }
      .label { font-family: system-ui, sans-serif; font-size: 14px; fill: #212529; }
      .code { font-family: 'Consolas', 'Monaco', monospace; font-size: 11px; fill: #495057; }
    </style>
    <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#495057"/>
    </marker>
  </defs>
  <!-- PDF -->
  <rect x="20" y="60" width="100" height="80" class="stage"/>
  <text x="70" y="95" class="label" text-anchor="middle">PDF</text>
  <text x="70" y="115" class="code" text-anchor="middle">load_pdf()</text>
  <text x="70" y="130" class="code" text-anchor="middle">get_toc()</text>
  
  <!-- Chunks -->
  <rect x="160" y="60" width="120" height="80" class="stage"/>
  <text x="220" y="95" class="label" text-anchor="middle">Chunks</text>
  <text x="220" y="115" class="code" text-anchor="middle">chunk_text()</text>
  <text x="220" y="130" class="code" text-anchor="middle">~300 tokens, by chapter</text>
  
  <!-- Embeddings -->
  <rect x="320" y="60" width="120" height="80" class="stage"/>
  <text x="380" y="95" class="label" text-anchor="middle">Embeddings</text>
  <text x="380" y="115" class="code" text-anchor="middle">SentenceTransformer</text>
  <text x="380" y="130" class="code" text-anchor="middle">all-MiniLM-L6-v2</text>
  
  <!-- FAISS Index -->
  <rect x="480" y="60" width="120" height="80" class="stage"/>
  <text x="540" y="95" class="label" text-anchor="middle">FAISS Index</text>
  <text x="540" y="115" class="code" text-anchor="middle">IndexFlatL2</text>
  <text x="540" y="130" class="code" text-anchor="middle">827 chunks</text>
  
  <!-- Arrows -->
  <path d="M 120 100 L 160 100" class="arrow"/>
  <path d="M 280 100 L 320 100" class="arrow"/>
  <path d="M 440 100 L 480 100" class="arrow"/>
</svg>

In this implementation, the pipeline uses PyMuPDF to extract text and table-of-contents from the PDF, then chunks by paragraph (~300 tokens) while preserving chapter metadata. Each chunk becomes a vector via `SentenceTransformer` (all-MiniLM-L6-v2), stored in a FAISS index. For *Introductory Statistics 2e*, that yields 827 searchable chunks.

Only after retrieving context from the textbook does the model step in. At that point, its job is not to recall facts from training, but to read, synthesize, and explain what's already been found. In other words, the model is operating more like a reasoning layer sitting on top of a knowledge store, rather than a knowledge store itself.

This shift, from "the model knows things" to "the model operates inside a knowledge system", is what turns a chatbot-style setup into something closer to a controlled reasoning system.

## The Agent Loop

A simple retrieval system would stop at:

`question -> retrieve chunks -> answer`

But that assumes the first retrieval step is enough. In practice, it often isn't. Some questions span multiple sections of a textbook. Others require definitions from one chapter and methods from another. A single pass at retrieval can miss important context.

So the system treats answering as a process:

1. **Interpret the question**  
   The model helps determine which parts of the textbook are likely to be relevant.

2. **Retrieve context**  
   The system pulls chunks from the vector database corresponding to those sections.

3. **Synthesize an answer**  
   The model generates a response based only on the retrieved material.

4. **Evaluate the answer**  
   The system checks whether the answer appears complete and grounded in the available context.

5. **Stop or continue**  
   If the answer is sufficient, return it. If not, the system can refine the search and repeat.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 380" width="500" height="380">
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
  <rect x="150" y="20" width="200" height="50" class="step"/>
  <text x="250" y="45" class="label" text-anchor="middle">1. Interpret the question</text>
  <text x="250" y="62" class="small" text-anchor="middle">LLM selects relevant chapters from TOC</text>
  
  <!-- 2. Retrieve -->
  <rect x="150" y="90" width="200" height="50" class="step"/>
  <text x="250" y="115" class="label" text-anchor="middle">2. Retrieve context</text>
  <text x="250" y="132" class="small" text-anchor="middle">Vector search with chapter filter (top_k=5)</text>
  
  <!-- 3. Synthesize -->
  <rect x="150" y="160" width="200" height="50" class="step"/>
  <text x="250" y="185" class="label" text-anchor="middle">3. Synthesize answer</text>
  <text x="250" y="202" class="small" text-anchor="middle">LLM answers from chunks only</text>
  
  <!-- 4. Evaluate -->
  <rect x="150" y="230" width="200" height="50" class="step step-decision"/>
  <text x="250" y="255" class="label" text-anchor="middle">4. Evaluate answer</text>
  <text x="250" y="272" class="small" text-anchor="middle">Self-check: complete & grounded?</text>
  
  <!-- 5. Stop or continue -->
  <rect x="150" y="300" width="200" height="50" class="step"/>
  <text x="250" y="325" class="label" text-anchor="middle">5. Stop or continue</text>
  <text x="250" y="342" class="small" text-anchor="middle">YES � return | NO � refine & repeat</text>
  
  <!-- Arrows -->
  <path d="M 250 70 L 250 90" class="arrow"/>
  <path d="M 250 140 L 250 160" class="arrow"/>
  <path d="M 250 210 L 250 230" class="arrow"/>
  <path d="M 250 280 L 250 300" class="arrow"/>
  
  <!-- Loop back arrow (NO path) -->
  <path d="M 150 255 L 80 255 L 80 135 L 150 135" class="arrow" stroke="#e03131" stroke-dasharray="4"/>
  <text x="100" y="200" class="small" fill="#e03131">NO</text>
</svg>

In the code, step 1 is implemented by asking the LLM to return a JSON array of chapter titles from the textbook's table of contents. Step 2 uses that to *filter* the vector search. Only chunks from selected chapters are considered, then the top 5 by L2 distance are retrieved. Step 4 uses a simple self-evaluation prompt: "Is this answer COMPLETE and WELL-SUPPORTED? Respond with ONLY ONE WORD: YES or NO."

```
# From agent.py: chapter selection narrows the search space
chapters, consulted_chapters = self.find_chapters(question, available_chapters, consulted_chapters)
results = self.db.query(question, top_k=5, chapter_keywords=chapters)
answer = self.synthesize_answer(question, all_chunks)
verdict = self.evaluate_answer(question, answer)  # YES or NO
```

*(Currently, when the verdict is NO, the agent ends its computation)*

This structure turns the system into something that behaves less like a chatbot and more like an agent navigating a knowledge space. The model isn't just generating text, it's participating in a perception–action loop:

- **Perception**: retrieved textbook chunks  
- **Action**: choosing how to use that information  
- **Control**: system logic deciding whether to stop or keep searching  

That control flow shifts responsibility for knowledge from the model's parameters to the system's architecture.

## The Real Role of the LLM

In this system, the LLM isn't the knowledge source.  
It's the reasoning layer.

| Task | System |
|------|--------|
| Store knowledge | Textbook + vector database |
| Retrieve relevant material | Embedding search |
| Synthesize and explain | LLM |
| Decide how to use context | LLM |
| Enforce grounding | System logic |

The model never sees the entire textbook at once. It only sees the chunks the system retrieves. Its job is to read those chunks, connect the dots, and produce a coherent explanation.

The synthesis prompt enforces this explicitly:

```
# From agent.py - the model is instructed to answer only from context
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

Each retrieved chunk is prefixed with `[Chapter: X, Page: Y]` so the model and user can trace answers back to specific pages. For a question like "What is regression and why is it useful?", the system might consult chapters like "12.3 The Regression Equation" and return chunks from pages 637–642, then synthesize an answer grounded in those passages.

This flips the usual assumption about LLMs. Instead of asking the model to remember facts, the system gives it the facts and asks it to reason over them. Knowledge lives outside the model, and the model operates inside a controlled information environment.

## What This Project Really Shows

At first glance, this looks like "LLM + vector database." But the deeper shift is architectural.

The system doesn't treat the model as the place where knowledge lives. It treats the model as a component inside a larger structure that manages knowledge explicitly. The textbook is the source of truth, the vector database is the memory layer, and the LLM is a reasoning module that operates over whatever information the system provides.

This is a different way of thinking about LLM limitations. The problem isn't just that models hallucinate. It's that we often use them as if they were self-contained knowledge systems. When we move knowledge out of the model and into the system, the role of the LLM becomes clearer and more manageable.

This project isn't about making the model smarter. It's about building a structure in which the model's strengths (language understanding and synthesis) are amplified, while its weaknesses (unreliable memory and implicit assumptions) are constrained.

## Lessons Learned Building Knowledge Systems

The biggest lesson from this project is that reliability doesn't come from the model alone, it comes from the system around it. Once knowledge is externalized into structured memory and access to that memory is controlled, the behavior of the model becomes more predictable.

The second lesson is that retrieval is not just a utility step, it's part of reasoning. Answering a question from documents is a navigation task: deciding where to look, pulling in context, and working with that information. Control flow around retrieval turns a static pipeline into an agent-like process.

Finally, intelligence often lives in the boundaries between components. The LLM brings flexible language reasoning. The vector database provides structured memory. The system logic enforces grounding. None of these pieces is sufficient on its own, but together they create behavior that feels more capable and reliable.

## Limitations

### Retrieval quality depends on chunking and TOC quality

The system assumes that the textbook’s table of contents is well-structured and that paragraph-based chunks of ~300 tokens are semantically coherent. If the TOC is noisy or the PDF text extraction breaks paragraphs strangely, relevant content can land in awkward or fragmented chunks, which hurts retrieval and answer quality.

### Local vector search is still approximate

Even with a good embedding model and FAISS index, similarity search is approximate. Subtle conceptual questions, or questions that span multiple chapters in non-obvious ways, can still retrieve incomplete or slightly off-target chunks. The self-evaluation step can catch some of this, but it can’t fix fundamentally bad retrieval.

### Latency and cost from multiple LLM calls

Each question triggers several LLM calls: one to choose chapters, one to synthesize an answer, and one to self-evaluate it. That’s fine for a personal assistant or a classroom tool, but it means the system is slower and more expensive than a single-shot "just ask GPT" interaction, especially as the number of textbooks or concurrent users grows.

### Grounding is enforced by prompts, not hard constraints

The model is instructed to answer only from the retrieved context, and in practice this works well, but it’s still a probabilistic guarantee. Under some prompts or edge cases, the model can still hallucinate or blend in prior training. The system reduces this risk but doesn’t eliminate it.

### Currently limited to a single textbook and domain

This implementation is tuned to one statistics textbook and a single embedding space. Extending it to multiple books, editions, or domains raises additional challenges: cross-book disambiguation, conflicting explanations, and larger indexes that increase retrieval complexity.

## Conclusion

Building this system changed how I think about LLMs. Their strength isn't in knowing everything. It's in reasoning over information when they're placed inside a structure that manages knowledge explicitly. When we stop treating the model as a standalone brain and start treating it as part of a larger knowledge system, the behavior shifts from guessing to grounded reasoning.

In the end, the intelligence of the system doesn't live in the model or the database alone. It emerges from how structured memory and probabilistic reasoning are combined.

---

## Implementation at a Glance

| Layer | Technology |
|-------|------------|
| PDF parsing | PyMuPDF (fitz), table-of-contents extraction |
| Chunking | Paragraph-based, ~300 tokens, chapter metadata preserved |
| Embeddings | SentenceTransformer `all-MiniLM-L6-v2` |
| Vector index | FAISS `IndexFlatL2` |
| LLM | GPT-4o-mini (OpenAI API) |
| Chapter filtering | LLM selects from TOC -> filter before vector search |

The chunk schema: `{ chapter_title, page_num, chunk_text }`. Retrieval returns the top-k by L2 distance, with optional `chapter_keywords` to scope the search. The `TextbookAgent` class orchestrates the full loop: `choose_chapters` → `query` → `synthesize_answer` → `evaluate_answer`.
