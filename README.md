# Building a Smart PDF Chatbot

This project was built as part of a workshop to explore how large language models (LLMs) can be used to answer questions grounded in a specific document. The document used throughout is the Singapore Budget Statement 2026, delivered by PM Lawrence Wong in Parliament on 12 February 2026.

The project is structured in three phases, each building on the last.

---

## Phase 1: RAG Pipeline

The first phase is about setting up a basic Retrieval-Augmented Generation (RAG) pipeline from scratch.

The PDF is loaded and split into smaller overlapping chunks using LangChain's `RecursiveCharacterTextSplitter`. Each chunk is then converted into a numerical vector using the `all-MiniLM-L6-v2` embedding model from HuggingFace. These vectors are stored in a FAISS index, which allows fast similarity search at query time.

When a user asks a question, the system retrieves the most relevant chunks from the FAISS index and passes them, along with the question, into a prompt sent to an LLM via the HuggingFace Inference API. The LLM (Qwen2.5-7B-Instruct by default) then generates an answer based only on the provided context.

The pipeline goes: Load PDF, chunk text, embed chunks, store in FAISS, retrieve relevant chunks, format a prompt, and generate a response.

---

## Phase 2: Source Attribution

Phase 2 upgrades the pipeline to track where each piece of information came from.

Each chunk is enriched with citation metadata during the indexing step. This includes a unique citation ID (e.g. `c3.0.0`), the page number, a paragraph ID, an estimated or extracted bounding box from the PDF, and a short text preview. The prompt is also updated to include citation guidelines that instruct the LLM to tag claims inline using citation IDs in square brackets, like `[c3.0.0]`.

After the LLM generates a response, a post-processing step extracts the citation markers using regex, links them back to the source chunks, and formats the output to show a structured sources list below the answer.

---

## Phase 3: LLM-as-a-Judge

Phase 3 adds a verification layer on top of the cited responses from Phase 2.

The core problem is that the LLM in Phase 2 might cite a source that does not actually support the claim it is making. To address this, a second LLM call is made to act as a fact-checker.

The response is first split into individual sentences. Each sentence is paired with its cited source text and sent to the evaluator LLM in a single batched call. The evaluator returns a JSON array where each claim is assigned a support level (supported, partial, unsupported, or contradicted) along with a confidence score between 0 and 1 and a one-sentence reasoning.

Claims without any citations are labelled as uncited. The overall confidence of a response is the average score across all cited claims, with uncited sentences excluded from the calculation.