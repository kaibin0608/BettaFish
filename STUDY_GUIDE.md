# BettaFish Codebase Study Guide

**Goal**: Learn the architecture of BettaFish so you can extract and implement its patterns into your own autonomous news-tracking and social-post aggregation tool (Python + PostgreSQL + LLMs).

**Your starting point**: Comfortable with asyncio, have used LLMs via API, running PostgreSQL.

**How to use this guide**: Work through each phase in order. Each phase has files to read, concepts explained with real code excerpts, and a checkpoint exercise. Don't move to the next phase until the checkpoint feels natural.

---

## Table of Contents

- [Phase 1: The Crawling Layer (MindSpider)](#phase-1-the-crawling-layer-mindspider)
- [Phase 2: The Agent Core Pattern](#phase-2-the-agent-core-pattern)
- [Phase 3: The LLM Client and Retry System](#phase-3-the-llm-client-and-retry-system)
- [Phase 4: The Tools Layer (Connecting Agents to Data)](#phase-4-the-tools-layer-connecting-agents-to-data)
- [Phase 5: ForumEngine — Multi-Agent Coordination](#phase-5-forumengine--multi-agent-coordination)
- [Phase 6: ReportEngine — Structured Output](#phase-6-reportengine--structured-output)
- [Build Order for Your Own Tool](#build-order-for-your-own-tool)

---

## Phase 1: The Crawling Layer (MindSpider)

**Why start here**: Everything in the system depends on having data. Before you can run any agents, you need scraped posts in your PostgreSQL database. This phase covers how BettaFish gets that data.

### 1.1 — The Two-Stage Crawl Architecture

BettaFish splits crawling into two stages. This is a design decision you should copy:

```
Stage 1: Broad Topic Extraction
  → What is trending right now?
  → LLM reads news headlines and extracts structured topics
  → Topics are saved to the `DailyTopic` table in PostgreSQL

Stage 2: Deep Sentiment Crawling
  → For each extracted topic, crawl 30+ platforms for posts and comments
  → Store raw content in the big content tables
```

**Why two stages matter**: If you deep-crawl everything without first filtering by relevance, you waste API calls and fill your DB with noise. The LLM in Stage 1 is a cheap filter that decides "is this topic worth crawling?" before you spend time on Playwright scraping.

**Files to read:**

| File | What it does |
|------|-------------|
| [MindSpider/BroadTopicExtraction/get_today_news.py](MindSpider/BroadTopicExtraction/get_today_news.py) | Fetches today's trending news from news APIs |
| [MindSpider/BroadTopicExtraction/topic_extractor.py](MindSpider/BroadTopicExtraction/topic_extractor.py) | LLM reads raw headlines, extracts structured `{topic, keywords, category}` |
| [MindSpider/BroadTopicExtraction/database_manager.py](MindSpider/BroadTopicExtraction/database_manager.py) | Persists extracted topics to PostgreSQL |
| [MindSpider/BroadTopicExtraction/main.py](MindSpider/BroadTopicExtraction/main.py) | Orchestrates the two sub-steps above |

**Pattern to extract for your tool**: The `topic_extractor.py` pattern — feed raw text to an LLM with a structured prompt, get back JSON with `{topic, keywords, priority}`. Use this as a pre-filter before you ever touch a browser.

---

### 1.2 — The Deep Crawl Architecture

**Files to read:**

| File | What it does |
|------|-------------|
| [MindSpider/DeepSentimentCrawling/keyword_manager.py](MindSpider/DeepSentimentCrawling/keyword_manager.py) | Expands topics into platform-specific search keywords |
| [MindSpider/DeepSentimentCrawling/platform_crawler.py](MindSpider/DeepSentimentCrawling/platform_crawler.py) | Dispatches crawl jobs to each platform crawler |
| [MindSpider/DeepSentimentCrawling/main.py](MindSpider/DeepSentimentCrawling/main.py) | Entry point: reads today's topics, runs deep crawl |

**Files to read in MediaCrawler:**

| File | What it does |
|------|-------------|
| [MindSpider/DeepSentimentCrawling/MediaCrawler/base/base_crawler.py](MindSpider/DeepSentimentCrawling/MediaCrawler/base/base_crawler.py) | Abstract base class every platform crawler inherits |
| [MindSpider/DeepSentimentCrawling/MediaCrawler/cache/abs_cache.py](MindSpider/DeepSentimentCrawling/MediaCrawler/cache/abs_cache.py) | Abstract cache interface |
| [MindSpider/DeepSentimentCrawling/MediaCrawler/cache/local_cache.py](MindSpider/DeepSentimentCrawling/MediaCrawler/cache/local_cache.py) | Local dict-based cache (good for dev) |
| [MindSpider/DeepSentimentCrawling/MediaCrawler/cache/redis_cache.py](MindSpider/DeepSentimentCrawling/MediaCrawler/cache/redis_cache.py) | Redis cache (for production) |

**The cache pattern** is important to understand. Every platform crawler checks the cache before making a network request:

```python
# Pseudocode showing the pattern used in every platform crawler
async def get_post_detail(self, post_id: str) -> dict:
    cached = await self.cache.get(f"post:{post_id}")
    if cached:
        return cached
    
    # Not cached — fetch from network
    async with aiohttp.ClientSession() as session:
        async with session.get(url, headers=self.headers) as resp:
            data = await resp.json()
    
    await self.cache.set(f"post:{post_id}", data, expire=3600)
    return data
```

**aiohttp lesson**: `aiohttp.ClientSession` is the async equivalent of `requests.Session`. The key difference:
- `requests.get(url)` — blocks the thread until response arrives
- `async with session.get(url) as resp:` — yields control back to the event loop while waiting

This is why you can run 50 concurrent requests with asyncio+aiohttp without 50 threads.

---

### 1.3 — The Database Schema

**Files to read:**

| File | What it does |
|------|-------------|
| [MindSpider/schema/mindspider_tables.sql](MindSpider/schema/mindspider_tables.sql) | Raw SQL — read this first for the full schema picture |
| [MindSpider/schema/models_bigdata.py](MindSpider/schema/models_bigdata.py) | SQLAlchemy ORM for the large content tables |
| [MindSpider/schema/models_sa.py](MindSpider/schema/models_sa.py) | ORM for operational tables (`DailyTopic`, `Task`) |
| [MindSpider/schema/db_manager.py](MindSpider/schema/db_manager.py) | Database connection and session management |
| [MindSpider/schema/init_database.py](MindSpider/schema/init_database.py) | Creates all tables on first run |

**Key design insight** — the schema is split into two groups:

```
Operational tables (models_sa.py):
  - DailyTopic: {id, topic_name, keywords, date, status, priority}
  - Task:       {id, topic_id, platform, status, created_at, completed_at}

Content tables (models_bigdata.py):
  - WeiboPosts, XhsPosts, DouyinPosts, etc. (one table per platform)
  - All share similar columns: {id, topic_id, content, author, likes, url, publish_time}
```

**Pattern to copy for your tool**: Separate your "what do I need to crawl" metadata (operational) from the raw scraped content. This lets you query job status without touching large content tables.

---

### Checkpoint 1

After reading Phase 1, you should be able to answer:
- Why does BettaFish do a broad pass before a deep crawl?
- What does `base_crawler.py` enforce that every platform crawler must implement?
- How does the cache prevent duplicate network requests?
- Why is `aiohttp` used instead of `requests` in the crawler?

**Mini exercise**: Sketch your own two-table schema — one for "topics I want to track" and one for "raw posts collected". What columns would you include?

---

## Phase 2: The Agent Core Pattern

**Why this is the most important phase**: This is the pattern you'll use for every AI-driven feature in your tool. Understand it deeply — everything else builds on it.

### 2.1 — The State Object

**File to read**: [InsightEngine/state/state.py](InsightEngine/state/state.py)

The `State` object is a dataclass hierarchy that flows through every processing step. Think of it as a clipboard that every node can read and write to.

```python
# From InsightEngine/state/state.py — simplified to show structure

@dataclass
class Search:
    """One individual search result"""
    query: str = ""
    url: str = ""
    title: str = ""
    content: str = ""
    score: Optional[float] = None

@dataclass
class Research:
    """The research progress for one paragraph/section"""
    search_history: List[Search] = field(default_factory=list)
    latest_summary: str = ""       # LLM's latest synthesis of all searches so far
    reflection_iteration: int = 0  # how many times we've refined
    is_completed: bool = False

@dataclass
class Paragraph:
    """One section of the final report"""
    title: str = ""
    content: str = ""              # initial plan (before research)
    research: Research = field(default_factory=Research)
    order: int = 0

@dataclass
class State:
    """The entire task — passed through all nodes"""
    query: str = ""                # original user query
    report_title: str = ""
    paragraphs: List[Paragraph] = field(default_factory=list)
    final_report: str = ""
    is_completed: bool = False
```

**Why this design matters**:
1. Each node receives the full `State` — it never needs to know what ran before it
2. Nodes write their results *into* the state instead of returning them — makes the flow readable in `agent.py`
3. The state can be serialized to JSON (`state.to_json()`) and saved to disk — you can resume a crashed run
4. Progress is trackable: `state.get_progress_summary()` tells you how many sections are done

**Pattern to adapt for your tool**: Your state might look like:

```python
@dataclass
class ArticleResearch:
    query: str = ""
    raw_posts: List[dict] = field(default_factory=list)
    sentiment_summary: str = ""
    key_themes: List[str] = field(default_factory=list)
    final_digest: str = ""
    is_completed: bool = False
```

---

### 2.2 — The Node Pattern

**File to read**: [InsightEngine/nodes/base_node.py](InsightEngine/nodes/base_node.py)

Every processing step is a `Node`. Here's the base class:

```python
# From InsightEngine/nodes/base_node.py

class BaseNode(ABC):
    def __init__(self, llm_client: LLMClient, node_name: str = ""):
        self.llm_client = llm_client
        self.node_name = node_name or self.__class__.__name__

    @abstractmethod
    def run(self, input_data: Any, **kwargs) -> Any:
        """Execute the node's processing logic"""
        pass

class StateMutationNode(BaseNode):
    @abstractmethod
    def mutate_state(self, input_data: Any, state: State, **kwargs) -> State:
        """Modify the state and return the updated state"""
        pass
```

There are two variants:
- `BaseNode.run()` — takes raw input, returns raw output (used for search queries, tool calls)
- `StateMutationNode.mutate_state()` — takes + returns the full State (used for LLM summary steps that update the state)

**Read these nodes in order:**

| File | Role in the pipeline |
|------|---------------------|
| [InsightEngine/nodes/search_node.py](InsightEngine/nodes/search_node.py) | Asks LLM which tool to use and what query to run |
| [InsightEngine/nodes/formatting_node.py](InsightEngine/nodes/formatting_node.py) | LLM cleans and structures raw search results |
| [InsightEngine/nodes/summary_node.py](InsightEngine/nodes/summary_node.py) | LLM synthesizes a paragraph summary from search results |
| [InsightEngine/nodes/report_structure_node.py](InsightEngine/nodes/report_structure_node.py) | LLM plans the report outline before any searching begins |

**The key separation**: Nodes are dumb — they just run a prompt and return. The *agent* decides which node to run when. You can add, remove, or swap nodes without touching the agent loop.

---

### 2.3 — The Agent Loop (The Most Important Pattern)

**File to read**: [InsightEngine/agent.py](InsightEngine/agent.py)

The `DeepSearchAgent.research()` method is the main loop. Read it carefully — this is what makes the agent "autonomous":

```python
# From InsightEngine/agent.py — the main research flow

def research(self, query: str) -> str:
    # Step 1: Plan the report structure
    # LLM decides what sections to write before searching anything
    self._generate_report_structure(query)

    # Step 2: For each planned section, run a search+reflection loop
    for i in range(len(self.state.paragraphs)):
        self._initial_search_and_summary(i)   # search once, summarize
        self._reflection_loop(i)               # refine MAX_REFLECTIONS times
        self.state.paragraphs[i].research.mark_completed()

    # Step 3: Combine all section summaries into a final report
    return self._generate_final_report()
```

**The reflection loop** (inside `_reflection_loop`) is the core autonomous behavior:

```python
# From InsightEngine/agent.py — simplified reflection loop

def _reflection_loop(self, paragraph_index: int):
    paragraph = self.state.paragraphs[paragraph_index]

    for i in range(self.config.MAX_REFLECTIONS):  # default: 3 iterations
        # Ask the LLM: "Given what you've found so far, what are you still missing?"
        reflection_output = self.reflection_node.run({
            "title": paragraph.title,
            "paragraph_latest_state": paragraph.research.latest_summary  # current best summary
        })

        # The LLM generates a NEW search query targeting the gap it identified
        new_query = reflection_output["search_query"]
        reasoning = reflection_output["reasoning"]  # why this query

        # Run the new search
        search_response = self.execute_search_tool(new_query)

        # Add results to history and update the summary
        paragraph.research.add_search_results(new_query, search_response.results)
        self.state = self.reflection_summary_node.mutate_state(...)
```

**Why this works**:
- Pass 1: agent searches broadly, gets a rough summary
- Pass 2: agent reads that summary, identifies what's missing, searches specifically for gaps
- Pass 3+: refines further until `MAX_REFLECTIONS` is reached

Without reflection, you'd get one search pass and whatever the LLM makes of it. With reflection, each pass fills in gaps the previous pass missed. The result is dramatically better coverage.

**One more important pattern** in `execute_search_tool()`: before every search, it runs the query through `keyword_optimizer` — a small LLM call that expands a natural-language query into 2-3 optimized search terms. This is what lets the agent handle queries like "public opinion on electric vehicles" and automatically search for "新能源汽车 舆情", "EV public sentiment", "electric car reviews", etc.

---

### Checkpoint 2

After reading Phase 2, you should be able to answer:
- Why does `State` use dataclasses with `field(default_factory=list)` instead of mutable defaults?
- What's the difference between `BaseNode.run()` and `StateMutationNode.mutate_state()`?
- What happens during `_reflection_loop`? Why is it better than a single-pass search?
- Why does the agent plan the report structure *before* doing any searching?

**Mini exercise**: Write a `State` dataclass for your own news aggregator tool. What fields do you need? (Hint: you need to track what you've searched, what you've found, and what the LLM has synthesized.)

---

## Phase 3: The LLM Client and Retry System

**Why its own phase**: In production, LLM APIs fail constantly — rate limits, timeouts, network blips. This phase shows how BettaFish handles that robustly.

### 3.1 — The LLM Client

**File to read**: [InsightEngine/llms/base.py](InsightEngine/llms/base.py)

The `LLMClient` wraps the OpenAI SDK with three features you need in any production tool:

**Feature 1: Unified interface for any OpenAI-compatible provider**

```python
# From InsightEngine/llms/base.py

class LLMClient:
    def __init__(self, api_key: str, model_name: str, base_url: Optional[str] = None):
        client_kwargs = {"api_key": api_key, "max_retries": 0}
        if base_url:
            client_kwargs["base_url"] = base_url  # point to DeepSeek, Kimi, etc.
        self.client = OpenAI(**client_kwargs)
```

Setting `max_retries=0` on the OpenAI client disables its built-in retry — BettaFish uses its own retry decorator instead, which gives more control over delays and backoff.

**Feature 2: Time-stamped prompts**

```python
    def invoke(self, system_prompt: str, user_prompt: str, **kwargs) -> str:
        # Prepend current time to every user message
        # This prevents the LLM from hallucinating outdated dates
        current_time = datetime.now().strftime("%Y年%m月%d日%H时%M分")
        user_prompt = f"今天的实际时间是{current_time}\n{user_prompt}"
```

Simple but important — the LLM always knows the current date when it analyzes news or social content.

**Feature 3: Streaming + safe byte concatenation**

```python
    def stream_invoke_to_string(self, system_prompt, user_prompt, **kwargs) -> str:
        # Collects streamed chunks as bytes, not strings
        # Then decodes once at the end
        # This prevents UTF-8 multi-byte character truncation mid-stream
        byte_chunks = []
        for chunk in self.stream_invoke(system_prompt, user_prompt, **kwargs):
            byte_chunks.append(chunk.encode('utf-8'))
        return b''.join(byte_chunks).decode('utf-8', errors='replace')
```

This matters if you're processing Chinese, Japanese, or emoji content — streaming can cut a multi-byte character in half if you naively concatenate strings. Collect bytes, decode once.

---

### 3.2 — The Retry System

**File to read**: [utils/retry_helper.py](utils/retry_helper.py)

BettaFish has two separate retry configs — one for LLM calls, one for search APIs. This is intentional:

```python
# From utils/retry_helper.py

# LLM calls: wait longer between retries (rate limits need time to reset)
LLM_RETRY_CONFIG = RetryConfig(
    max_retries=6,
    initial_delay=60.0,    # wait 60 seconds before first retry
    backoff_factor=2.0,    # 60s → 120s → 240s → ...
    max_delay=600.0        # never wait more than 10 minutes
)

# Search API calls: retry faster (these fail transiently)
SEARCH_API_RETRY_CONFIG = RetryConfig(
    max_retries=5,
    initial_delay=2.0,
    backoff_factor=1.6,
    max_delay=25.0
)
```

The retry decorator is applied on top of the LLM client's `invoke` method:

```python
class LLMClient:
    @with_retry(LLM_RETRY_CONFIG)  # automatically retries on any exception
    def invoke(self, system_prompt, user_prompt, **kwargs) -> str:
        response = self.client.chat.completions.create(...)
        return response.choices[0].message.content
```

There's also `with_graceful_retry` — same idea but instead of raising an exception on final failure, it returns a default value. Use this for non-critical API calls where failure should degrade gracefully rather than crash the pipeline.

---

### Checkpoint 3

**Mini exercise**: Copy `LLMClient` and `retry_helper.py` into your own project. Modify `LLMClient.__init__` to read credentials from your `.env`. Call `client.invoke("You are a news analyst", "Summarize this headline: ...")`. Verify the retry decorator fires if you set `max_retries=1` and temporarily break the API key.

---

## Phase 4: The Tools Layer (Connecting Agents to Data)

**Why this matters for you**: Your tool needs agents that can query your PostgreSQL DB and external news APIs. This phase shows exactly how BettaFish wires those connections.

### 4.1 — Database Query Tools (InsightEngine)

**Files to read:**

| File | What it does |
|------|-------------|
| [InsightEngine/utils/db.py](InsightEngine/utils/db.py) | Async SQLAlchemy engine setup (copy this for your own DB connection) |
| [InsightEngine/tools/search.py](InsightEngine/tools/search.py) | The 5 database query functions agents can call |

**The async DB pattern** from `InsightEngine/utils/db.py`:

```python
# Pattern for async PostgreSQL queries with SQLAlchemy

from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Use asyncpg driver: postgresql+asyncpg://...
engine = create_async_engine(DATABASE_URL, pool_size=5, max_overflow=10)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_session():
    async with AsyncSessionLocal() as session:
        yield session
```

**The 5 tool functions** in `InsightEngine/tools/search.py` are what the agent can call:

```python
# Simplified signatures — shows the interface the agent uses

search_hot_content(time_period="week", limit=100) -> DBResponse
search_topic_globally(topic: str, limit_per_table=50) -> DBResponse
search_topic_by_date(topic: str, start_date, end_date, limit_per_table=100) -> DBResponse
get_comments_for_topic(topic: str, limit=500) -> DBResponse
search_topic_on_platform(platform: str, topic: str, limit=200) -> DBResponse
```

**The `DBResponse` wrapper**:

```python
@dataclass
class DBResponse:
    tool_name: str
    parameters: dict          # what was searched (for the LLM to understand context)
    results: List[DBResult]   # actual rows from the database
    results_count: int
    metadata: dict = None     # optional: sentiment analysis, aggregation stats
```

The agent receives `DBResponse` objects — it never writes raw SQL. Tools are the only layer that knows about tables.

**For your tool**: Create one `search.py` file with async functions like:
```python
async def search_posts_by_keyword(keyword: str, limit=100) -> list[dict]
async def get_posts_by_date_range(start: date, end: date) -> list[dict]
async def get_comments_for_post(post_id: str) -> list[dict]
```
Return plain dicts, not ORM objects — the agent passes results to LLM prompts and LLMs need clean text, not SQLAlchemy model instances.

---

### 4.2 — The Keyword Optimizer (Critical Pattern)

**File to read**: [InsightEngine/tools/keyword_optimizer.py](InsightEngine/tools/keyword_optimizer.py)

This is a small but high-leverage tool. When an agent receives "analyze public opinion on electric vehicles", it calls `keyword_optimizer.optimize_keywords()` first:

```python
# What keyword_optimizer does (simplified)

def optimize_keywords(original_query: str, context: str) -> OptimizedKeywords:
    prompt = f"""
    Original query: {original_query}
    Context: {context}
    
    Generate 2-4 optimized search keywords that would find relevant database entries.
    Consider: synonyms, abbreviated forms, related terms, both Chinese and English variants.
    Return as JSON: {{"keywords": [...], "reasoning": "..."}}
    """
    response = self.llm_client.invoke(SYSTEM_PROMPT, prompt)
    return parse_json_response(response)
```

Then the agent runs each keyword separately and merges results:

```python
# From InsightEngine/agent.py — after keyword optimization

for keyword in optimized_keywords:
    response = db_tools.search_topic_globally(keyword, limit=50)
    all_results.extend(response.results)

# Deduplicate by URL
unique_results = deduplicate_by_url(all_results)
```

**Pattern to steal**: never run a single hardcoded keyword search. Always expand first. This is cheap (a small/fast LLM model works fine here) and dramatically improves recall.

---

### 4.3 — Clustering for LLM Context Management

Back in `InsightEngine/agent.py` there's a clustering step that's easy to miss but very useful:

```python
# From InsightEngine/agent.py

ENABLE_CLUSTERING = True
MAX_CLUSTERED_RESULTS = 50
RESULTS_PER_CLUSTER = 5

def _cluster_and_sample_results(self, results, max_results=50):
    """
    When you have 500 posts about a topic, you can't send them all to the LLM.
    Use KMeans clustering to find representative samples from each sub-theme.
    Returns max_results posts that together cover the diversity of all 500.
    """
    texts = [r.title_or_content[:500] for r in results]
    embeddings = SentenceTransformer("paraphrase-multilingual-MiniLM-L12-v2").encode(texts)
    
    n_clusters = max_results // RESULTS_PER_CLUSTER  # e.g. 50/5 = 10 clusters
    labels = KMeans(n_clusters=n_clusters).fit_predict(embeddings)
    
    # Pick the 5 hottest posts from each cluster
    sampled = []
    for cluster_id in range(n_clusters):
        cluster_posts = [r for r, label in zip(results, labels) if label == cluster_id]
        cluster_posts.sort(key=lambda x: x.hotness_score, reverse=True)
        sampled.extend(cluster_posts[:RESULTS_PER_CLUSTER])
    
    return sampled
```

**Why this matters**: LLM context windows are limited. If you find 500 posts, you can't send all 500 to the LLM. Naive truncation (top 50 by score) biases towards viral content and misses minority voices. Clustering samples from *across* the opinion space — you get 50 posts that together represent the diversity of all 500.

**For your tool**: You can skip this initially and just truncate by score. Add clustering once you notice your summaries are biased towards high-engagement posts.

---

### Checkpoint 4

After reading Phase 4, you should be able to answer:
- Why do tools return `DBResponse` objects instead of raw lists?
- Why does keyword optimization run as a separate LLM call before searching?
- What problem does clustering solve that truncating by score doesn't?

**Mini exercise**: Write a `search_posts_by_keyword(keyword, limit)` async function for your own PostgreSQL schema. Return a list of dicts with at least `{title, content, url, published_at, platform}`. Test it with `asyncio.run(search_posts_by_keyword("bitcoin", 10))`.

---

## Phase 5: ForumEngine — Multi-Agent Coordination

**Prerequisite**: Complete Phases 2 and 3 first. ForumEngine only makes sense once you know what each agent does alone.

**What ForumEngine solves**: When you run 3 agents in parallel, they might all research the same sub-topic while ignoring others. ForumEngine is a moderator that reads what all agents have found and tells each one "you've covered X well, now focus on Y."

### 5.1 — The Log-Based Communication Pattern

**File to read**: [ForumEngine/monitor.py](ForumEngine/monitor.py)

The communication mechanism is simple but clever: agents write structured output to log files. The monitor watches those files for changes.

```python
# From ForumEngine/monitor.py

class LogMonitor:
    def __init__(self, log_dir="logs"):
        # Watch these three files — one per agent
        self.monitored_logs = {
            'insight': log_dir / 'insight.log',
            'media':   log_dir / 'media.log',
            'query':   log_dir / 'query.log'
        }
        
        # Trigger host speech every 5 new agent messages
        self.host_speech_threshold = 5
        self.agent_speeches_buffer = []
        
        # Which log lines to look for (summary nodes write these patterns)
        self.target_node_patterns = [
            'FirstSummaryNode',
            'ReflectionSummaryNode',
            '正在生成首次段落总结',
            '正在生成反思总结',
        ]
```

When a matching line appears in any agent's log, it's captured as an "agent speech" and added to the buffer. When 5 speeches accumulate, the host LLM fires.

**Why log files instead of a message queue**: The agents are separate processes (Streamlit apps). Shared files work across process boundaries without additional infrastructure. In a single-process design, you'd use a shared in-memory queue instead — simpler and lower latency.

---

### 5.2 — The LLM Moderator

**File to read**: [ForumEngine/llm_host.py](ForumEngine/llm_host.py)

```python
# Simplified version of what the host LLM does

def generate_host_speech(agent_speeches: list[str]) -> str:
    """
    Reads what all agents have said so far.
    Generates guidance directing agents toward unexplored angles.
    """
    combined = "\n".join(agent_speeches)
    
    prompt = f"""
    You are a research moderator. Three agents (Query, Media, Insight) are researching 
    a topic. Here is what they've found so far:
    
    {combined}
    
    Based on this, identify:
    1. What has been covered well
    2. What important angles are missing
    3. Direct each agent toward specific unexplored areas
    
    Be concrete and specific. Your guidance will be read by the agents in their next search.
    """
    return llm.invoke(MODERATOR_SYSTEM_PROMPT, prompt)
```

This guidance is written to `logs/forum.log`. The agents read it via the `forum_reader` tool (see [utils/forum_reader.py](utils/forum_reader.py)) and incorporate it into their next search queries.

**For your tool**: If you only run one agent, skip ForumEngine entirely. Add it if you want to run a "news tracker agent" + "social post agent" in parallel — the moderator prevents them from duplicating work.

---

### Checkpoint 5

After reading Phase 5, you should be able to answer:
- Why does ForumEngine use log files instead of a shared in-memory queue?
- What triggers the host LLM to generate guidance?
- How do agents actually receive and use the host's guidance?

---

## Phase 6: ReportEngine — Structured Output

**You can skip this phase** if you want structured JSON output (a digest, a DB record, a webhook payload) rather than a visual HTML report. Come back to it once your core pipeline works.

### 6.1 — The IR (Intermediate Representation) Pipeline

Instead of asking the LLM to write HTML directly (fragile, hard to validate), BettaFish uses a two-step approach:

```
Step 1: LLM generates structured JSON (the IR)
  {
    "sections": [
      {
        "title": "Market Overview",
        "type": "text",
        "content": "The electric vehicle market saw..."
      },
      {
        "title": "Sentiment Distribution",
        "type": "chart",
        "chart_type": "pie",
        "data": {"positive": 0.62, "negative": 0.21, "neutral": 0.17}
      }
    ]
  }

Step 2: A deterministic renderer converts IR → HTML/PDF/Markdown
  (No LLM involved in rendering — pure template code)
```

**Why this is better than LLM → HTML directly**:
- You can validate the JSON before rendering (is the chart data valid? are sections complete?)
- You can re-render the same IR to different formats (HTML, PDF, Markdown)
- If the LLM generates bad output, you know exactly where it failed (JSON validation step)

**Key files:**

| File | What it does |
|------|-------------|
| [ReportEngine/ir/schema.py](ReportEngine/ir/schema.py) | Defines all valid block types and their required fields |
| [ReportEngine/ir/validator.py](ReportEngine/ir/validator.py) | Validates each chapter JSON before rendering |
| [ReportEngine/core/stitcher.py](ReportEngine/core/stitcher.py) | Assembles individual chapter JSONs into a complete Document IR |
| [ReportEngine/renderers/html_renderer.py](ReportEngine/renderers/html_renderer.py) | Document IR → interactive HTML |

### 6.2 — The Chapter Generation Node

**File to read**: [ReportEngine/nodes/chapter_generation_node.py](ReportEngine/nodes/chapter_generation_node.py)

This is the most instructive node in the whole codebase. It shows how to get reliable structured JSON from an LLM:

```python
# Pseudocode of the pattern used in chapter_generation_node.py

def generate_chapter(self, chapter_title: str, research_content: str) -> dict:
    for attempt in range(MAX_RETRIES):
        raw_response = self.llm_client.invoke(
            CHAPTER_SYSTEM_PROMPT,
            f"Write the '{chapter_title}' chapter.\n\nSource material:\n{research_content}"
        )
        
        # Try to extract JSON from the response
        chapter_json = extract_json(raw_response)
        
        # Validate against the IR schema
        errors = self.validator.validate(chapter_json)
        
        if not errors:
            return chapter_json  # success
        
        # If validation fails, ask the LLM to fix it
        raw_response = self.llm_client.invoke(
            REPAIR_PROMPT,
            f"Your previous output had these errors: {errors}\nOriginal output: {raw_response}\nPlease fix it."
        )
    
    raise ValueError(f"Could not generate valid chapter after {MAX_RETRIES} attempts")
```

**For your tool**: use this exact pattern whenever you need structured output from an LLM. Never just `json.loads(response)` without a retry loop — LLMs occasionally output malformed JSON or add markdown fences around it.

---

### Checkpoint 6

After reading Phase 6, you should understand:
- Why the IR approach is better than LLM → HTML directly
- How the validation + repair loop works in chapter generation
- When you'd want structured IR vs. just returning raw Markdown

---

## Build Order for Your Own Tool

Here's the recommended build sequence, using BettaFish's patterns. Each step builds directly on the previous one.

### Week 1 — Data Foundation

**Goal**: Async news fetcher → posts stored in PostgreSQL

1. Copy `InsightEngine/utils/db.py` — set up your async SQLAlchemy engine
2. Design your schema (copy the operational/content table split from Phase 1.3)
3. Copy the `aiohttp` session pattern from `MediaCrawler` — write one async fetcher for one news source
4. Copy `MindSpider/BroadTopicExtraction/topic_extractor.py` — LLM extracts structured topics from headlines

**End of Week 1**: You can fetch news, extract topics with LLM, store in PostgreSQL.

---

### Week 2 — Single Agent

**Goal**: An agent that autonomously researches a topic using your database

1. Copy `InsightEngine/llms/base.py` + `utils/retry_helper.py` → your LLM client
2. Write your `State` dataclass (Phase 2.1)
3. Write your `BaseNode` + 3 nodes: `SearchNode`, `SummaryNode`, `ReflectionNode` (Phase 2.2)
4. Copy `InsightEngine/tools/keyword_optimizer.py` → your keyword expander
5. Write your `search_posts_by_keyword()` tool (Phase 4.1)
6. Write your `Agent.research(query)` loop with `_reflection_loop` (Phase 2.3)

**End of Week 2**: You can give your agent a topic, it searches your DB, reflects, produces a written summary.

---

### Week 3 — Web Search + Better Tools

**Goal**: Agent can search the web in addition to your local DB

1. Read `QueryEngine/tools/` — add a web search tool (Tavily or similar)
2. Add the clustering step from Phase 4.3 when you have too many results
3. Add `with_graceful_retry` to your search tools (so web failures don't crash the loop)
4. Add the `time-stamped prompt` trick from Phase 3.1 to your LLM client

**End of Week 3**: Agent can combine local DB + live web search + reflection. This is the core of BettaFish's power.

---

### Week 4 — Structured Output

**Goal**: Agent produces a validated, structured digest instead of raw Markdown

1. Define your output schema (simpler than BettaFish's full IR — e.g. `{summary, key_themes, top_posts, sentiment}`)
2. Add a `FormattingNode` that asks the LLM to populate that schema as JSON
3. Copy the validation + repair loop from Phase 6.2
4. Write a simple renderer (even just a Jinja2 HTML template is fine)

**End of Week 4**: Full pipeline: crawl → DB → agent research → structured output → rendered digest.

---

### (Optional) Week 5 — Multi-Agent

Only tackle this after Week 4 is working well.

1. Split into two agents: "web search agent" + "database agent"
2. Copy `ForumEngine/monitor.py` pattern — but use a shared in-memory queue (simpler than log files for single-process)
3. Copy `ForumEngine/llm_host.py` — LLM moderator reads both agents' outputs and guides the next round

---

## Quick Reference: Key Patterns to Copy

| Pattern | Location in BettaFish | When to use |
|---------|----------------------|-------------|
| LLM client with retry | `InsightEngine/llms/base.py` | Any LLM call in production |
| Exponential backoff | `utils/retry_helper.py` | Any external API call |
| State dataclass | `InsightEngine/state/state.py` | Any multi-step pipeline |
| Node pattern | `InsightEngine/nodes/base_node.py` | Any single processing step |
| Reflection loop | `InsightEngine/agent.py` | Iterative search/refinement |
| Keyword optimizer | `InsightEngine/tools/keyword_optimizer.py` | Before every DB/API query |
| Async DB session | `InsightEngine/utils/db.py` | PostgreSQL async access |
| Tool → DBResponse | `InsightEngine/tools/search.py` | Agent tool interface |
| Clustering sampler | `InsightEngine/agent.py` | When results > LLM context |
| JSON validation+repair | `ReportEngine/nodes/chapter_generation_node.py` | Any structured LLM output |
| Two-stage crawl | `MindSpider/BroadTopicExtraction/` | Before running heavy crawlers |
| Cache layer | `MediaCrawler/cache/` | Any repeated network requests |

---

## Glossary of BettaFish Terms

| Term | Meaning |
|------|---------|
| **State** | Typed dataclass passed through all nodes; holds the full task context |
| **Node** | Single processing step (one LLM call or one tool call) |
| **Reflection** | LLM reads current summary, identifies gaps, generates a new search query |
| **Tool** | A function the agent can call to get external data (DB query, web search) |
| **IR** | Intermediate Representation — structured JSON that sits between LLM output and the renderer |
| **ForumEngine** | Background process that monitors agent logs and generates cross-agent guidance |
| **MindSpider** | The crawling subsystem; runs separately from the agents |
| **DBResponse** | Wrapper returned by all DB tools: `{tool_name, parameters, results, metadata}` |
| **Keyword Optimizer** | Small LLM call that expands a query into multiple search terms before searching |
