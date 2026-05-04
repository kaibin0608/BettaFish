# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BettaFish (微舆) is a multi-agent public opinion analysis system. Users describe an analysis topic conversationally, and the system autonomously searches 30+ social media platforms, analyzes millions of comments, and generates interactive HTML reports. All LLM calls use the OpenAI API format, so any compatible provider works.

## Running the System

**Prerequisites**: Copy `.env.example` → `.env` and fill in all API keys and database credentials.

```bash
# Install dependencies
pip install -r requirements.txt

# Install Playwright browser driver (for crawlers)
playwright install chromium

# Start full system (Flask on port 5000)
python app.py
# Access at http://localhost:5000
```

**Run individual agents** (without the full system):
```bash
streamlit run SingleEngineApp/insight_engine_streamlit_app.py --server.port 8501
streamlit run SingleEngineApp/media_engine_streamlit_app.py --server.port 8502
streamlit run SingleEngineApp/query_engine_streamlit_app.py --server.port 8503
```

**CLI report generation** (skips analysis engines, reads latest logs):
```bash
python report_engine_only.py --query "topic name"
python report_engine_only.py --skip-pdf --verbose
```

**Re-render an existing report** from saved chapter JSON:
```bash
python regenerate_latest_html.py
python regenerate_latest_pdf.py
```

**MindSpider crawler**:
```bash
cd MindSpider
python main.py --setup                              # initialize
python main.py --complete --date 2024-01-20        # full crawl
python main.py --broad-topic                        # topic extraction only
python main.py --deep-sentiment --platforms xhs dy wb  # targeted crawl
```

**Docker**:
```bash
docker compose up -d
```

## Running Tests

```bash
# From project root
pytest tests/

# Or using the custom runner
cd tests && python run_tests.py
```

## Configuration

All configuration is managed by `config.py` via Pydantic Settings, loading from `.env`. The web UI at `/api/config` also reads/writes `.env` at runtime.

Each agent has independent LLM credentials (`{ENGINE}_API_KEY`, `{ENGINE}_BASE_URL`, `{ENGINE}_MODEL_NAME`). Recommended models per agent are documented in `.env.example`.

Key tuning parameters are in each engine's `utils/config.py` (e.g. `MAX_REFLECTIONS`, `MAX_SEARCH_RESULTS`).

## Architecture

### Multi-Agent Pipeline

A user query triggers three agents running in parallel, coordinated by **ForumEngine**:

| Agent | Port | Role |
|-------|------|------|
| QueryEngine | 8503 | Web/news search (Tavily, BochaAPI, AnspireAPI) |
| MediaEngine | 8502 | Multimodal content (video, image, structured cards) |
| InsightEngine | 8501 | Private database mining (SQLAlchemy async) |
| ReportEngine | — | Collects all results, generates IR, renders HTML/PDF/MD |
| ForumEngine | — | "Debate forum": monitors agent logs, LLM moderator guides cross-agent synthesis |

### ForumEngine Collaboration Mechanism

`ForumEngine/monitor.py` watches `logs/` for agent output. `ForumEngine/llm_host.py` generates moderator guidance that agents read via the `forum_reader` tool. This creates a multi-turn debate loop before report generation. The forum transcript is logged to `logs/forum.log` and streamed to the UI via Socket.IO.

### ReportEngine Pipeline

1. **Template selection** — picks a Markdown template from `ReportEngine/report_template/`
2. **Document layout** — LLM designs title, TOC, theme
3. **Word budget** — plans section lengths
4. **Chapter generation** — per-chapter JSON with IR blocks (text, charts, tables)
5. **Stitching** — assembles Document IR (`ReportEngine/core/stitcher.py`)
6. **Rendering** — `ReportEngine/renderers/html_renderer.py` → interactive HTML; `pdf_renderer.py` → WeasyPrint PDF

IR block/marker schema is defined in `ReportEngine/ir/schema.py`.

### Agent Internal Structure

Every engine (Query/Media/Insight) follows the same layout:
- `agent.py` — main orchestration loop with reflection
- `llms/base.py` — unified OpenAI-compatible streaming client with retry
- `nodes/` — processing nodes: `search_node`, `formatting_node`, `summary_node`, `report_structure_node`
- `tools/` — tool implementations called by the agent
- `state/state.py` — typed state dict passed between nodes
- `prompts/prompts.py` — all prompt templates for this engine
- `utils/config.py` — engine-specific tunable parameters

### Database

PostgreSQL (recommended) or MySQL. Schema defined in `MindSpider/schema/`. Tables split into:
- `models_bigdata.py` — large-scale media/social content tables (SQLAlchemy ORM)
- `models_sa.py` — `DailyTopic`, `Task`, and other operational tables

Database is auto-initialized when `app.py` starts via `MindSpider.initialize_database()`.

### Sentiment Analysis Models

Multiple interchangeable models in `SentimentAnalysisModel/`:
- `WeiboSentiment_Finetuned/` — BERT-Chinese LoRA and GPT-2 LoRA
- `WeiboMultilingualSentiment/` — multilingual
- `WeiboSentiment_SmallQwen/` — small Qwen3 fine-tune
- `WeiboSentiment_MachineLearning/` — SVM and other classical methods

Active model is configured in `InsightEngine/tools/sentiment_analyzer.py` via `SENTIMENT_CONFIG`.

### Key Output Directories

- `final_reports/` — generated HTML reports
- `final_reports/pdf/` — PDF exports
- `final_reports/md/` — Markdown exports
- `final_reports/ir/` — Document IR JSON (used for re-rendering)
- `logs/` — per-engine runtime logs and `forum.log`
- `{engine}_streamlit_reports/` — intermediate reports from individual Streamlit apps
