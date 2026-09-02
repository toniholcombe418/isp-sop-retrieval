# ISP SOP Retrieval

Ask a plain-language question, get the Indiana State Police internal
procedure that actually answers it — plus a grounded two-line answer.

## How it works

### 1. Extraction — `fetch_sops.py`

Pulls the document tree from PowerDMS (ISP's public procedure site).
Walks the tree to find every real document (skips folders), downloads
each PDF, extracts the text, and saves everything to `sops/nodes.json`
plus one `.txt` file per document in `sops/text/`.

### 2. Enrichment — `parse_headers.py`

Reads those text files. Every SOP has a standard form header —
Reference Number, Subject, Effective Date — plus a "I. PURPOSE"
section. This regex-parses those fields out and saves them to
`sops/nodes_enriched.json`. Subject and Purpose are human-written, so
they're a clean, free retrieval signal — no LLM needed for this step.

### 3. Retrieval, Stage 1 — shortlist (`sop_tester.py`, `shortlist()`)

A question gets embedded and compared against five signals, each
embedded the same way:

- **purpose** — the Purpose statement
- **subject** — the Subject line
- **question** — pre-generated FAQ-style questions per doc (if cached)
- **name** — the raw filename/title (catches Rules docs with no Purpose)
- **summary** — LLM-written one-liners for docs with no metadata at all

Top 8 matches from each signal are unioned into one candidate pool
(~15–25 docs). This stage is about **recall** — get the right doc
somewhere in the pool.

### 4. Retrieval, Stage 2 — pick (`llm_pick()`)

The candidate pool (id, subject, purpose) is handed to llama3.1 with
the actual question, and it picks the one that really answers it — or
says NONE. This stage is about **precision** — embeddings only measure
word overlap; the LLM reasons about meaning.

### 5. Answer — `answer_from_sop()`

The picked doc's full text is fed back to llama3.1 with the question,
forced into a two-line `ANSWER:` / `COVERS:` format, grounded only in
that document.

### 6. Logging — `run_tester()`

Every question, pick, and answer is logged to `tester_log.jsonl` along
with a y/n/skip verdict — turning test sessions into an evaluation
dataset.

## Why two stages

Embeddings are cheap and fast but only measure resemblance. The LLM is
slow but can reason about whether a document actually answers a
question. Use embeddings to narrow 260 documents down to ~20, then let
the LLM make the one hard judgment call.

## Setup

```
pip install requests pypdf ollama sentence-transformers numpy
ollama pull llama3.1
```

## Run order

```
python fetch_sops.py       # builds sops/nodes.json + sops/text/*.txt
python parse_headers.py    # builds sops/nodes_enriched.json
python sop_tester.py       # run_tester() to drive it, benchmark() to sanity-check
```

`sop_summaries.json` is generated automatically on first run of
`sop_tester.py` for any document missing a Purpose statement.
