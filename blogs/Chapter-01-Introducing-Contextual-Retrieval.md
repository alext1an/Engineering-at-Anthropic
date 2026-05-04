# Introducing Contextual Retrieval

**Published:** September 19, 2024
**Source:** https://www.anthropic.com/engineering/contextual-retrieval

## Overview

Anthropic has introduced Contextual Retrieval, a technique designed to improve how AI systems retrieve relevant information from knowledge bases. The method addresses a key limitation in traditional Retrieval-Augmented Generation (RAG) systems: the loss of context when documents are broken into smaller chunks for processing.

## The Problem with Traditional RAG

Standard RAG systems work by:

1. Breaking documents into smaller text chunks
2. Converting chunks into vector embeddings
3. Storing embeddings in a searchable database
4. Retrieving relevant chunks based on semantic similarity to user queries

However, this approach has a significant flaw. When chunks are isolated from their source documents, they often lack sufficient context, making retrieval less effective. For example, a financial statement excerpt stating "The company's revenue grew by 3% over the previous quarter" doesn't specify which company or time period without surrounding context.

## The Solution: Contextual Retrieval

Contextual Retrieval uses two complementary techniques:

**Contextual Embeddings:** Prepending chunk-specific explanatory context before embedding text, so the model understands: "This chunk is from an SEC filing on ACME corp's performance in Q2 2023..."

**Contextual BM25:** Adding context before creating keyword-matching indices, which improves exact phrase matching alongside semantic understanding.

## Implementation

Developers use Claude to automatically generate contextual descriptions for each chunk. A simple prompt instructs the model to provide "a short succinct context to situate this chunk within the overall document for the purposes of improving search retrieval." The resulting context, typically 50-100 tokens, is prepended before processing.

With prompt caching, contextualizing chunks costs approximately "$1.02 per million document tokens," making the approach economically viable.

## Performance Results

Testing across multiple knowledge domains showed substantial improvements:

- **Contextual Embeddings alone:** 35% reduction in retrieval failures
- **Combined Contextual Embeddings + BM25:** 49% reduction in failures
- **With reranking included:** 67% reduction in failures

These improvements directly enhance downstream task performance.

## Key Findings

Research demonstrated that optimal results come from combining multiple techniques:

1. Using both embeddings and BM25 outperforms embeddings alone
2. Gemini and Voyage embeddings showed superior performance
3. Retrieving 20 chunks proved more effective than 5 or 10
4. Contextual information consistently improved results across all embedding models
5. Reranking filters further optimize which chunks reach the final model

## Additional Considerations

Implementation requires attention to chunk boundaries, embedding model selection, domain-specific prompt customization, and balancing the number of chunks retrieved. The article emphasizes running evaluations on specific use cases to optimize performance.

## Resources

Developers can access implementation guidance through Anthropic's cookbook and documentation, including prompt caching techniques for cost efficiency.
