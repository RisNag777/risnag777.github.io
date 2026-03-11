---
layout: post
title: "Designing an Agentic Textbook TA"
---

<div align="center">
  <img src="https://github.com/user-attachments/assets/e0e1bf89-2956-4a89-b389-96bf713655bf" 
       alt="User Input Question Interface" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%;">
</div>


Let's do a quick flashback to our college days. Picture it: You're cramming last minute after not touching your textbook (if you even had one) for the last 3 months. In that last minute rush, if you have any questions about the topics you're studying, your only real option is to call up that one smart friend at 2 in the morning, and pray that they are -
a. Awake
b. Attentive
c. Able to answer your questions in a way that you actually understand and not in a way that made you realize that there was no way that you would ever understand anything ever again and so you would have no choice but to close the book, give up, hope that someone famous would die tomorrow (leading the exam to get postponed) and go to sleep.

If it's unclear, I had a nightmare that played out exactly like this. In the dream, my favorite football team's star player died and, because it was a nightmare, the exam *still* wasn't postponed. Once I woke up and processed the trauma, I thought about the exam scenario. Nowadays, a student could just open up ChatGPT or Gemini and ask their question, but there's a catch. When the LLM has the entire internet to pull from, you're never 100% certain if the answer is "textbook-accurate". So then, what if the LLM specifically "learned" from your textbook? What if it gave you the answer and also was able to point to the exact chapter, page and paragraph so you could cross check it? That would've been quite the godsend for me back in college. That's what I set out to build.

<div align="center">
  <img src="https://github.com/user-attachments/assets/041b059d-132a-4b97-a620-91cd45ef67fe" 
       alt="User Input Question Interface" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%;">
  <p><i>The Interface.</i></p>
</div>

### The Pipeline: Parsing and Retrieval

I used [openstax.org](https://openstax.org/) to source free textbooks that I parsed using [PyMuPDF](https://pymupdf.readthedocs.io/en/latest/). Since passing an entire textbook into the LLM's context window is both expensive and inefficient, I broke the text into chunks capturing `chapter_title` and `page_num` in addition to `chunk_text`. I made use of a vector database (Faiss - Facebook AI Similarity Search) to pinpoint the most relevant chapters to further reduce the number of tokens being passed to the LLM.
> So, given a set of vectors, we can index them using Faiss — then using another vector (the query vector), we search for the most similar vectors within the index. - [Pinecone.io](https://www.pinecone.io/learn/series/faiss/faiss-tutorial/)

To turn my text into these vectors, I encoded the `chunk_text` using [`all-MiniLM-L6_v2`](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2) from HuggingFace's Sentence-Transformers library.
> It maps sentences & paragraphs to a 384 dimensional dense vector space and can be used for tasks like clustering or semantic search. - [HuggingFace](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2)

Now, the search space is ready. The textbook has been parsed into encoded chunks which have been fed to the vector database. When a user asks a question, it is similarly encoded and Faiss finds the top n most relevant chunks and presents them to the user. This is where the agent comes into play.

<div align="center">
  <img src="https://github.com/user-attachments/assets/5f42a736-dad9-42c4-bcce-a74748dafbd9" 
       alt="User Input Question Interface" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%;">
</div>

### Multi-Agent Orchestration

To keep the system fast and the logic transparent, I bypassed heavy agent frameworks in favor of direct orchestration using the OpenAI SDK. By building a custom state machine, I gained total control over the handoffs between three specialized agents:

<div align="center">
  <img src="https://github.com/user-attachments/assets/42c6e21d-8b83-45be-a571-fd4789d1c5da" 
       alt="User Input Question Interface" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%;">
</div>

1. The Librarian: Before searching the entire textbook, this agent analyzes the list of chapters against the user's question. It tells Faiss exactly which chapters to focus on, thereby improving search speed and reducing the search space.
2. The Scholar: Once Faiss retrieves the top n most relevant chunks from the chapters picked by the Librarian, the Scholar reads the raw text chunks and synthesizes a comprehensive answer.
3. The Auditor: This agent evaluates the Scholar's answer and confirms that it is **well-supported** by the source text.  
Finally, the app displays the verified answer alongside the top n relevant chunks from the textbook.

<div align="center">
  <img src="https://github.com/user-attachments/assets/7cffcd01-9ed6-49fc-932a-00ed95ac5166" 
       alt="Generated Answer and Textbook Chunks" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 90%;">
  <p><i>The Scholar’s synthesis.</i></p>
</div>

<div align="center">
  <img src="https://github.com/user-attachments/assets/4a98b657-1178-4c92-a174-2c7976022f1a" 
       alt="Textbook Chunk Metadata" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 90%;">
  <p><i>Raw textbook chunks retrieved by Faiss and verified by the Auditor.</i></p>
</div>

### What's Next?

The foundation is firmly set, and there is so much potential to build on this:
- Interactive Quizzing: An agent that generates mini-quizzes to ensure the user fully understands the concept
- Library-Scale Search: Expanding the "Librarian" agent to find relevant information across multiple books simultaneously
- Multi-Modal Support: Giving the agents the ability to read and interpret diagrams and charts

### The Takeaway

Building this agentic system taught me that the power of AI really lies in how we constrain it to be useful. By grounding an LLM in a specific textbook and using an agentic chain of command, we move from a potentially hallucinating chatbot to a reliable academic partner. Now, the next time a student is cramming at 2:00 AM, they don't have to pray for a miracle or wake up their friend, because they have a private, verified tutor ready to walk them through the syllabus, one page at a time
