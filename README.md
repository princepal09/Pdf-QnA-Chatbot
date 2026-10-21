# PDF Q&A Chatbot

A Python notebook that answers questions about a PDF using retrieval-augmented generation (RAG). It extracts PDF text, creates embeddings with a local Ollama model, stores them in Chroma, and sends relevant passages to a Groq-hosted language model to generate an answer.

The notebook currently uses an Apache Kafka guide as its source document. You can adapt it to another PDF by changing the loader path and rebuilding the document index.

## Features

- Extract text and page metadata with `PyMuPDFLoader`.
- Split documents into overlapping chunks of 1,000 characters with a 200-character overlap.
- Generate embeddings through Ollama.
- Persist embeddings in a local Chroma collection and reuse an existing collection on subsequent runs.
- Retrieve the three most similar chunks for each question.
- Generate answers with Groq at a temperature of `0`.
- Inspect retrieved passages and PDF page numbers using sample evaluation questions.
- Prompt the model to use only the supplied context and acknowledge insufficient evidence.

## How it works

```text
PDF → Text extraction → Text chunks → Ollama embeddings → Chroma
                                                            ↓
Question → Similarity search → Top 3 chunks → Prompt → Groq → Answer
```

PDF processing, embeddings, and vector storage run locally. The question and retrieved text passages are sent to Groq for answer generation.

## Project structure

```text
Pdf-QnA-Chatbot/
├── pdf_qna_chatbot.ipynb   # Document indexing, retrieval, and question answering
├── requirements.txt       # Python dependencies
├── Readme.md              # Project documentation
├── .gitignore             # Excludes local configuration, PDFs, and vector data
├── .env                   # Local model settings and API key (not committed)
└── chroma_db/             # Persistent vector database (created at runtime)
```

The default source file, `Apache Kafka Beginner to Advanced Guide.pdf`, is excluded from Git. Supply it yourself or configure another PDF before running the notebook.

## Prerequisites

- Python; the notebook was authored with Python **3.12.1**.
- A notebook editor such as VS Code with Python and Jupyter support, or JupyterLab.
- Ollama installed and running locally, with an embedding model downloaded.
- A Groq API key and access to a supported chat model.
- Internet access to install dependencies, download the embedding model, and call Groq.

## Setup

Run the following commands from the project directory.

### 1. Create a virtual environment

Windows PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

macOS / Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

### 2. Prepare an Ollama embedding model

Choose an Ollama model that supports embeddings, then replace `YOUR_EMBEDDING_MODEL` with its model name:

```bash
ollama pull YOUR_EMBEDDING_MODEL
```

Keep Ollama running while indexing documents or asking questions. If the Ollama service is not already running, start it with `ollama serve` in a separate terminal.

### 3. Configure environment variables

Create a `.env` file in the project root:

```dotenv
GROQ_API_KEY=your_groq_api_key
GROQ_MODEL=your_groq_chat_model
OLLAMA_EMBEDDING_MODEL=your_ollama_embedding_model
```

Replace all placeholder values. `OLLAMA_EMBEDDING_MODEL` must match the model downloaded in the previous step. The notebook loads this file with `load_dotenv()`; `ChatGroq` reads `GROQ_API_KEY` from the environment.

Keep your API key private. The repository's `.gitignore` excludes `.env` files.

### 4. Provide a PDF

Place your PDF in the project directory and update the document-loading cell if needed:

```python
loader = PyMuPDFLoader(file_path="your-document.pdf")
documents = loader.load()
```

### 5. Open and run the notebook

Open `pdf_qna_chatbot.ipynb` and select the Python interpreter from `.venv` as its kernel. Run the cells in order from the project directory so that relative PDF and database paths resolve correctly.

If you prefer JupyterLab, install and launch it separately; it is not included in `requirements.txt`:

```bash
python -m pip install jupyterlab
python -m jupyterlab
```

The first run embeds the document chunks and saves them to `./chroma_db`. Later runs reuse the collection if it contains any records, although the preceding PDF-loading and splitting cells still run.

## Ask a question

The final cell requests a question through `input()` and prints the answer:

```python
question = input()
answer = rag_chain.invoke(question)
print(answer)
```

Example questions for the Kafka guide:

- What is Apache Kafka?
- What is a Kafka consumer group?
- What is the difference between a topic and a partition?
- How do partitions and consumer groups work together?

After initializing the chain, you can also ask questions directly:

```python
answer = rag_chain.invoke("What is Apache Kafka?")
print(answer)
```

## Configuration

| Setting | Current value | Where to change it |
| --- | --- | --- |
| Source PDF | `Apache Kafka Beginner to Advanced Guide.pdf` | `PyMuPDFLoader(file_path=...)` |
| Chunk size | `1000` characters | `RecursiveCharacterTextSplitter` |
| Chunk overlap | `200` characters | `RecursiveCharacterTextSplitter` |
| Embedding model | Environment variable | `OLLAMA_EMBEDDING_MODEL` in `.env` |
| Vector database directory | `./chroma_db` | `persist_directory` |
| Collection name | `kafka_pdf` | `collection_name` |
| Retrieved chunks | `3` | Retriever `search_kwargs["k"]` |
| Answer model | Environment variable | `GROQ_MODEL` in `.env` |
| Generation temperature | `0` | `ChatGroq(...)` |

**Changing the PDF or embedding model requires a fresh index.** The notebook only checks whether the collection has records; it does not detect source-document or model changes. Use a new collection name for a different PDF. If changing the embedding model, use a new collection or database directory and rerun the indexing cells with that model.

## Retrieval evaluation

The notebook includes five sample cases covering definitions, specific concepts, comparisons, multiple passages, and a question outside the document's scope. The evaluation loop prints the top retrieved passages and their one-based PDF page numbers for manual review.

Fill in `expected_answer` and `expected_pages` after reviewing your source PDF. These fields are placeholders: the loop does not score retrieval, compare answers, or run the generation chain. `should_answer` is displayed for reference only.

## Current limitations

- The interface is a Jupyter notebook with single-question input. There is no web UI, HTTP API, or conversation memory.
- The prompt requests `evidence_found`, `answer`, and `cited_chunk_ids`, but the chain uses `StrOutputParser`, so it returns text without enforcing a structured response schema.
- Retrieved chunks are joined as plain text without chunk IDs or page metadata. Reliable chunk citations are not implemented; page numbers are available in the separate retrieval inspection loop.
- The prompt asks the model to respond with `I don't know based on the provided documents.` when evidence is insufficient. This behavior depends on the model and is not programmatically validated.
- There is no OCR step for scanned, image-only PDFs.
- Python dependencies are unpinned, so installations may resolve to different versions over time.

## Troubleshooting

| Problem | What to check |
| --- | --- |
| PDF cannot be found | Confirm the filename, loader path, and notebook working directory. |
| `Set OLLAMA_EMBEDDING_MODEL in your .env file.` | Add the model name to `.env`, then rerun the environment-loading and embedding cells. |
| Ollama connection or model error | Confirm Ollama is running and the configured embedding model has been downloaded. |
| Groq authentication or model error | Check `GROQ_API_KEY`, `GROQ_MODEL`, and your account's model access. |
| Answers refer to an earlier PDF | Select a fresh Chroma collection and rerun indexing for the intended PDF. |
| Embedding dimension mismatch | Create a fresh collection or database directory for the configured embedding model. |
| Little or no text is extracted | Check that the PDF contains selectable text; scanned pages may need OCR before ingestion. |
| Imports fail in the notebook | Select the `.venv` kernel and install `requirements.txt` in that environment. |
