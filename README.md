# Reddit Insight Radar

A weekly content intelligence workflow built with **n8n, Reddit RSS, OpenAI, and Slack**.

The system monitors high-signal Reddit discussions across selected technical topics, evaluates each post using both **topic relevance** and **Reddit's weekly ranking signal**, generates concise AI-powered insights, and delivers a curated digest directly to Slack.

The current workflow monitors:

- **n8n**
- **Automation**
- **AI Engineering**

Each weekly run selects **4 posts per topic**, producing a final digest of **12 curated Reddit insights**.

---

## Problem

Reddit is one of the most useful sources for identifying recurring questions, emerging tools, technical pain points, market demand, and topics that are actively being discussed by practitioners.

The problem is operational.

Manually searching multiple topics, reviewing dozens of posts, identifying what is actually relevant, removing duplicates, extracting useful signals, summarizing the discussion, and organizing everything into a digest is repetitive and difficult to maintain consistently.

For content, research, and technical learning workflows, this creates a gap between the amount of useful information available and the amount of information a person can realistically review every week.

---

## Solution

I built an automated Reddit research pipeline in **n8n** that turns weekly Reddit discussions into a structured Slack digest.

The workflow:

1. runs automatically once per week;
2. creates dedicated search queries for each monitored topic;
3. retrieves the top weekly Reddit results through public RSS feeds;
4. processes each topic sequentially to reduce request bursts;
5. removes non-post RSS results;
6. normalizes the relevant Reddit post data;
7. scores each candidate using topical relevance and its weekly Reddit position;
8. prevents the same Reddit post from being selected for multiple categories whenever possible;
9. selects the top 4 posts for each topic;
10. sends the 12 selected posts to an OpenAI model for insight generation;
11. formats the results for Slack;
12. delivers the final digest automatically to a dedicated channel.

The result is a repeatable research system that converts public Reddit discussions into a compact weekly source of ideas, technical signals, and topics worth investigating.

---

## Architecture

```text
Schedule Trigger
      │
      ▼
Workflow Configuration
      │
      ▼
Build Topic Queue
      │
      ▼
Loop Over Topics
      │
      ├── RSS Read
      │      │
      │      ▼
      │   Filter Reddit Posts
      │      │
      │      ▼
      │   Extract Post Data
      │      │
      │      ▼
      │   Wait Before Next Topic
      │      │
      └──────┘

      │ done
      ▼
Calculate Relevance Score
      │
      ▼
Sort
      │
      ▼
Get Top 4 Per Topic
      │
      ▼
OpenAI Insight Generation
      │
      ▼
Format Slack Message
      │
      ▼
Slack Delivery
```

The topic loop replaces multiple duplicated branches with one reusable processing path. Adding another monitored topic only requires extending the topic configuration instead of creating another full RSS-processing branch.

---

## Data Collection

The workflow uses **public Reddit RSS search feeds** rather than the Reddit Data API.

Each topic receives its own search query and weekly result set.

Current topics:

```text
n8n
Automation
AI Engineering
```

The workflow retrieves up to **30 weekly candidates per topic** before the final ranking stage.

Because RSS does not provide the complete engagement dataset available through the Reddit Data API, this project does **not** claim exact upvote or comment-based scoring.

Instead, the system uses Reddit's weekly ordering as one ranking signal and combines it with a separate relevance score calculated inside the workflow.

---

## Sequential Topic Processing

The Reddit searches are processed through a **Loop Over Items** pattern with one topic handled at a time.

```text
Topic 1
→ RSS request
→ filtering
→ normalization
→ controlled wait

Topic 2
→ RSS request
→ filtering
→ normalization
→ controlled wait

Topic 3
→ RSS request
→ filtering
→ normalization
```

This design was implemented after parallel RSS requests produced HTTP `429` responses.

The loop keeps the workflow compact while also reducing simultaneous requests and making the collection stage more resilient to temporary rate limiting.

---

## Post Filtering and Normalization

RSS results can contain entries that are not individual Reddit posts.

Before ranking, the workflow filters the feed and keeps only URLs representing actual Reddit discussions.

Each valid result is then normalized into a predictable internal structure containing fields such as:

```text
topic
title
reddit_link
published_at
author
text
rss_rank
engagement_signal
```

This gives the downstream nodes a consistent schema regardless of differences in the original RSS payload.

---

## Relevance Ranking

Selecting the first four RSS results was not enough.

Related technical searches can surface posts that are highly ranked on Reddit but only weakly connected to the intended topic. To improve the final selection, the workflow calculates a **topic relevance score** before choosing the final posts.

The scoring logic evaluates terms found in the post title and available text.

For example, the **AI Engineering** category can prioritize discussions containing terms such as:

```text
AI engineer
AI engineering
RAG
LangGraph
LangChain
embeddings
LLM
vector database
OpenAI
AI agents
```

The final ranking combines:

```text
Topic relevance
+
Reddit weekly position
```

Topic relevance receives the stronger weight, while the RSS position remains a secondary signal.

This allows a technically relevant post to outrank a higher RSS result that is only loosely related to the monitored topic.

---

## Duplicate Control

The same Reddit post can appear in more than one search because the monitored topics overlap.

Before the final selection, the workflow normalizes Reddit URLs and tracks links that have already been selected.

The goal is to return:

```text
4 n8n posts
4 Automation posts
4 AI Engineering posts
=
12 curated posts
```

without repeatedly delivering the same discussion across different categories.

---

## AI Insight Generation

After ranking, the final 12 posts are sent individually to an OpenAI model.

The model receives the available structured data for each selected discussion, including:

- topic;
- post title;
- post text available through RSS;
- weekly Reddit ranking signal;
- publication date.

The prompt enforces a concise and predictable output.

Each response must be a single paragraph that explains:

- what the discussion is about;
- why it matters to people interested in the topic;
- what pain point, opportunity, content idea, or market signal can be extracted from it.

The model is explicitly instructed not to invent:

- upvote counts;
- comment counts;
- engagement metrics;
- facts that are not present in the provided Reddit data.

This keeps the LLM focused on interpretation rather than fabricating missing platform data.

---

## Slack Delivery

The final results are sent automatically to a dedicated Slack channel.

The digest is organized into three sections:

```text
🧩 n8n
⚙️ Automation
🤖 AI Engineering
```

Each selected post contains:

```text
Title
Signal
Published date
Insight
Reddit link
```

Posts are sent individually rather than concatenated into one large Slack message.

This keeps each insight visually separated and allows Slack to associate the Reddit link preview with the corresponding post.

![Slack output](images/slack-output.png)

*Weekly Reddit insight digest delivered automatically to Slack.*

---

## Workflow Configuration

Operational values are centralized in a configuration node instead of being distributed across the workflow.

Example:

```text
timeframe: week
limit: 30
top_per_topic: 4
total_results: 12
slack_channel: #reddit-insights
```

The topic definitions and Reddit search queries are managed separately in the `Build Topic Queue` node.

This makes the workflow easier to maintain and prevents configuration values from becoming tightly coupled to individual processing nodes.

---

## Reliability and Engineering Decisions

### Dynamic Topic Loop

The first version of the workflow used separate branches for each Reddit topic.

That approach duplicated RSS, filtering, and extraction nodes.

The final architecture uses a single reusable loop:

```text
Topic Queue
→ Loop
→ RSS
→ Filter
→ Extract
→ Wait
→ Next Topic
```

This reduces duplicated logic and makes the workflow easier to extend.

### Controlled Requests

Parallel RSS requests produced rate-limit responses from Reddit.

The final workflow processes topics sequentially and introduces a controlled delay between requests.

### Relevance Before Selection

Reddit ranking alone was not enough to guarantee topic quality.

A deterministic JavaScript scoring layer was added before final selection so the workflow considers both platform ranking and topic-specific relevance.

### Duplicate Protection

Related queries can return the same discussion.

Normalized Reddit URLs are used as stable identifiers during the final selection stage to prevent repeated delivery across categories whenever possible.

### Structured LLM Output

The OpenAI prompt uses explicit output constraints so every generated insight follows the same format and can be inserted into Slack without additional interpretation.

### One Slack Message per Post

Each Reddit post is sent as its own Slack message.

This keeps the digest readable and preserves the relationship between the text, link, and Slack-generated Reddit preview.

---

## Tech Stack

- **n8n** — workflow orchestration and scheduling
- **JavaScript** — filtering, normalization, relevance scoring, deduplication, and output formatting
- **Reddit RSS** — public weekly discussion discovery
- **OpenAI API** — insight generation
- **Slack API / OAuth** — automated digest delivery

---

## Repository Structure

```text
reddit-insight-radar/
│
├── workflows/
│   └── reddit-insight-radar.json
│
├── images/
│   └── slack-output.png
│
├── .gitignore
├── LICENSE
└── README.md
```

---

## Setup

### 1. Import the workflow

Import:

```text
workflows/reddit-insight-radar.json
```

into n8n.

### 2. Configure OpenAI

Connect an OpenAI credential to the `Message a model` node.

### 3. Configure Slack

Connect the Slack credential used by the final `Send a message` node.

The Slack app must have permission to post messages to the destination channel.

### 4. Review workflow configuration

Confirm the operational values:

```text
timeframe
limit
top_per_topic
total_results
slack_channel
```

### 5. Review monitored topics

The search topics and queries are defined in:

```text
Build Topic Queue
```

### 6. Test the workflow

Run the workflow manually and validate:

- RSS retrieval;
- topic association;
- final 12 selected posts;
- generated insights;
- Slack formatting and previews.

### 7. Publish the workflow

Publishing enables the configured weekly schedule.

Credentials and secrets are intentionally excluded from this repository.

---

## What This Project Demonstrates

This project demonstrates more than connecting an RSS feed to an LLM.

It combines **workflow orchestration, external data ingestion, deterministic ranking logic, rate-limit handling, state-aware looping, structured LLM integration, OAuth-based Slack delivery, and defensive data processing** in one automated research pipeline.

Key engineering skills demonstrated:

- designing reusable n8n workflow architecture;
- replacing duplicated branches with dynamic loop-based processing;
- working with external RSS data and inconsistent payloads;
- normalizing raw data into a predictable internal schema;
- handling HTTP rate limits through controlled request sequencing;
- implementing deterministic relevance scoring in JavaScript;
- combining platform ranking signals with topic-specific relevance;
- preventing duplicate records through normalized identifiers;
- designing constrained prompts for consistent LLM output;
- mapping AI-generated responses back to their source records;
- formatting multi-item outputs for downstream delivery;
- integrating Slack through OAuth credentials;
- building an automated scheduled pipeline with clear separation between configuration, processing, ranking, AI interpretation, and delivery.

The result is a compact content intelligence system designed around **reliability, maintainability, and clear data flow**, rather than a single AI call attached to an automation.
