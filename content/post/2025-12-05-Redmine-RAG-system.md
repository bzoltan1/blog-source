---
title : "Redmine RAG system"
subtitle : "It was so tempting to give it the title \"Oops, I did it again.\""
date : 2025-12-05T05:00:00+02:00
tags : ["Redmine", "LLM", "Chroma DB", "Ollama"]
type: post
---
### The Goal
The goal was to extract all issues from a Redmine instance, anonymize the data, and build a local RAG system for semantic search and Q&A.

Just like with the previous experiment with Bugzilla it started with data extraction. I planned to make a simple bulk download via Redmine API. Then came the first problem. Redmine's API doesn't return journal entries (comments) when using the project-level endpoint, even with the include=journals parameter. I tried out different ways but nothing worked. The solution was, after all, to change the strategy and fetch each issue individually via /issues/{id}.json. This was much slower but guaranteed complete data including all comments.

All I needed to be careful about was being graceful with the server, so I slept 2–5 seconds between API requests.

### Downloading the data
An important detail of this phase was that because I did not trust the stability of the API and my network, I wanted to build in a checkpoint system to track progress, retry logic in case something failed (3 attempts per issue), and save after every 50 issues to enable resume on failure. Here is the script: [download_redmine.py](https://github.com/bzoltan1/Redmine-RAG/blob/main/download_redmine.py)
It may not be so exciting to see, but this is how it looked in action:

```
~/Redmine-RAG $ python3.11 download_redmine.py
$ python3.11 download_redmine.py 
✓ Completed 2513 issues for 'virtualization'
✓ Completed 1920 issues for 'performance'
✓ Completed 583 issues for 'qesecurity'
✓ Completed 1662 issues for 'qe-kernel'
✓ Completed 561 issues for 'qam'
✓ Completed 2312 issues for 'qe-yast'
✓ Completed 12987 issues for 'openqatests'
✓ Completed 2150 issues for 'openqa-infrastructure'
✓ Completed 3300 issues for 'containers'

Total issues downloaded: 27988
✓ Merged 27988 issues into redmine_master_dataset.json
Next step: Preprocess and chunk 'redmine_master_dataset.json' for RAG.
```


I scheduled the downloading process for 2–3 days. During this time I encountered one massive DNS resolution failure. Around 700+ issues failed with “Failed to resolve '…'” errors. I do not know if it was caused by network instability or if my request rate was still too aggressive. Anyhow, thanks to the success/failure tracking in checkpoints and the retry/resume mechanism I successfully downloaded all 27,975 issues with complete journal data across all the projects I was interested in.[download_individual_redmine_issues.py](https://github.com/bzoltan1/Redmine-RAG/blob/main/download_individual_redmine_issues.py)

This is how it works on the second round when I downloaded the journals for those issues what failed on the first round:


```
~/Redmine-RAG $ python3.11 download_individual_redmine_issues.py

======================================================================
Redmine Journal Enrichment Script (with Retry Logic)
======================================================================

✓ Loaded 27979 issues from redmine_master_dataset.json
✓ Loaded checkpoint: 27966 successful, 13 failed out of 27979 total

⟳ Resuming from checkpoint
   - 27966 issues already successful
   - 13 issues need retry

======================================================================
Starting Journal Enrichment Process
======================================================================
Total issues to process: 27979
Already successful: 27966
To retry: 13
Fresh attempts needed: 0
Rate: 1 request every 1.0 seconds
Estimated time: ~0.2 minutes
======================================================================

[5662/27979] Retrying issue #129166 (attempt 2/3)... ✓ 10 journal(s)
[7094/27979] Retrying issue #59801 (attempt 2/3)... ○ No journals
[7385/27979] Retrying issue #62729 (attempt 2/3)... ✓ 11 journal(s)
[7386/27979] Retrying issue #62741 (attempt 2/3)... ✓ 8 journal(s)
[7387/27979] Retrying issue #63256 (attempt 2/3)... ✓ 3 journal(s)
[7388/27979] Retrying issue #63328 (attempt 2/3)... ✓ 5 journal(s)
[7389/27979] Retrying issue #63331 (attempt 2/3)... ✓ 7 journal(s)
[7390/27979] Retrying issue #63655 (attempt 2/3)... ✓ 6 journal(s)
[11549/27979] Retrying issue #29086 (attempt 2/3)... ✓ 5 journal(s)
[16060/27979] Retrying issue #64054 (attempt 2/3)... ✓ 3 journal(s)
[20770/27979] Retrying issue #133517 (attempt 2/3)... ✓ 2 journal(s)
[20771/27979] Retrying issue #133529 (attempt 2/3)... ✓ 7 journal(s)
[27700/27979] Retrying issue #187119 (attempt 2/3)... ✓ 2 journal(s)

--- Progress Checkpoint ---
Successful: 27979/27979 (100.0%)
With journals: 12 | Without: 1
Currently failing: 0 issues
Max retries exhausted: 0 issues
Elapsed: 0.3 min
---------------------------

✓ Enriched data saved to redmine_master_dataset_with_journals.json
✓ Checkpoint file removed (all issues processed successfully)

======================================================================
Journal Enrichment Run Complete!
======================================================================
Successfully enriched: 27979/27979 (100.0%)
  - With journals: 12
  - Without journals: 1
Still failing: 0 issues (will retry on next run)
Max retries exhausted: 0 issues (kept original data)
Run time: 0.3 minutes
======================================================================


======================================================================
Journal Coverage Analysis
======================================================================
Total issues: 27979
Issues with journals: 12 (0.0%)
Issues without journals: 27967 (100.0%)
Total journals fetched: 69
Average journals per issue: 0.00

Journal count distribution:
  0 journal(s): 27967 issues (100.0%)
  2 journal(s): 2 issues (0.0%)
  3 journal(s): 2 issues (0.0%)
  5 journal(s): 2 issues (0.0%)
  6 journal(s): 1 issues (0.0%)
  7 journal(s): 2 issues (0.0%)
  8 journal(s): 1 issues (0.0%)
  10 journal(s): 1 issues (0.0%)
  11 journal(s): 1 issues (0.0%)
======================================================================

✓ Run complete! Enriched data saved to: redmine_master_dataset_with_journals.json

```


### Data Anonymization
A funny detail is that whoever I talked to about my project felt the instant urge to remind me of the importance of data privacy. I felt like when I was ten years old and my grandma told me to wear warm clothes when it is cold outside. Naturally I considered the anonymizing step important. I made a simple script [redmine_master_dataset_anonymizer.py](https://github.com/bzoltan1/Redmine-RAG/blob/main/redmine_master_dataset_anonymizer.py)to remove all real usernames before building the RAG system. The script was anonymizing the Issue authors, Assigned users, Journal/comment authors, and Watchers. The rest of the data was safe to store, process, and play with.
The anonymized proof is here that I did it :)
```
~/Redmine-RAG $ python3.11 redmine_master_dataset_anonymizer.py

======================================================================
Redmine Dataset Anonymization
======================================================================

Loaded 27979 issues. Anonymizing user data...
  Processed 1000/27979 issues
  [...]
  Processed 27979/27979 issues

✓ Anonymized dataset saved: redmine_master_dataset_anonymized.json
✓ User mapping saved: user_anonymization_mapping.json

======================================================================
Anonymization Complete!
Total issues: 27979
Unique users anonymized: 266

======================================================================
Verification
======================================================================

Original issues: 27979
Anonymized issues: 27979

Sample Author:
  Original: XXX
  Anonymized: User_00XXX

✓ Anonymization verified successfully

Ready for RAG system!
Use the file: redmine_master_dataset_anonymized.json
```


### Built-in vs. External Embeddings in ChromaDB
When I started experimenting with Redmine issue retrieval, one of the first architectural choices was how to generate embeddings. Should I let ChromaDB handle embeddings internally (built-in embeddings), or embed everything ourselves before ingestion (e.g., using Ollama, nomic-embed-text, mxbai-embed-large, etc.)? Surprisingly, the difference turned out to be significant, both in control and in debugging effort.

## Without embeddings
In this case the script ingests the Redmine issues into ChromaDB without generating embeddings at ingest time. We store only the documents created from the JSON records directly and some metadata. Chroma then generates embeddings on demand at query time using its built-in default embedding model (usually all-MiniLM-L6-v2). This is a fairly simple solution that makes the ingestion fast. The downside is that semantic querying will be slower and less accurate. In short, it is good for quick testing and keyword-style search. It is still a smart solution but more like a glorious and expensive search engine.

## Embeddings with Ollama SDK
In this case we ingest the Redmine issues into ChromaDB with embeddings computed using Ollama before being stored. I tried various models starting with nomic-embed-text and also qwen3-embedding and mxbai, but I have not done enough testing to say anything credible about the differences. Using an Ollama embedding model will store not only the documents and the metadata but also the embeddings (vectors). This enables better semantic search quality. The ingestion is slow but the queries will be faster because vectors already exist. In short, embeddings with the Ollama SDK give full semantic ingestion with high-quality embeddings stored up front. This is ideal for production-level semantic search.

[ingest_to_chromadb-embed.py](https://github.com/bzoltan1/Redmine-RAG/blob/main/ingest_to_chromadb-embed.py)

```
$ python3.11 ingest_to_chromadb-embed-qwen3-embedding.py

======================================================================
Redmine -> ChromaDB ingestion WITH embeddings (Ollama SDK)
======================================================================

Loading redmine_master_dataset_anonymized.json...
✓ Loaded 27975 issues

Initializing embedding model: nomic-embed-text (Ollama SDK)
✓ Embedding function ready

Initializing ChromaDB at: ./chroma_db
✓ Created collection 'redmine_issues' with embeddings

Processing issues (batch size: 50)...

  ✓ Progress: 50/27975 (0.2%)
  ✓ Progress: 100/27975 (0.4%)
  [...]
  ✓ Progress: 27950/27975 (99.9%)
  ✓ Progress: 27975/27975 (100.0%)

======================================================================
Ingestion complete with embeddings
======================================================================
✓ Collection contains 27975 documents

✓ ChromaDB ready for semantic search!

```
### Query CLI or as grownups call the frontend
Finally I made a tiny script as a semantic search tool for your Redmine-in-ChromaDB setup. After all issues are embedded and stored, this script lets you search them using natural language.

It connects to the local ChromaDB directory, loads the selected collection, and uses the same Ollama embedding model (e.g., nomic-embed-text or mxbai-embed-large) to embed your query text. Then it performs a similarity search against all stored Redmine issue vectors and returns the closest matches. The result list shows the document ID, similarity score, and a snippet of the stored issue text, giving you a fast, offline, vector-based search engine for historical Redmine data.
It is important to note here that for consistent and meaningful queries, the same embedding model must be used for both ingesting and querying.

### Conclusion
During this year's Hackweek I built a complete RAG pipeline from API extraction to semantic search. The checkpoint system and retry logic were essential for reliability. ChromaDB + Ollama provide a solid local-first RAG stack without external dependencies or API costs. I dumped the code to here: [https://github.com/bzoltan1/Redmine-RAG](https://github.com/bzoltan1/Redmine-RAG)

And the most important question: how does it work? I do not know yet. It needs to be used first with real questions. For this project I focused on building the pipeline.

My fundamental worry about this architecture and project in general is that the RAG system with a good LLM frontend is as smart as the data we built the database with. If the Redmine issues do not have smart comments, smart descriptions, and smart resolutions then the RAG system will be flat and stupid.

