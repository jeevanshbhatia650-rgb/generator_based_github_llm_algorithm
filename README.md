# README → Workflow Extraction Pipeline

Scrapes GitHub README files and uses LLMs (via OpenRouter/Groq) to extract structured, step-by-step setup and usage workflows from them — built with **LangChain**, **LangGraph**, and **Pydantic**.

A different LLM is randomly selected from a model pool on every single repo, so the pipeline naturally load-balances across models and lets you compare extraction quality/reliability across them.

---

## How it works

```
repo_urls (generator)
      │
      ▼
┌─────────────────────────────────────────────┐
│              LangGraph pipeline               │
│                                               │
│   [scrape]  →  [call_llm]  →  [parse]        │
│                                               │
└─────────────────────────────────────────────┘
      │
      ▼
yields WorkflowExtraction objects, one repo at a time
```

1. **`scrape`** — fetches the raw README for a repo via the GitHub API.
2. **`call_llm`** — randomly picks a model from `MODEL_POOL`, fills a prompt template with the README content, and sends it off.
3. **`parse`** — validates and parses the model's response into a typed `WorkflowExtraction` object (project name, summary, ordered setup steps, ordered usage steps) using a Pydantic output parser.

The whole thing is driven by a Python generator (`run_pipeline`), so nothing is scraped or sent to an LLM until you actually iterate over the results — no wasted calls, no loading everything into memory upfront.

If a scrape or parse fails for a given repo, that failure is recorded in the result (`error` field) rather than crashing the run, so one bad repo doesn't stop the rest of the batch.

---

## Tech stack

| Piece | Purpose |
|---|---|
| [LangChain](https://python.langchain.com/) | Prompt templates, output parsing, chaining |
| [LangGraph](https://langchain-ai.github.io/langgraph/) | State-machine orchestration of scrape → call → parse |
| [Pydantic](https://docs.pydantic.dev/) | Schema validation for structured LLM output |
| [OpenRouter](https://openrouter.ai/) / [Groq](https://groq.com/) | LLM access — one endpoint, many models |
| GitHub REST API | Fetching raw README content |

---

## Setup

### 1. Install dependencies

```bash
pip install langchain langchain-openai langgraph pydantic requests
```

### 2. Set your API key

**In Google Colab (recommended):**
Add your key to Colab's secrets manager (🔑 icon in the left sidebar) under a name like `GROQ_API_KEY` or `OPENROUTER_API_KEY`, then in the notebook:

```python
from google.colab import userdata
API_KEY = userdata.get('GROQ_API_KEY')
```

**Locally / outside Colab:**
Create a `.env` file (already gitignored — never commit this):

```
GROQ_API_KEY=your_real_key_here
```

```python
from dotenv import load_dotenv
import os

load_dotenv()
API_KEY = os.environ.get("GROQ_API_KEY")
```

> ⚠️ Never hardcode a real API key directly in the notebook or scripts — it will get committed to git history and GitHub's push protection will block the push. Revoke any key that ever gets exposed this way.

### 3. (Optional) Add a GitHub token

For scraping many repos, add a `GITHUB_TOKEN` to avoid GitHub API rate limits (60 requests/hour unauthenticated vs. 5,000/hour authenticated).

---

## Usage

```python
repo_urls = [
    "https://github.com/langchain-ai/langgraph",
    "https://github.com/langchain-ai/langchain",
]

for result in run_pipeline(repo_urls):
    if result.get("error"):
        print(result["repo_url"], "->", result["error"])
        continue

    wf = result["workflow"]
    print(f"{wf.project_name}: {wf.summary}")
    print(f"  {len(wf.setup_steps)} setup steps, {len(wf.usage_steps)} usage steps")
```

Each result includes:

| Field | Description |
|---|---|
| `repo_url` | The repo processed |
| `model_used` | Which model from the pool handled this repo |
| `workflow.project_name` | Inferred project name |
| `workflow.summary` | 1–2 sentence summary |
| `workflow.setup_steps` | Ordered list of install/setup steps |
| `workflow.usage_steps` | Ordered list of run/usage steps |
| `error` | Present only if scraping or parsing failed |

---

## Project structure

```
.
├── readme_workflow_pipeline.ipynb   # Main notebook: config, pipeline, run
└── README.md
```

---

## Notes

- **Random model selection** happens per repo, not per run — useful for spreading load and for A/B-ing extraction quality across models in your pool.
- **README size**: very large READMEs may exceed a model's context window (this shows up as a `413 Payload Too Large` error). Consider truncating or chunking `readme_text` before sending it if you hit this.
- **Secrets**: this repo is safe to make public as long as API keys are only ever loaded via Colab secrets or a local `.env` file — never pasted directly into the notebook.
