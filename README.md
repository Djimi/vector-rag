# Vector RAG Project

## How to Run

1. Create a `.env` file and add your OpenAI API key:

     ```env
     OPENAI_API_KEY="<your_api_key>"
     ```

     You can also set the key as a system environment variable instead of using `.env`.

2. Run `RAG_with_TOC.ipynb`.

     This notebook prepares the Retrieval-Augmented Generation (RAG) pipeline, extracts text from the source PDF, chunks the content, and builds the vector database.

3. Run `FAISS_OPENAI.ipynb`.

     This notebook loads the prepared data, retrieves relevant chunks with FAISS, and uses the OpenAI API to generate responses.

4. Review the output in `FAISS_OPENAI.ipynb`.

## Why There Are Two Notebooks

`RAG_with_TOC.ipynb` is responsible for the ingestion and indexing pipeline. It handles the heavier preprocessing work, including extracting text from the PDF and storing the chunks in FAISS.

`FAISS_OPENAI.ipynb` is focused on inference. It retrieves relevant chunks from the vector store and sends the final context to the OpenAI API.

This separation keeps the main notebook cleaner and avoids mixing preprocessing logic with the user-facing query flow.

## High-Level Overview

### RAG Pipeline

The project uses the table of contents (TOC) to split the book into structured sections. The hierarchy has up to three levels:

- `main_section`
- `section`
- `subsection`

In practice, the structure can look like this:

```text
main_section -> section -> subsection
main_section
```

The text between the start of one subsection and the next section boundary is then chunked.

The chunking strategy is designed to preserve context:

- First split the text by subsection.
- Then split each subsection into sentences.
- Each chunk is limited to 15 sentences or 500 tokens, whichever comes first.
- Sentences longer than 500 tokens are removed. In this project, those are usually code-heavy blocks.
- Chunks shorter than 30 tokens are also removed because they are often low-value text such as "This page is intentionally left blank."

FAISS is used as the vector store, while the metadata is stored separately.

The metadata includes:

- `main_section`
- `section`
- `subsection`
- `page`
- `content`

The `content` field is stored mainly to make debugging and inspection easier.

Additional cleanup and filtering are applied during ingestion:

- "This page is intentionally left blank."
- Page headers such as the page number and chapter name
- Large code blocks that should not be processed
- Images, which are not processed
- The first few and last few sections, because they do not add meaningful search context or contain large code blocks
- Text extraction issues caused by special character combinations such as "fi", "ff", and "ffi"
- Font size, which is also used when detecting where a subsection begins

> Important: `RAG_with_TOC.ipynb` contains debugging output to make bug fixing easier.

### Serving User Requests

The project currently uses the OpenAI API with OpenAI-hosted models.

The request flow is:

1. The user sends a question.
2. The system extracts the semantic meaning of the question and determines the expected response type: text, image, or audio.
3. Redundant phrasing such as "I would like" is removed before semantic search so the vector lookup stays focused.
4. The refined query is used to search FAISS for the most relevant chunks.
5. The question is then sent to the OpenAI API with tool use enabled so the model can leverage the RAG pipeline.
6. Tool calls are executed and their results are returned to the model.
7. The final response is generated.

If the requested output is text, the process ends there. If audio is requested, the final text is converted to speech and saved. If an image is requested, the response is used for image generation, while the original user prompt is also preserved.

Preserving the original prompt matters because some details can be useful for image generation even if they were intentionally removed before semantic search. For example, a style preference such as "I would like a pink background" may be irrelevant for retrieval but still important for the final image prompt.

Text streaming is also implemented.



     





