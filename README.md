# Blog Writing Agent

A multi-agent blog writer built with **LangGraph** and **Streamlit**. Give it a topic, and it plans a full blog post, optionally researches the web for current information, fans the work out to parallel "worker" agents to draft each section, then merges everything into a final Markdown file — with optional AI-generated images.

The repo is organized as a progression of notebooks, each adding a capability, culminating in a production-ready backend (`bwa_backend.py`) and a Streamlit frontend (`bwa_frontend.py`).

## How it works

The core pipeline is a LangGraph `StateGraph`:

```
START
  │
  ▼
router ──(needs research?)──► research ──► orchestrator
  │                                             │
  └───────────────(no)─────────────────────────┘
                                                 │
                                          fan-out (Send)
                                                 │
                                     ┌───────────┼───────────┐
                                     ▼           ▼           ▼
                                  worker      worker      worker
                                     │           │           │
                                     └───────────┼───────────┘
                                                 ▼
                                 reducer (merge → decide images → generate images)
                                                 │
                                                END
```

1. **Router** — decides whether the topic needs live web research (`closed_book`, `hybrid`, or `open_book` mode) and generates search queries if so.
2. **Research** — runs the queries via Tavily and synthesizes the raw results into structured, deduplicated `EvidenceItem`s, filtered by recency.
3. **Orchestrator** — plans the blog: title, audience, tone, and 5–9 sections (`Task`s), each with a goal, bullet points, and a target word count.
4. **Workers** — run in parallel (via `Send`), each writing one section in Markdown, grounded in the provided evidence when required.
5. **Reducer** — a subgraph that:
   - merges all sections into one document,
   - decides whether diagrams/images would help (max 3),
   - generates those images (via Gemini) and inlines them into the Markdown,
   - saves the final post to disk.

The final output is a `.md` file (plus an `images/` folder if any images were generated).

## Project structure

```
├── 1_bwa_basic.ipynb                  # Minimal orchestrator → workers → reducer graph
├── 2_bwa_improved_prompting.ipynb     # Richer prompting/schemas
├── 3_bwa_research.ipynb               # Adds the router + Tavily research step
├── 4_bwa_research_fine_tuned.ipynb    # Tuned research/grounding logic
├── 5_bwa_image.ipynb                  # Adds the image planning/generation subgraph
├── bwa_backend.py                     # Final compiled LangGraph app (router→research→orchestrator→workers→reducer)
├── bwa_frontend.py                    # Streamlit UI for running the graph and browsing results
└── tavily_test.ipynb                  # Scratch notebook for testing the Tavily search tool
```

## Setup

### 1. Install dependencies

```bash
pip install langgraph langchain-openai langchain-community streamlit pandas python-dotenv google-genai
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```env
OPENAI_API_KEY=your-openai-api-key
TAVILY_API_KEY=your-tavily-api-key      # optional — enables web research
GOOGLE_API_KEY=your-google-api-key      # optional — enables image generation
```

- `OPENAI_API_KEY` is required (used for planning and writing via `gpt-4.1-mini`).
- `TAVILY_API_KEY` is optional. Without it, the research step simply returns no evidence and the graph falls back to closed-book writing.
- `GOOGLE_API_KEY` is optional. Without it, image generation fails gracefully and a placeholder note is inserted in the Markdown instead of the image.

## Usage

### Run the Streamlit app

```bash
streamlit run bwa_frontend.py
```

Then, in the sidebar:
1. Enter a blog **topic**.
2. Pick an **as-of date** (used for recency filtering during research).
3. Click **🚀 Generate Blog**.

The app streams progress through each graph node and shows:
- **🧩 Plan** — the generated outline and task table
- **🔎 Evidence** — sources pulled from research (if any)
- **📝 Markdown Preview** — the rendered blog post, with download buttons for the `.md` file and a zipped bundle (markdown + images)
- **🖼️ Images** — generated image assets
- **🧾 Logs** — raw event log of the graph run

Previously generated posts (`*.md` files in the working directory) are listed in the sidebar and can be reloaded into the viewer.

### Run programmatically

```python
from bwa_backend import app

result = app.invoke({
    "topic": "Write a blog on Self Attention",
    "mode": "",
    "needs_research": False,
    "queries": [],
    "evidence": [],
    "plan": None,
    "as_of": "2026-07-12",
    "recency_days": 7,
    "sections": [],
    "merged_md": "",
    "md_with_placeholders": "",
    "image_specs": [],
    "final": "",
})

print(result["final"])
```

The finished post is also written to disk as `<slugified-title>.md`, with any images saved under `images/`.

## Key concepts

- **Orchestrator–worker pattern**: a single planning step produces a set of tasks, which are dispatched to parallel workers via LangGraph's `Send` API, then reduced back into one document.
- **Grounded generation**: in `hybrid`/`open_book` modes, workers are instructed to cite only URLs present in the retrieved evidence, and to explicitly flag unsupported claims rather than inventing them.
- **Graceful degradation**: missing API keys (Tavily, Google) don't break the graph — research and image generation degrade to no-ops or placeholder text instead of failing the whole run.

## Notes

- Models are currently hardcoded to `gpt-4.1-mini` (text) and `gemini-2.5-flash-image` (images) inside `bwa_backend.py` — change these if you'd like to use different models.
- The notebooks (`1_bwa_basic.ipynb` → `5_bwa_image.ipynb`) are meant to be read in order — they show the incremental design decisions behind `bwa_backend.py` and are a good place to start if you want to understand or extend the pipeline.