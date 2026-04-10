# Spring Boot RAG Pipeline with Nvidia KIMI

A full-fledged Retrieval-Augmented Generation (RAG) pipeline built using Spring Boot and the Spring AI framework. This application allows you to seamlessly ingest PDF documents, automatically vectorize their text into local in-memory embeddings, and perform intelligent, context-aware Q&A against the PDF using the NVIDIA `moonshotai/kimi-k2-instruct` AI model.

---

## 🚀 Tech Stack

- **Java**: 17
- **Framework**: Spring Boot 3.4.0
- **AI Orchestration**: Spring AI (1.0.0-M6)
- **Vector Storage**: SimpleVectorStore (In-Memory Spring AI vector store, meaning your vectors don't persist on app crash)
- **Embeddings Model**: `spring-ai-transformers` (Local ONNX-based sentence transformers, completely free and offline)
- **Chat/LLM Model**: `moonshotai/kimi-k2-instruct` provided by NVIDIA NIM (`integrate.api.nvidia.com`)

---

## 📦 Key Dependencies

The project leverages Spring AI's modern ecosystem to avoid building custom chunking and tokenization logic.

1. **`spring-boot-starter-web`**: Exposes the REST APIs (`/api/rag/upload` and `/api/rag/ask`).
2. **`spring-ai-openai-spring-boot-starter`**: The OpenAI-compatible client used to securely establish a connection with NVIDIA's endpoints using the OpenAi API format.
3. **`spring-ai-pdf-document-reader`**: A specialized utility mapping Apache PDFBox to convert raw PDF bytes into Spring AI Document objects.
4. **`spring-ai-transformers-spring-boot-starter`**: Automatically boots a local NLP tokenizer/transformer when the app starts. This prevents you from needing to hit an external API to create your Vector Embeddings, saving latency and money. 

---

## 🧠 Internal Working: How the RAG Pipeline operates

The Retrieval-Augmented Generation pattern limits AI hallucinations by injecting factual context into the system prompt. Here is exactly how this project does it:

### Step 1: Document Ingestion & Chunking (`/upload`)
When you send a PDF via `POST /api/rag/upload`, the `RagService`:
1. Reads the raw bytes via `ByteArrayResource` and pipes it through Spring AI's `PagePdfDocumentReader`.
2. The raw text is passed to the **`TokenTextSplitter`**, which carefully splits paragraphs into smaller tokenized "Chunks". If you pass an entire book to the AI, it will hit token limits. Chunking breaks it down so we only fetch exactly the relevant paragraphs.
3. The chunks are passed to your **Local Embedding Model** (Transformer). It mathematically encodes the sentences into high-dimensional float vectors (Arrays of numbers that represent the meaning of the sentence).
4. These vectors are inserted into your completely in-memory **`SimpleVectorStore`**.

### Step 2: Semantic Similarity Search (`/ask`)
When you ask a question like "Summarize the conclusion" via `GET /api/rag/ask`, the `RagService`:
1. Takes your question string and converts it to a vector using the exact same local Embedding model.
2. Performs a **Mathematical Similarity Search** against the `SimpleVectorStore`. We are comparing the distance between your Question Vector and all your PDF Chunk Vectors to find the top `4` most closely related paragraphs.
3. We extract the raw text out of these top 4 similar chunks.

### Step 3: Prompt Augmentation & Answer Setup
1. A meticulously crafted `System Prompt` is assembled.
2. We inject the raw text of the 4 most relevant chunks into the `Context:` block of the System Prompt telling it exactly what facts it has available.
3. Using Spring AI's `ChatClient`, the combined payload (System Prompt + Your literal question) is sent via an encrypted HTTP request to the NVIDIA NIM endpoint.
4. NVIDIA's `moonshotai/kimi-k2-instruct` interprets our provided factual context and responds securely. This final answer is delivered right back to you as JSON.
---

## 🗄️ Why Does it "Forget" My PDF on Restart? (And the Production Solution)

Currently, the application uses Spring AI's `SimpleVectorStore`. As the name implies, this is an **in-memory** data structure. 
Every time you restart the application, the heap memory is cleared, meaning your generated Vector Embeddings are wiped out. In a local testing environment, this is great because it's fast and requires zero database setup. However, in a real-world scenario, you do not want to re-parse and re-embed thousands of PDFs every time the server restarts.

### The Production Solution: `pgvector`

In a production-grade application, you swap the `SimpleVectorStore` for a permanent database optimized for vector math. **PostgreSQL with the `pgvector` extension** is the industry standard for this within the Spring ecosystem.

**What is `pgvector`?**
It is an open-source extension for PostgreSQL that allows you to store your vector embeddings directly in the database alongside traditional relational data. 

**How it works internally:**
1. When you upload a PDF, the chunks are vectorized just like before.
2. Instead of saving them to a Java memory `List`, Spring AI sends an `INSERT` statement to your PostgreSQL database, saving the vector array into a special `vector` column type.
3. When a user asks a question, PostgreSQL uses specialized indexing algorithms (like HNSW - Hierarchical Navigable Small World) to perform lighting-fast Nearest-Neighbor vector math *at the database level*, returning the closest chunks. 
4. **Result**: Your vectors survive server restarts, and you can query millions of documents instantly!

### Alternative Production Vector Stores
If you don't want to use PostgreSQL, Spring AI supports several other brilliant alternatives for production storage:

1. **ChromaDB**: An open-source, AI-native database explicitly built for embeddings and RAG pipelines.
2. **Milvus / Zilliz**: Built from the ground up for massive-scale vector deployment, heavily utilized in enterprise AI.
3. **Qdrant**: A high-performance vector search engine written in Rust, available locally and via cloud.
4. **Pinecone**: A fully managed cloud Vector Database. You don't have to host anything; you just send your vectors over an API.
5. **Redis (RediSearch)**: If you're already using Redis for caching, Redis supports vector similarity search right inside the cache!

---

## 💻 Running the Application

Ensure you are located at the root of the project directory.

Run the Spring Boot application using Maven:
```bash
./mvnw clean spring-boot:run
```

Ensure the port `8080` is freely available.

---

## 🧪 Testing the API Endpoints

### 1. Upload the PDF
Use the following `curl` command to parse and store your PDF as local embeddings.

```bash
curl -X POST http://localhost:8080/api/rag/upload -F "file=@/path/to/your/pdf_file.pdf"
```
**Success Response:** 
```json
{
  "message": "PDF loaded and indexed successfully"
}
```

### 2. Ask a Question
Perform a semantic search and get a response from the Nvidia AI.

```bash
curl -G "http://localhost:8080/api/rag/ask" --data-urlencode "question=What are the primary findings in this report?"
```
**Expected Response:**
```json
{
  "answer": "Based on the provided PDF context, the primary findings state that..."
}
```
