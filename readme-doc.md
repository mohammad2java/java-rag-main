# markets — Spring AI RAG Demo (Documented)

A complete, runnable demonstration of **Retrieval Augmented Generation (RAG)** built with Spring Boot 3.3.x and Spring AI. The application ingests a PDF document, embeds its paragraphs into a PostgreSQL/PGVector database, and exposes a REST endpoint that augments LLM responses with relevant document context.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Project Structure](#project-structure)
5. [Dependencies](#dependencies)
6. [Configuration](#configuration)
7. [Key Components](#key-components)
   - [Application](#application)
   - [IngestionService](#ingestionservice)
   - [ChatController](#chatcontroller)
8. [Docker / PostgreSQL Setup](#docker--postgresql-setup)
9. [Running the Application](#running-the-application)
10. [Making Queries](#making-queries)
11. [Troubleshooting](#troubleshooting)
12. [Best Practices](#best-practices)
13. [License](#license)

---

## Project Overview

The `markets` project is a minimal RAG pipeline:

1. **Ingest** — a PDF is read, split into text chunks, and each chunk is embedded and stored in PGVector.
2. **Retrieve** — when a user asks a question, relevant chunks are fetched by similarity search.
3. **Generate** — the retrieved chunks are injected as context into an LLM prompt; the LLM returns an answer grounded in the document.

The demo uses a single PDF (`article_thebeatoct2024.pdf`) and exposes one endpoint (`GET /`) that answers a fixed question about the document.

---

## Architecture

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
│  PDF file    │────▶│ IngestionService │────▶│ PGVector     │
│  (classpath) │     │ (CommandLineRunner)│    │ (PostgreSQL) │
└─────────────┘     └──────────────────┘     └──────┬───────┘
                                                     │
                                                     ▼
┌─────────────┐     ┌──────────────────┐     ┌───────────────┐
│  HTTP Client │────▶│  ChatController  │────▶│  LLM (OpenAI) │
│  (curl/browser)│   │  (REST endpoint) │     │  (GPT model)  │
└─────────────┘     └──────────────────┘     └───────────────┘
```

**Flow:**
1. On startup, `IngestionService` reads the PDF, splits it into paragraphs, embeds each paragraph via the configured embedding model, and stores the vectors in PGVector.
2. When a request hits `GET /`, `ChatController` uses `ChatClient` with a `QuestionAnswerAdvisor` to perform similarity search against PGVector, inject the top results into the prompt, and send it to the LLM.
3. The LLM returns a response grounded in the retrieved document context.

---

## Prerequisites

| Requirement | Version / Notes |
|---|---|
| Java JDK | 21 (or higher) |
| Maven | 3.9+ (or use the bundled `mvnw`) |
| Docker Desktop | Required for PostgreSQL + PGVector |
| OpenAI API Key | Required for embeddings and chat completions |
| OS | Windows / macOS / Linux |

---

## Project Structure

```
java-rag-main/
├── compose.yaml                          # Docker Compose for PostgreSQL/PGVector
├── pom.xml                               # Maven project & dependencies
├── README.md                             # Original project README
├── readme-doc.md                         # This comprehensive documentation
├── src/
│   ├── main/
│   │   ├── java/dev/danvega/markets/
│   │   │   ├── Application.java          # Spring Boot entry point
│   │   │   ├── ChatController.java       # REST endpoint for RAG queries
│   │   │   └── IngestionService.java     # PDF ingestion & vector store population
│   │   └── resources/
│   │       ├── application.properties    # Spring Boot configuration
│   │       └── docs/
│   │           └── article_thebeatoct2024.pdf  # Source document
│   └── test/
│       └── java/dev/danvega/markets/
│           └── MarketsApplicationTests.java
└── target/                               # Build output
```

---

## Dependencies

All dependencies are declared in `pom.xml`:

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring Boot JDBC (for PostgreSQL connection) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>

    <!-- Spring AI OpenAI integration (chat + embeddings) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>

    <!-- Spring AI PDF document reader -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pdf-document-reader</artifactId>
    </dependency>

    <!-- Spring AI PGVector vector store -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
    </dependency>

    <!-- Spring Boot Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Docker Compose support (runtime, optional) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-docker-compose</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-spring-boot-docker-compose</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>
</dependencies>
```

**BOM (Bill of Materials):** Spring AI `1.0.0-M3` (milestone release).

---

## Configuration

All configuration lives in `src/main/resources/application.properties`.

### Current configuration

```properties
# Application name
spring.application.name=markets

# OpenAI / OpenRouter API key — read from environment variable at runtime
# Set OPENAI_API_KEY in your shell before starting the app
spring.ai.openai.api-key=${OPENAI_API_KEY}

# Chat model used for generating responses
spring.ai.openai.chat.options.model=inclusionai/ling-3.0-flash:free

# Base URL for the OpenAI-compatible API endpoint
# OpenRouter: https://openrouter.ai/api
# OpenAI official: leave empty or use https://api.openai.com
spring.ai.openai.base-url=https://openrouter.ai/api

# Embedding model used to convert text chunks into vectors
# text-embedding-3-small → 1536 dimensions
# text-embedding-3-large → 3072 dimensions
spring.ai.openai.embeddings.model=text-embedding-3-small

# PGVector vector store configuration
spring.ai.vectorstore.pgvector.initialize-schema=true
spring.ai.vectorstore.pgvector.embedding-dimensions=1536

# Logging
logging.level.org.apache.pdfbox.pdmodel.font.FileSystemFontProvider=ERROR
logging.level.org.springframework.ai=DEBUG
logging.level.org.springframework.web.client=DEBUG

# Docker Compose lifecycle
spring.docker.compose.lifecycle-management=start_only
```

### Configuration properties explained

| Property | Purpose | Notes |
|---|---|---|
| `spring.ai.openai.api-key` | API key for the LLM/embedding provider | Use `${OPENAI_API_KEY}` to read from env; never hard-code secrets |
| `spring.ai.openai.chat.options.model` | Model for chat completions | Provider-specific model name (e.g., `gpt-4`, `inclusionai/ling-3.0-flash:free`) |
| `spring.ai.openai.base-url` | Base URL for the OpenAI-compatible API | Do **not** include `/v1` — the client appends it |
| `spring.ai.openai.embeddings.model` | Model for generating embeddings | Must be supported by the provider; `text-embedding-3-small` is 1536 dims |
| `spring.ai.vectorstore.pgvector.initialize-schema` | Auto-create PGVector schema on startup | Set to `true` for first run |
| `spring.ai.vectorstore.pgvector.embedding-dimensions` | Vector dimension count | Must match the embedding model (1536 for `text-embedding-3-small`) |
| `spring.docker.compose.lifecycle-management` | Docker Compose lifecycle mode | `start_only` keeps the DB running after app stops |

---

## Key Components

### Application

**File:** `src/main/java/dev/danvega/markets/Application.java`

The Spring Boot entry point. A standard `@SpringBootApplication` class with a `main` method.

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### IngestionService

**File:** `src/main/java/dev/danvega/markets/IngestionService.java`

A `CommandLineRunner` that executes on application startup. It:

1. Reads the PDF from `classpath:/docs/article_thebeatoct2024.pdf`.
2. Splits the PDF content into paragraphs using `ParagraphPdfDocumentReader`.
3. Further splits paragraphs into tokens using `TokenTextSplitter`.
4. Stores the resulting text chunks as embeddings in PGVector via `VectorStore.accept(...)`.

```java
@Component
public class IngestionService implements CommandLineRunner {

    private static final Logger log = LoggerFactory.getLogger(IngestionService.class);
    private final VectorStore vectorStore;

    @Value("classpath:/docs/article_thebeatoct2024.pdf")
    private Resource marketPDF;

    public IngestionService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    @Override
    public void run(String... args) throws Exception {
        var pdfReader = new ParagraphPdfDocumentReader(marketPDF);
        TextSplitter textSplitter = new TokenTextSplitter();
        vectorStore.accept(textSplitter.apply(pdfReader.get()));
        log.info("VectorStore Loaded with data!");
    }
}
```

**Important:** The `VectorStore.accept(...)` call triggers an embedding request to the configured embedding model. If this fails (e.g., due to a bad API key, wrong base URL, or unsupported model), the application will not start correctly. See [Troubleshooting](#troubleshooting).

### ChatController

**File:** `src/main/java/dev/danvega/markets/ChatController.java`

A `@RestController` that exposes a single `GET /` endpoint. It uses Spring AI's `ChatClient` with a `QuestionAnswerAdvisor` to:

1. Perform a similarity search in PGVector for the user's question.
2. Inject the top matching document chunks as context.
3. Send the augmented prompt to the LLM.
4. Return the LLM's response as a plain string.

```java
@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder
                .defaultAdvisors(new QuestionAnswerAdvisor(vectorStore))
                .build();
    }

    @GetMapping("/")
    public String chat() {
        return chatClient.prompt()
                .user("How did the Federal Reserve's recent interest rate cut impact various asset classes according to the analysis")
                .call()
                .content();
    }
}
```

---

## Docker / PostgreSQL Setup

The project uses Docker Compose to run a PostgreSQL instance with the PGVector extension.

**File:** `compose.yaml`

```yaml
services:
  pgvector:
    image: 'pgvector/pgvector:pg16'
    environment:
      - 'POSTGRES_DB=markets'
      - 'POSTGRES_PASSWORD=password'
      - 'POSTGRES_USER=user'
    labels:
      - "org.springframework.boot.service-connection=postgres"
    ports:
      - '5432:5432'
```

Spring Boot's Docker Compose support will automatically start this container when the application runs (thanks to `spring.docker.compose.lifecycle-management=start_only`).

**Connection details:**
- Host: `localhost` (or Docker host)
- Port: `5432`
- Database: `markets`
- User: `user`
- Password: `password`

These are auto-configured by Spring Boot's service connection mechanism — no additional JDBC URL is needed in `application.properties` when Docker Compose is active.

---

## Running the Application

### 1. Start Docker Desktop

Ensure Docker Desktop is running so that the PostgreSQL/PGVector container can be started.

### 2. Set your API key

```bash
export OPENAI_API_KEY="sk-your-openai-or-openrouter-api-key"
```

On Windows (PowerShell):
```powershell
$env:OPENAI_API_KEY = "sk-your-openai-or-openrouter-api-key"
```

On Windows (cmd):
```cmd
set OPENAI_API_KEY=sk-your-openai-or-openrouter-api-key
```

### 3. Run the application

Using the Maven wrapper:
```bash
./mvnw spring-boot:run
```

Or build and run the JAR:
```bash
./mvnw package
java -jar target/markets-0.0.1-SNAPSHOT.jar
```

### 4. Verify startup

On successful startup you should see:
```
HikariPool-1 - Start completed.
VectorStore Loaded with data!
```

The application will be available at `http://localhost:8080`.

---

## Making Queries

### Using curl

```bash
curl http://localhost:8080/
```

The response will be a text answer from the LLM, grounded in the content of the ingested PDF document.

### Using a browser

Open `http://localhost:8080/` in your browser.

### Customizing the query

Edit the `ChatController` to change the prompt or add a query parameter:

```java
@GetMapping("/")
public String chat(@RequestParam(defaultValue = "What does the article say about interest rates?") String question) {
    return chatClient.prompt()
            .user(question)
            .call()
            .content();
}
```

Then query with:
```bash
curl "http://localhost:8080/?question=What%20is%20the%20main%20topic?"
```

---

## Troubleshooting

### 1. Embedding call fails with 404 HTML response

**Symptom:** The application fails to start with an error like:
```
404 - <!DOCTYPE html><html ...>
```

**Cause:** The base URL is incorrect or has a duplicate `/v1` path segment. The Spring AI OpenAI client appends `/v1/embeddings` to the configured `base-url`. If your `base-url` already ends with `/v1`, the resulting URL becomes `/v1/v1/embeddings`, which returns a 404 HTML page.

**Fix:** Ensure `spring.ai.openai.base-url` does **not** include `/v1`:
```properties
# Correct — client appends /v1/embeddings
spring.ai.openai.base-url=https://openrouter.ai/api

# Wrong — causes double /v1/v1/embeddings (do NOT include /v1 in base-url)
# spring.ai.openai.base-url=https://openrouter.ai/api/v1
```

### 2. "cannot retry due to server authentication, in streaming mode"

**Symptom:** After a 404 or auth challenge, the HTTP client throws:
```
java.net.HttpRetryException: cannot retry due to server authentication, in streaming mode
```

**Cause:** The server returned an authentication challenge (401/403) while the request body was being streamed. Java's `HttpURLConnection` cannot retry streaming requests after an auth challenge.

**Fixes:**
- Ensure `OPENAI_API_KEY` is set correctly in the environment.
- Ensure the API key is valid and not expired.
- Ensure the `base-url` points to the correct provider endpoint.
- If using OpenRouter, verify your API key has access to the embedding model.

### 3. "Failed to obtain the embedding dimensions ... fall backs to default:1536"

**Symptom:** A WARN log from `PgVectorStore` about falling back to default embedding dimensions.

**Cause:** The embedding model call failed (auth, network, or unsupported model), so the application could not query the model for its output dimension count.

**Fix:** Explicitly set the embedding dimensions to match your model:
```properties
spring.ai.openai.embeddings.model=text-embedding-3-small
spring.ai.vectorstore.pgvector.embedding-dimensions=1536
```

### 4. Docker container won't start

**Symptom:** `HikariPool` cannot connect to PostgreSQL.

**Fixes:**
- Ensure Docker Desktop is running.
- Ensure port `5432` is not already in use: `docker ps` should not show another container on that port.
- Check `docker-compose logs` for errors.

### 5. PDF not found

**Symptom:** `IngestionService` fails with a file-not-found error for `article_thebeatoct2024.pdf`.

**Fix:** Ensure the PDF exists at `src/main/resources/docs/article_thebeatoct2024.pdf`.

### 6. Debug logging

The current `application.properties` includes debug logging for Spring AI and Spring Web Client. To reduce log noise after troubleshooting, remove or change these lines:
```properties
logging.level.org.springframework.ai=INFO
logging.level.org.springframework.web.client=INFO
```

---

## Best Practices

### Document Ingestion
- Add checks before reinitializing the vector store (e.g., check if data already exists).
- Use scheduled tasks or triggers for document updates instead of re-ingesting on every startup.
- Implement proper error handling for document processing failures.

### Query Optimization
- Monitor token usage to control costs.
- Implement rate limiting on the REST endpoint.
- Cache frequently requested results.

### Security
- Never hard-code API keys in source files. Use environment variables or a secrets manager.
- Secure API endpoints with authentication (e.g., Spring Security).
- Protect sensitive document content from unauthorized access.

### Production Considerations
- Use a connection pool (HikariCP is included by default with Spring Boot JDBC).
- Configure proper logging levels (avoid DEBUG in production).
- Use a managed PostgreSQL instance with PGVector for production workloads.
- Consider using a dedicated embedding model endpoint separate from the chat endpoint.

---

## Testing

Run the existing test suite:
```bash
./mvnw test
```

The project includes a basic Spring Boot test class at `src/test/java/dev/danvega/markets/MarketsApplicationTests.java`.

---

## Build

```bash
# Compile
./mvnw compile

# Package (skip tests)
./mvnw package -DskipTests

# Run the packaged JAR
java -jar target/markets-0.0.1-SNAPSHOT.jar
```

---

## License

This project is provided as a demo/example. See the original README for any license information.
