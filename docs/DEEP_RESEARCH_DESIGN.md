# Deep Research Design Overview

This document consolidates the detailed explanations from previous discussions into a single design overview of the DeepResearch agent. It captures the reasoning cycle, main components, and the flow used when answering a user query.

## Purpose

DeepResearch aims to repeatedly search, read webpages, reflect on questions, and reason over gathered knowledge until it confidently provides an answer or the token budget is exhausted. The emphasis is on iterative improvement of the answer rather than producing long-form content.

## Core Reasoning Cycle

The reasoning logic lives in `src/agent.ts`. The entry function `getResponse` maintains a token budget and a list of unanswered sub-questions known as **gaps**. The loop proceeds while token consumption stays under about 85% of the budget (with the remainder saved for the final step).

1. **Initialization**
   - Set up token tracking and extract any URLs from the incoming messages.
   - Reserve budget for an optional "Beast Mode" step at the end.
2. **Main Loop**
   - Pick the current question from `gaps` and decide which evaluations or tools may be required.
   - Generate a prompt via `getPrompt`, combining diary context, known knowledge snippets, available URLs, and which actions are permitted in this step.
   - The language model is called through `ObjectGeneratorSafe.generateObject` to produce structured JSON describing the next action and a `think` field with reasoning text.
3. **Execute Action**
   - Based on the chosen action, the agent does one of the following:
     - **answer** – Attempt to directly answer the question. If it is the first attempt and answer bans are not triggered, it might finish early. Otherwise `evaluateAnswer` checks if the answer is definitive. Rejected answers are stored for context and the loop continues.
     - **reflect** – Generate new sub-questions. Unique ones are added back into the `gaps` queue.
     - **search** – Deduplicate search queries, optionally rewrite them via `rewriteQuery`, execute them, and store found URLs plus any snippet knowledge.
     - **visit** – Retrieve webpage content using `processURLs` and store new knowledge for later steps.
     - **coding** – Run the coding tool via `CodeSandbox` to produce or debug code, storing successful outputs as knowledge.
   - After each action, the context is saved for debugging and the loop pauses briefly before the next iteration.
4. **Beast Mode**
   - If the loop exits without a verified answer, the agent enters Beast Mode. Most actions are disabled except answering. It consumes the remaining budget to force a final response and then performs final cleanup steps, including formatting and reference building.

## High-Level Flow

A flowchart in the README depicts the algorithm:

```
Query -> Search -> Read -> Reason -> Search ... [loop] -> Answer
```

This cycle continues until a satisfactory answer is produced, or Beast Mode is triggered when the token budget is nearly exhausted.

## Utilities and Supporting Modules

DeepResearch relies on a collection of supporting modules that each handle a
specific aspect of the overall workflow.  Conceptually these helpers keep the
core loop small while allowing specialized logic to live in isolated files.

### Query and Search Helpers

- **`query-rewriter.ts`** – expands a user query from several cognitive
  perspectives (skeptic, analyst, historian, etc.) and also adds recency-focused
  variations. This diversification drives more comprehensive search results.
- **`jina-search.ts`**, **`brave-search.ts`** and **`serper-search.ts`** –
  implement a common interface to different search engines. Each wrapper
  normalizes the external API so the agent can swap providers without changing
  the main logic.
- **`read.ts`** – fetches web pages through the Jina Reader service, returning
  the page text and optional link or image summaries. Centralizing network reads
  makes it easy to track tokens and deduplicate URLs.

### Content Processing

- **`reducer.ts`** – merges multiple partial answers into a single article by
  removing duplicate sentences and selecting the strongest version of each idea.
- **`research-planner.ts`** – breaks a broad topic into orthogonal sub-problems
  so multiple bots (or iterations) can tackle them independently.
- **`finalizer.ts`** – polishes a draft answer into a concise, opinionated style
  as the last step before returning the result.
- **`build-ref.ts`** – analyzes the generated answer against collected web
  content, using embeddings to locate the most relevant quote snippets for
  citation.

### Diagnosis and Feedback

- **`evaluator.ts`** – checks an answer for qualities such as definitiveness or
  attribution and returns structured feedback.
- **`error-analyzer.ts`** – reviews the entire reasoning trail when repeated
  failures occur and proposes a new strategy to escape dead ends.

### Infrastructure and Tracking

- **`action-tracker.ts`** – records each chosen action and the agent's internal
  "think" text so a chronological log can be reviewed later.
- **`token-tracker.ts`** – aggregates token usage from every tool and enforces
  the configured budget.
- **`safe-generator.ts`** – wraps language-model calls in schema validation so
  tool outputs are always well formed.
- **`schemas.ts`** – contains the JSON schema definitions used across the
  project for actions, evaluations and references.

These helpers isolate complex behavior from the reasoning loop and make the
system easier to extend in future iterations.

The repository includes several helper modules to assist the main loop:

- `src/tools/query-rewriter.ts` rewrites search queries from different perspectives and time filters.
- `src/tools/read.ts` fetches webpages via Jina Reader.
- `src/utils/finalizer.ts`, `src/utils/reducer.ts`, and `src/utils/research-planner.ts` help refine answers, merge partial results, and break down complex research tasks.
- `src/app.ts` provides an OpenAI-compatible HTTP endpoint for using the agent as a service.
- `src/tools/jina-search.ts`, `brave-search.ts`, and `serper-search.ts` offer interchangeable search providers.
- `src/utils/action-tracker.ts` and `token-tracker.ts` record each reasoning step and track token consumption.
- `src/tools/error-analyzer.ts` summarizes failures when repeated bad answers occur.

## Iterative Design Considerations

- **Token Budgeting** – The agent carefully tracks token usage, reserving space for a final forced answer if needed.
- **Self-Evaluation** – Failed answers or repeated sub‑questions feed back into the context so the model can adjust.
- **Search and Visit Deduplication** – Duplicate queries and URLs are suppressed to use the budget efficiently.
- **Extensible Actions** – The action model (`search`, `visit`, `answer`, `reflect`, `coding`) allows integration of more tools in future iterations.

## Summary

DeepResearch follows a disciplined loop of planning, searching, reading, reflecting, and answering, backed by strict token accounting. The design enables incremental improvements and provides a clear structure for further enhancements.


## Evaluation and Token Tracking

- **TokenTracker** logs usage from every model call to enforce the current budget and provide usage breakdowns.
- **ActionTracker** stores each step's chosen action and reasoning. It also collects "think" traces for later debugging.
- Answers are validated by the evaluator tool against metrics like `definitive`, `attribution`, `freshness`, and `completeness`. Failed evaluations feed back into the diary context so the next iteration can improve.
- When repeated failures occur, the error analyzer summarizes mistakes and suggests new strategies.

## Server and Configuration

The project ships with `src/app.ts`, an OpenAI-compatible HTTP API exposing `/v1/chat/completions`. Environment variables in `config.json` specify API keys and default providers:

- `GEMINI_API_KEY` or `OPENAI_API_KEY` for the reasoning LLM.
- `JINA_API_KEY`, `BRAVE_API_KEY`, or `SERPER_API_KEY` for search providers.
- `DEFAULT_MODEL_NAME`, `search_provider`, and `llm_provider` control which model and search engine are used.

The same configuration also defines model temperatures and token limits for the various tools (e.g. finalizer, query rewriter, agent beast mode).


## Repository Structure and CLI

Most logic lives inside the `src/` directory. Key entry points include:

- `agent.ts` – implements the iterative reasoning loop.
- `app.ts` and `server.ts` – provide an Express service exposing an OpenAI
  compatible `/v1/chat/completions` endpoint with streaming support.
- `cli.ts` – a command line wrapper. Running `npx deepresearch "question"`
  triggers the same cycle locally.

Jest tests reside in `src/__tests__` where network calls are mocked so the agent
behavior can be validated without external APIs.

## Evaluation Metrics

The evaluator checks each answer for several qualities:

- **definitive** – confident wording with no hedging.
- **attribution** – cited statements must include URLs and quotes.
- **freshness** – time sensitive questions should include recent references.
- **completeness** – multi-part questions require all parts to be covered.
- **strict** – used when repeated failures occur and the agent must provide an
  improvement plan.

Each failed metric decrements its `numEvalsRequired` counter until either it
passes or Beast Mode forces a final answer.

## Using DeepResearch From Other Bots

Agents can leverage this project in two ways:

1. Call the HTTP API from `server.ts`. Requests follow the OpenAI
   `chat/completions` schema and return streaming delta chunks. Use the model
   name `jina-deepsearch-v1`.
2. Import `getResponse` or run the CLI directly. Both methods honor the
   environment variables described above to select search and language model
   providers.

With these options, other bots can delegate complex research tasks to
DeepResearch while retaining control of their own conversation flow.
