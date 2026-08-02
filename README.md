# Spring AI RAG Demo

A demonstration project showcasing Retrieval Augmented Generation (RAG) implementation using Spring AI and OpenAI's GPT models. This application enables intelligent document querying by combining the power of Large Language Models (LLMs) with local document context.

## Overview

This project demonstrates how to:
- Ingest PDF documents into a vector database
- Perform semantic searches using Spring AI
- Augment LLM responses with relevant document context
- Create an API endpoint for document-aware chat interactions

## Project Requirements

- Java 21
- Maven
- Docker Desktop
- OpenAI API Key (or OpenRouter API key)
- Dependencies: [Spring Initializer](https://start.spring.io/#!type=maven-project&language=java&platformVersion=3.3.4&packaging=jar&jvmVersion=21&groupId=dev.danvega&artifactId=markets&name=markets&description=Demo%20project%20for%20Spring%20Boot&packageName=dev.danvega.markets&dependencies=web,spring-ai-openai,spring-ai-pdf-document-reader,spring-ai-vectordb-pgvector,docker-compose)

## Dependencies

The project uses the following Spring Boot starters and dependencies:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pdf-document-reader</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

## Getting Started

1. Configure your environment variables:
```properties
OPENAI_API_KEY=your_api_key_here
```

2. Update `application.properties`:
```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=inclusionai/ling-3.0-flash:free
spring.ai.openai.base-url=https://openrouter.ai/api
spring.ai.openai.embeddings.model=text-embedding-3-small
spring.ai.vectorstore.pgvector.initialize-schema=true
```

3. Place your PDF documents in the `src/main/resources/docs` directory

## Running the Application

1. Start Docker Desktop

2. Launch the application:
```bash
./mvnw spring-boot:run
```

The application will:
- Start a PostgreSQL database with PGVector extension
- Initialize the vector store schema
- Ingest documents from the configured location
- Start a web server on port 8080

## Key Components

### IngestionService

The `IngestionService` handles document processing and vector store population:

```java
@Component
public class IngestionService implements CommandLineRunner {
    private final VectorStore vectorStore;
    
    @Value("classpath:/docs/your-document.pdf")
    private Resource marketPDF;
    
    @Override
    public void run(String... args) {
        var pdfReader = new ParagraphPdfDocumentReader(marketPDF);
        TextSplitter textSplitter = new TokenTextSplitter();
        vectorStore.accept(textSplitter.apply(pdfReader.get()));
    }
}
```

### ChatController

The `ChatController` provides the REST endpoint for querying documents:

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
                .user("Your question here")
                .call()
                .content();
    }
}
```

## Making Requests

Query the API using curl or your preferred HTTP client:

```bash
curl http://localhost:8080/
```

The response will include context from your documents along with the LLM's analysis.

## Architecture Highlights

- **Document Processing**: Uses Spring AI's PDF document reader to parse documents into manageable chunks
- **Vector Storage**: Utilizes PGVector for efficient similarity searches
- **Context Retrieval**: Automatically retrieves relevant document segments based on user queries
- **Response Generation**: Combines document context with GPT-4's capabilities for informed responses

## Best Practices

1. **Document Ingestion**
    - Consider implementing checks before reinitializing the vector store
    - Use scheduled tasks for document updates
    - Implement proper error handling for document processing

2. **Query Optimization**
    - Monitor token usage
    - Implement rate limiting
    - Cache frequently requested information

3. **Security**
    - Secure your API endpoints
    - Protect sensitive document content
    - Safely manage API keys
