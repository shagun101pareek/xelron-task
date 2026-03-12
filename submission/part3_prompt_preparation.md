# Part 3: Prompt Preparation

## Selected PR: #1224 - Fix the potential duplicate embeddings in the RAG module

---

## 3.1.1 Repository Context

MetaGPT is a framework for building AI-driven systems using multi-agent collaboration. It enables developers to define agents with specific roles (engineer, product manager, designer) that work together to accomplish complex tasks like software development. Each agent can leverage large language models (LLMs) to make decisions, write code, and communicate with other agents.

The Retrieval-Augmented Generation (RAG) component within MetaGPT enhances agent capabilities by allowing them to access external knowledge sources. When an agent needs to answer a question or make a decision, it can search a vector database of documents, retrieve relevant information, and use that context to generate more accurate responses. This reduces hallucinations and grounds decisions in factual data.

Users of MetaGPT include AI researchers, software development teams, and organizations building intelligent automation systems. They use it to prototype and deploy autonomous agents that can handle document analysis, code generation, research synthesis, and other knowledge-intensive tasks. The typical workflow involves training agents on custom datasets and deploying them in production environments.

The RAG subsystem addresses a fundamental problem in AI systems: how to provide external context to language models without exceeding token limits or incurring excessive API costs. Vector databases efficiently store and retrieve semantically similar documents, but the process of converting text to embeddings (dense numerical representations) is computationally expensive and often charged per API call. When the same documents need to be retrieved by multiple agents or in multiple queries, unnecessarily recalculating embeddings wastes resources.

---

## 3.1.2 Pull Request Description

This PR addresses inefficient embedding generation in the RAG system. Previously, whenever a retriever was created or rebuilt, the system would generate embeddings for documents from scratch, even if those same documents had already been embedded in an earlier operation. This became especially problematic in scenarios where:

1. Multiple agents accessed the same knowledge base
2. The application was restarted and needed to rebuild the retrieval system
3. Users iteratively refined search parameters and regenerated retrievers

The previous behavior was straightforward but wasteful: each call to `get_retriever()` with a configuration would trigger the full embedding pipeline, converting all documents to vectors regardless of whether those vectors already existed somewhere in the system.

The new behavior implements intelligent caching and reuse. Before generating new embeddings, the system checks whether an index with embeddings already exists in the provided configuration. If it does, the system reuses those embeddings. If not, the system generates embeddings once and stores them in a way that allows future retrievers to discover and reuse them. This is accomplished through a decorator pattern that intercepts index creation requests and makes a decision based on the presence of cached indexes.

Additionally, the PR refactors the underlying architecture from a direct dependency on `VectorStoreIndex` to a transformation-based pattern. Instead of tightly coupling retriever logic to index creation, transformations are applied as composable steps. This architectural change provides flexibility for future enhancements and makes it easier to support different vector store backends (FAISS, Elasticsearch, Milvus, etc.) with consistent behavior.

The changes reduce API costs by eliminating redundant embedding calls, improve performance by avoiding unnecessary computation, and maintain backward compatibility with existing code.

---

## 3.1.3 Acceptance Criteria

1. **Single Embedding Generation**: When multiple retrievers are created with the same documents and configuration, the embedding model is called only once, not once per retriever. Verification: Monitor embedding API calls or token usage; should be constant regardless of retriever count.

2. **Index Detection and Reuse**: When a retriever configuration includes an existing index reference, the system detects this and uses the cached index instead of rebuilding. Verification: Pass a configuration with `index` field populated; confirm new index is not created via factory logs or database checks.

3. **Transformation Pipeline Functionality**: The refactored SimpleEngine using transformations produces identical retrieval results to the original VectorStoreIndex-based implementation. Verification: Run existing retrieval tests; all pass with zero failures.

4. **Error Handling for Missing Configuration**: When configuration is incomplete or missing required fields, the system returns `None` from `_val_from_config_or_kwargs()` instead of raising `KeyError`, allowing graceful degradation. Verification: Pass incomplete config; verify operation completes without exception.

5. **Cross-Vector-Store Consistency**: The caching pattern works uniformly across all supported vector stores (FAISS, Elasticsearch, Milvus). Each retriever type respects cached indexes when available. Verification: Create retrievers using FAISSRetrieverConfig, ElasticsearchRetrieverConfig, and MilvusRetrieverConfig with pre-built indexes; all should reuse embeddings.

6. **Backward Compatibility**: Existing code that doesn't populate the index field continues to work, generating new indexes as before. Verification: Run existing applications without modifications; functionality unchanged.

7. **Metadata Preservation**: When embeddings are reused through cached indexes, associated metadata (document source, creation time, custom fields) remains intact and searchable. Verification: Query retrieved documents; confirm metadata fields are populated correctly.

---

## 3.1.4 Edge Cases

**Edge Case 1: Configuration with Partial Index Information**
When a configuration specifies some index parameters (collection name, embedding model) but not others (stored index reference), the system must decide whether to reuse or rebuild. The system should attempt to find an existing index matching the parameters; if found, reuse it. If not found, create new index. This handles the scenario where a user provides the same configuration twice but expects incremental updates, not full rebuilds.

**Edge Case 2: Stale or Corrupted Index References**
A configuration may contain a reference to an index that no longer exists (deleted from vector database) or is corrupted. When the system attempts to reuse this index, the operation should fail gracefully with a clear error message indicating the index is unavailable, allowing the user to regenerate. Without this, the system would hang or crash silently.

**Edge Case 3: Dimension Mismatch**
Different embedding models produce vectors of different dimensionalities (OpenAI: 1536 dims, Ollama: 4096 dims). If a configuration tries to reuse an index created with one embedding model using a different embedding model, dimension mismatch occurs. The system should detect this incompatibility and either reject the configuration or generate a new index, not silently produce incorrect results.

**Edge Case 4: Concurrent Retriever Builds**
If two processes attempt to build retrievers simultaneously with the same configuration, a race condition could occur where both detect "no cached index" and both begin embedding. The system should handle this through locking or atomic operations, ensuring only one embedding operation proceeds while others wait for completion, preventing duplicate work.

**Edge Case 5: Memory Constraints with Large Indexes**
When reusing very large cached indexes (millions of vectors), loading them into memory could exhaust available resources. The system should gracefully handle out-of-memory scenarios, providing informative errors rather than crashing the application.

**Edge Case 6: Configuration with None/Empty Values**
A configuration object may have fields with None or empty string values where defaults should apply. The code must distinguish between "explicitly set to None" and "not provided," applying appropriate defaults without overwriting intentional None values.

---

## 3.1.5 Initial Prompt

You are implementing a refactoring of MetaGPT's RAG (Retrieval-Augmented Generation) system to eliminate duplicate embedding generation. Your task is to modify the retriever factory pattern to intelligently detect and reuse cached embeddings, reducing API costs and improving performance.

### Context
MetaGPT is an AI agent framework where multiple agents may access the same knowledge base. The RAG system converts documents to embeddings (dense vectors) for semantic search. Currently, every time a retriever is created, embeddings are regenerated from scratch, even if identical documents have already been embedded. This wastes API tokens and computation.

### Requirements

**Architectural Goal:**
Transform the SimpleEngine from direct VectorStoreIndex dependency to a transformation-based pipeline. This decouples index creation logic from retrieval logic, enabling flexible reuse patterns across all vector store backends (FAISS, Elasticsearch, Milvus, Chroma).

**Core Implementation Tasks:**

1. **Implement the Caching Decorator (`@get_or_build_index`)**
   - Create a decorator that intercepts index building requests
   - Before building, check if an index already exists in the configuration via `_extract_index()`
   - If index exists, return it immediately (REUSE path)
   - If not, execute the wrapped function to build new index (BUILD path)
   - Apply this decorator to `_build_faiss_index()`, `_build_elasticsearch_index()`, and `_build_milvus_index()` methods in the RetrieverFactory

2. **Refactor SimpleEngine Architecture**
   - Remove direct dependency on `VectorStoreIndex` class
   - Replace with transformation-based pattern using `_from_nodes()` method
   - Transformations should be composable pipeline steps that process nodes before index creation
   - Ensure transformation results are indistinguishable from original VectorStoreIndex behavior in retrieval operations

3. **Improve Configuration Handling**
   - Modify `ConfigBasedFactory._val_from_config_or_kwargs()` to return `None` instead of raising `KeyError` when keys are missing
   - Update error handling in `ConfigBasedFactory` with clearer error messages indicating which config fields caused issues
   - Ensure graceful degradation when optional configuration fields are absent

4. **Enhance RetrieverFactory Index Management**
   - Add `_extract_index()` method to detect existing indexes in configuration objects
   - Refactor `_build_default_index()`, `_build_faiss_index()`, and other build methods to support both fresh builds and reuse scenarios
   - Ensure factory routing consistently applies the caching pattern across all vector store types

5. **Add Docstrings and Exception Handling**
   - Document the RAGExample class methods with clear descriptions of inputs, outputs, and expected behavior
   - Add `@handle_exception` decorators to async methods in the RAG pipeline for robust error capture
   - Provide informative error messages when caching fails or configuration is invalid

### Acceptance Criteria to Verify
- Embedding models are called only once regardless of retriever count (verify via API token monitoring)
- Configurations with cached index references skip embedding generation entirely
- All vector store backends (FAISS, Elasticsearch, Milvus) respect cached indexes uniformly
- Existing tests pass without modification; new tests cover index reuse scenarios
- Missing configuration fields return None gracefully instead of raising exceptions
- Metadata associated with embeddings is preserved during reuse
- The system handles dimension mismatches, stale index references, and concurrent builds safely

### Edge Cases to Handle
1. Configuration specifies collection name but no stored index reference → attempt discovery; rebuild if not found
2. Index reference points to deleted/corrupted index → fail gracefully with clear error message
3. Different embedding models with different dimensions trying to reuse same index → detect incompatibility
4. Concurrent retriever builds with same configuration → use locking to prevent duplicate embeddings
5. Configuration fields with None values → distinguish from missing fields; apply appropriate defaults
6. Large cached indexes approaching memory limits → handle out-of-memory errors gracefully

### Testing Requirements
- Unit tests for `_extract_index()` method with various configuration formats
- Integration tests verifying embedding API is called exactly N times for M retrievers (N < M)
- Tests for each vector store backend confirming caching works uniformly
- Tests for error conditions: missing config, dimension mismatch, stale indexes
- Performance benchmarks showing reduced latency and API costs with caching enabled
- Backward compatibility tests confirming existing code without cache hints still functions

### Deliverables
1. Modified retriever factory files with caching decorator and dynamic index handling
2. Refactored SimpleEngine using transformation pattern
3. Updated RAGExample with enhanced docstrings
4. Comprehensive test suite covering caching scenarios and edge cases
5. Documentation explaining the new caching behavior and configuration options

### Notes
- Preserve backward compatibility; code without index hints should work exactly as before
- The refactoring enables future additions of new vector store backends with minimal changes
- Focus on making the caching logic deterministic and testable; avoid hidden state
- Consider both in-memory and persistent vector stores in your implementation