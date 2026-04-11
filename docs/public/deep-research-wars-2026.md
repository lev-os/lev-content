# The Deep Research Wars: 40+ Companies, $1B in Funding, and Everyone Claims #1

*March 2026*

---

I just ran the same research query through six different backends simultaneously. Brave, Valyu, Perplexity, Exa, arXiv, and HackerNews returned 185KB of structured JSON in under 30 seconds. No browser tabs. No copy-pasting. One command:

```
timetravel search "deep research agents" -s deep
```

That command hit three paid APIs in parallel, merged the results, and handed me a deduplicated source list with confidence scores. It cost less than a dollar.

This is the state of deep research in 2026. And it's absolute chaos.

---

## Everyone is #1 (on their own benchmark)

Here's the dirty secret of the deep research space: every single company claims to be the best. And they're all telling the truth -- on the benchmark they chose.

| Company | "We're #1 on..." | Score |
|---|---|---|
| MiroThinker | BrowseComp | 88.2% |
| CellCog Max | DeepResearch Bench | 56.13 |
| Linkup | SimpleQA | 91.0% F1 |
| Tongyi | Humanity's Last Exam | 32.9% |
| Valyu | ScholarQA | 74.5% |
| Writer | GAIA Level 3 | 61% |
| Perplexity | DRACO | 67.15% |
| You.com | DeepSearchQA | 83.67% |
| Brave | AIMultiple Agent Score | 14.89 |
| Parallel | Their own DeepResearch Bench | 82% win rate |

Ten companies. Ten benchmarks. Ten winners.

The pattern is obvious once you see it: Perplexity created DRACO and won it. Parallel runs their own DeepResearch Bench and wins it. Valyu picked the academic benchmark where proprietary data access matters. MiroThinker optimized specifically for BrowseComp's multi-hop navigation.

This is the AI equivalent of every restaurant claiming "best pizza in New York" -- they all have a certificate on the wall and none of them agree on what "best" means.

## The Three Layers Nobody Talks About

The market has silently split into three distinct layers, and most coverage treats them as one thing:

**Layer 1: Search Infrastructure** -- The pipes. Brave, Exa, Tavily, Linkup, Serper, SerpAPI. They return raw results. You bring your own brain. This is the layer Google is suing people over (SerpAPI got hit with DMCA in December 2025 for scraping at "hundreds of millions" of daily requests).

**Layer 2: Research Agents** -- The brains. Perplexity, Manus, Tongyi, Genspark, MiroThinker. They search, read, think, and synthesize. They give you a report, not a list of links. This is where the $24B valuations live.

**Layer 3: Agent Search APIs** -- The middleware. Valyu, Parallel, Firecrawl, You.com. They exist specifically so that other AI agents can search the web programmatically. This is where the actual infrastructure money is flowing -- Tavily got acquired for $275-400M after raising only $25M.

The interesting companies are the ones that span layers. Valyu does Layer 1 (search API) but also Layer 2 (deep research with synthesis). Parallel does Layer 3 (agent API) but benchmarks against Layer 2 players.

## The China Speed Run

While Western companies spent 2025 raising $100M rounds and filing benchmarks, Chinese labs did something different: they shipped.

Alibaba's Tongyi DeepResearch is Apache 2.0, 30B parameters, and it beats OpenAI's o3 on Humanity's Last Exam. It's free.

Moonshot's Kimi K2.5 coordinates 100 sub-agents in parallel ("Agent Swarm") and costs $0.60 per million input tokens. Claude Opus costs $5. That's an 8x price gap for competitive benchmark scores.

DeepSeek is building a multilingual search engine to compete with Google and Perplexity simultaneously, at projected costs of $0.10-0.30 per million tokens -- 50x cheaper than GPT-5.2.

ByteDance open-sourced DeerFlow 2.0, which hit #1 on GitHub Trending. Each agent runs in an isolated Docker container with its own filesystem. It's model-agnostic -- you can point it at OpenAI, Anthropic, Google, or DeepSeek.

Zhipu's GLM-5 (MIT license, 744B MoE) was trained entirely on Huawei Ascend chips, proving you don't need NVIDIA to build frontier models. Their stock is up 250% since IPO.

The Western response? Acquisitions. Meta bought Manus for ~$2B. Nebius bought Tavily for $275-400M. Elastic bought Jina AI. When you can't out-ship, you buy.

## The Only Moat That's Real

After researching 40+ companies in this space, one pattern is clear: **proprietary data access is the only defensible moat.**

Every web search tool -- Brave, Exa, Perplexity, Parallel, SearXNG -- is searching the same public web. They might index it differently, rank it differently, or synthesize it differently. But the underlying corpus is identical.

Valyu is the only company with licensed access to 50+ proprietary sources: SEC/EDGAR filings, paywalled academic papers from PubMed and bioRxiv, clinical trial data, NCBI genomics, financial data feeds, and patent databases. Nobody else has this. Not Perplexity. Not Exa. Not Google.

This matters because the highest-value research questions -- "What does the Phase III trial data show for this drug candidate?" or "How does this company's 10-K risk section compare to last quarter?" -- require data that doesn't exist on the public web.

Open source can replicate search. Open source can replicate synthesis. Open source cannot replicate licensed data partnerships.

## What We Built Instead

Lev's approach to this problem is different from all 40+ companies above. We didn't build a search engine or a research agent. We built a **research router**.

`timetravel` is a plugin with 12 adapters: Brave, Firecrawl, Valyu, Perplexity, Exa, Tavily, Grok, arXiv, HackerNews, ScrapECreators, and an Oracle synthesizer. Each adapter is a thin HTTP client. The router picks a strategy (quick, full, deep, max, social, academic), fans out to the right backends in parallel, and merges the results.

```
timetravel status

  Adapter        Available  Capabilities
  brave           Y          search
  firecrawl       Y          search, extract
  valyu           Y          search, deepsearch, context
  perplexity      Y          search, synthesize
  exa             Y          search
  tavily          Y          search
  grok            Y          synthesize
  arxiv           Y          search
  hackernews      Y          search
  scrapecreators  Y          search

  10/11 adapters available
```

The insight: no single backend wins every query. Valyu crushes academic research but can't search Twitter. Grok has real-time X data but can't search PubMed. Brave is fast and free but shallow. Perplexity synthesizes well but costs per token.

A router that picks the right backends per query type will always outperform a single monolithic research agent. The strategies are defined in code, not in a prompt:

```typescript
const STRATEGIES = {
  quick:    { adapters: ['brave', 'tavily'], mode: 'first-success' },
  deep:     { adapters: ['valyu', 'perplexity', 'exa'], mode: 'parallel', synthesizer: 'oracle' },
  max:      { adapters: ['brave', 'firecrawl', 'perplexity', 'exa', 'valyu'], mode: 'parallel', synthesizer: 'oracle' },
  social:   { adapters: ['grok', 'hackernews', 'scrapecreators'], mode: 'parallel' },
  academic: { adapters: ['arxiv', 'exa', 'perplexity'], mode: 'parallel' },
};
```

When a new backend launches that beats Exa on structured extraction, we add one adapter file and it's available to every strategy. When Valyu adds a new data source, every deep research query automatically gets it. The router doesn't care about benchmark wars -- it uses the best tool for each job.

This is the bet: the winning architecture for AI research isn't one model doing everything. It's a router that treats search backends like interchangeable parts, picks the right combination per query, and lets the best tool win each sub-task.

## The Uncomfortable Prediction

Within 12 months:
- At least two more acquisitions in the search API layer (Linkup and Exa are the obvious targets)
- Google's SerpAPI lawsuit will force every scraping-based API to either build a proprietary index or die
- Chinese models will be the default for cost-sensitive deep research (DeepSeek at $0.10/M tokens is just too cheap)
- Open-source agents (Tongyi, DeerFlow, MiroThinker) will match proprietary ones on benchmarks but lag on proprietary data access
- The "deep research" feature will become table stakes in every LLM -- the way "web search" already is

The companies that survive will be the ones with either (a) proprietary data nobody else has, or (b) infrastructure so deeply embedded in the agent toolchain that switching costs are real.

Everyone else is fighting over who gets the best score on a benchmark nobody agreed to standardize.

---

*This research was conducted using Lev's timetravel plugin, which hit Brave, Valyu, Perplexity, Exa, and 6 other backends in parallel. Total cost: approximately $2.50. Total time: 35 seconds for the structured data, plus the thinking.*
