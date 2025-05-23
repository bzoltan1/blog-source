---
title : "Building a Local Bugzilla RAG System"
subtitle : "A guide to building a local RAG system for Bugzilla data using Ollama and ChromaDB."
date : 2025-05-23T10:00:00+02:00
tags : ["LLM", "RAG", "Bugzilla", "Ollama", "ChromaDB", "openSUSE"]
type: post
---

My goal was to build a local database that could:

* Ingest my ~4GB Bugzilla database
* Answer questions or give advice on new bugs based on historical ones
* Run offline on my openSUSE Tumbleweed machine, which is equipped with 64GB RAM and an AMD Ryzen 7 PRO 7840U

Naturally, my first idea was to build a standalone LLM like GPT. But fine-tuning an LLM on custom data is resource-intensive—a massive understatement. When I started to fine-tune an LLM on my laptop, I let the process run for a full week, and it reached only 1%. Using cloud-based services or investing in powerful new hardware were not options. Also, the problem with standalone LLMs is that they may hallucinate or generate inaccurate information, especially on domain-specific topics. The other disadvantage of using LLMs is that they are static; once trained, they don't know anything that happened afterward.

After several spectacular failures and frustrating dead ends, I figured out that I better try out Retrieval-Augmented Generation (RAG). RAG is considered better than other approaches involving natural language generation because it combines the strengths of Pre-trained Language Models (for fluent generation and reasoning) and External Knowledge Retrieval (for up-to-date, grounded information).

RAG retrieves relevant documents from a knowledge base (database, vector store) and uses them to anchor responses in actual facts. As no retraining is needed, it is easier and cheaper to maintain domain-specific knowledge, which as a bonus, can be updated anytime.

I found these three articles interesting on the subject:

* [https://www.wired.com/story/reduce-ai-hallucinations-with-rag/](https://www.wired.com/story/reduce-ai-hallucinations-with-rag/)
* [https://blogs.nvidia.com/blog/what-is-retrieval-augmented-generation/](https://blogs.nvidia.com/blog/what-is/retrieval-augmented-generation/)
* [https://aws.amazon.com/what-is/retrieval-augmented-generation/](https://aws.amazon.com/what-is/retrieval-augmented-generation/)

## Downloading Bugzilla Records for Offline Use

I’ve written a small utility to download all Bugzilla records into a local database. This is especially useful for teams who need to analyze historical bug data without exposing their sensitive data to online services. After all, data protection does matter.

The script uses the Bugzilla REST API and supports full dataset extraction, including bugs, comments, and metadata. It’s designed to be robust, simple, and easy to customize for different Bugzilla setups.

You can find it here: [github.com/bzoltan1/download-bugzilla](https://github.com/bzoltan1/download-bugzilla)

It is important to note that this script may be extremely slow. Also, there is a reason why the API keys are an array. Some Bugzilla servers have limitations for API keys. For such servers, and if the Bugzilla database is huge, it is safer to have ten or even more keys.

`$ python ./download_bugzilla.py`

This command creates a `bug_reports.json` file with all the Bugzilla records.

## Convert Bugzilla records to Chroma Database

The next step is to index Bugzilla bug reports stored in local JSON files into a Chroma vector database for semantic search and retrieval.

Chroma is useful because it lets me search and organize text data based on meaning, not just keywords. In our case, it means I can search Bugzilla bug reports and find similar ones, even if they don't use the exact same words.

Instead of just indexing text like a traditional database, Chroma turns each bug report into a vector (a list of numbers that represent its meaning). This allows it to compare bugs by their content, which is helpful for finding duplicates, related issues, or patterns.

It's fast, works well with large datasets, and is easy to use in Python. It's a good choice for building a smarter search or analysis tool using Bugzilla data.

The process involves:

* Loading the `bug_reports.json` file.
* Flattening each bug into a plain text "document."
* Creating embeddings using `sentence-transformers`.
* Storing them in a local Chroma vector DB.

`$ pip install -U chromadb sentence-transformers langchain-community langchain-chroma langchain-huggingface langchain-ollama`
`$ python ./index_bugs_to_chroma.py`

The code is from here: [https://github.com/bzoltan1/download-bugzilla/blob/main/index_bugs_to_chroma.py](https://github.com/bzoltan1/download-bugzilla/blob/main/index_bugs_to_chroma.py)

## Run a Local LLM

Ollama is an easy solution with full models like Mistral, Llama3, or Phi. And Ollama is available on openSUSE Tumbleweed:

`$ sudo zypper in ollama`
`$ sudo systemctl enable ollama`
`$ sudo systemctl start ollama`

## Make a Simple Query Interface (RAG Retrieval + Generation)

Finally, I needed a simple script that lets me ask questions in plain English about my Bugzilla bug reports and returns smart, helpful answers.

Here is the code: [https://github.com/bzoltan1/download-bugzilla/blob/main/query_interface.py](https://github.com/bzoltan1/download-bugzilla/blob/main/query_interface.py)

`$ python3.13 query_interface.py`
`Ask a question about the Bugzilla data (Ctrl+C to exit):`

`Prompt:`

It works by finding the most relevant Bugzilla records using a vector search (based on meaning, not just keywords), then feeding those to a local language model (Mistral via Ollama) to generate a response.

The result is a Retrieval-Augmented Generation (RAG) system that gives answers grounded in your actual data, all running locally from the command line.
