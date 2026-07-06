# Obsidian Notekit

Obsidian Notekit is a planned set of small Go libraries for managing a Markdown note vault. It is built for people who use Obsidian as their main knowledge base and want better tools for reviewing inbox notes, organizing notes into useful categories, finding old material, ingesting links, and keeping related notes connected.

The project treats an Obsidian vault as plain files on disk. It does not require a database, a hosted service, or a custom note format. The goal is to add structure around the vault while keeping the notes readable, editable, and safe to manage by hand.

## Who this is for

This project is for people who:

- Keep long-lived notes in Obsidian or another Markdown-based system
- Capture many notes into an inbox and need help reviewing them later
- Want category paths, index notes, backlinks, and link suggestions without giving up control of their files
- Want to use lower-cost LLMs through OpenAI-compatible APIs, including providers such as OpenRouter
- Prefer libraries that can be wrapped by a CLI, service, scheduled job, or local agent
- Care about dry runs, reviewable changes, audit logs, and deterministic validation

It is not meant to replace Obsidian. It is meant to work beside it.

## What it helps with

Obsidian Notekit is designed around a few common note-management problems.

### Inbox review

Many notes start as quick captures. Some are only a title, a rough thought, or a copied link. The system can scan an inbox folder, inspect each note, and propose where it should live, what categories it belongs to, and which existing notes it may relate to.

### Link ingestion

When a note contains a URL, the system can fetch or accept externally fetched content, extract readable text, summarize it, and propose a richer note. The original source URL, fetch metadata, extracted text, and summary provenance can be retained for review.

### Hierarchical categorization

The system can classify notes into several three-level taxonomy paths. For example, a note may belong under more than one path if it is useful from different points of view. Accepted taxonomy paths are managed in a small vocabulary file, while new paths are proposed for review instead of being silently added.

### Search and rediscovery

The search layer starts simple: fielded keyword search over titles, paths, note body, frontmatter, links, and taxonomy. Search results should explain why they matched, not just return a score. More advanced search providers, such as semantic search or external indexes, can be added later behind the same interface.

### Orphan and weakly connected notes

The system can identify notes with few or no links, notes that do not share categories with the rest of the vault, and notes that may be stale. It should not delete notes automatically. It should produce reviewable proposals for linking, categorizing, archiving, or pruning.

### Index notes

The system can propose index notes that point to related notes by category, topic, source type, or other rules. Generated index notes should be repeatable, reviewable, and safe to update without overwriting hand-written content.

## Benefits

### Keeps the vault usable in Obsidian

The project works with regular Markdown files and Obsidian-style links. Metadata can live in a sidecar state directory by default, with frontmatter writes treated as opt-in projections. This avoids unnecessary churn in human-edited notes.

### Review before write

The default workflow is proposal first. The system can suggest moves, links, category changes, summaries, and index-note updates, but those changes are written only after validation and approval.

### Small libraries instead of one large app

Each major capability is designed as a library with a narrow interface. That makes it easier to test, easier to wrap with a CLI, and easier to use from an agent or service.

### Minimal dependency posture

The core design favors the Go standard library and simple file formats. Optional providers can add more powerful indexing, extraction, or LLM behavior later without changing the core contracts.

### Agent-friendly by design

Outputs are meant to be structured, inspectable, and traceable. An agent should be able to run a task, check validation errors, adjust its inputs, and retry without guessing what went wrong.

### Safe around personal notes

The system should avoid destructive actions by default. It should ignore its own sidecar state, avoid scanning Obsidian internals, protect secrets, validate file paths, and make audit records for proposed and applied changes.

## High-level design

The system is split into composable modules. A CLI or service can wire these modules together, but the modules should not depend on a specific user interface.

```text
Markdown vault
    |
    v
Vault scanner
    |
    v
Parser and note model
    |
    +--> Link graph
    +--> Taxonomy classifier
    +--> Search index
    +--> URL ingestion
    +--> Orphan analysis
    +--> Index note builder
              |
              v
        Proposed changes
              |
              v
        Validation and review
              |
              v
        Safe writer and audit log
```

### Vault scanner

The scanner walks the vault, applies ignore rules, and returns candidate Markdown files. It should skip `.obsidian/`, `.notekit/`, trash folders, generated caches, and configured ignore paths.

The scanner should support a simple fast path based on modified time and file size, with content hashing for changed files and conflict detection.

### Note parser

The parser reads Markdown files and produces a normalized note model. The model should include the vault-relative path, title, frontmatter fields, body text, outbound links, tags, source URLs, and basic file metadata.

The parser should preserve the original file bytes when possible so later writers can avoid unnecessary formatting changes.

### Identity and state

The human-facing note address is the vault-relative path. The system may also maintain stable internal note IDs in sidecar state for audit trails and proposal references.

Frontmatter IDs can be honored only when explicitly configured and unique. Content hashes are used for change detection and best-effort rename or move inference, not as the only identity source.

### Sidecar state

Small system state lives under a single sidecar directory, such as `.notekit/` at the vault root. This can hold indexes, proposal queues, taxonomy files, audit records, and other metadata.

Large raw fetch caches should be disabled by default or stored outside the vault. Retaining raw fetched content should be opt-in, size-capped, and time-limited.

### Proposal queue

Actions are represented as proposed changes. A proposed change might move a note, add links, apply taxonomy paths, create a summary note, or update an index note.

The canonical proposal format should be structured JSON or JSONL. Markdown review notes can be rendered for convenience, but they should be treated as read-only reports in the first version unless a dedicated review adapter is added later.

### Taxonomy manager

The taxonomy manager owns the accepted category paths. The default model is soft-closed: notes can only be assigned to accepted paths, but the system can propose new paths when the current taxonomy is not enough.

A machine-readable file such as `.notekit/taxonomy.json` should be the canonical source. A rendered `Taxonomy.md` can be generated for human review.

### LLM client

The LLM client talks to an OpenAI-compatible endpoint. It should be provider-neutral and should support model name, base URL, API key source, timeout, token budget, retry policy, and structured output contracts.

The system should validate every model response before using it. Invalid responses should produce clear validation errors that an agent or user can inspect.

### URL ingestion

URL ingestion is split into fetching, extraction, and note proposal.

- The fetcher gets raw content from a URL or returns a typed error.
- The extractor turns raw content into readable text and metadata.
- The ingestor builds a proposed note, summary, source metadata, and category suggestions.

Fetching is pluggable. A default HTTP fetcher can handle normal web pages, while external tools or agents can provide already-fetched content for sources that need special handling.

### Search provider

The first search provider can be a simple local inverted index. It should support title, path, body, frontmatter, link, and taxonomy fields. Results should include snippets, matched fields, scores, and human-readable reasons.

The interface should allow later providers for fuzzy search, semantic search, embeddings, or full-text engines.

### Link graph

The graph module resolves links between notes and identifies backlinks, missing targets, orphan notes, weakly connected notes, and candidate relationships.

The graph should work from parsed Markdown links, Obsidian wikilinks, tags, taxonomy paths, and optional model-generated link suggestions.

### Index note builder

The index note builder creates or updates notes that collect related notes by topic, category, source, or rule. It should write only to managed sections so hand-written content remains safe.

### Validator

The validator checks proposed changes before anything is written. It should verify paths, taxonomy paths, link targets, frontmatter write rules, managed-section boundaries, trace IDs, and required metadata.

Validation errors should be structured so a user or agent can fix the input and retry.

### Writer

The writer applies approved changes. It should support dry-run mode, atomic writes, backups or snapshots, conflict checks, and audit events.

Frontmatter writes should be opt-in. When enabled, the writer should only touch configured managed keys and should refuse to write if the frontmatter block cannot be safely located.

### Audit and observability

Every run should have a trace ID. Proposals, validation results, applied changes, skipped files, fetch attempts, LLM calls, and errors should be recorded in a small audit log.

Logs should be useful to a person reading them, but structured enough for a tool or agent to inspect.

## Design principles

- Plain files first
- Dry run by default
- Proposals before writes
- Sidecar metadata before frontmatter mutation
- Small interfaces
- Deterministic validation
- Human review for destructive or high-impact changes
- No automatic deletion
- Minimal required dependencies
- Useful without a hosted service

## Project status

This repository is currently intended as a design and implementation workspace. The first milestone should define the module contracts, data model, proposal format, validation rules, and acceptance tests before implementation begins.

## Possible first milestone

A useful first milestone would include:

1. Vault scanning and Markdown note parsing
2. Sidecar state layout
3. Basic note identity and content hashing
4. Proposal records and dry-run validation
5. Soft-closed taxonomy file
6. Simple keyword search provider
7. Link graph construction
8. Audit log with trace IDs

That milestone would not need LLM calls, URL fetching, or automatic writing. Those can be layered in after the core contracts are stable.

## License

BSD 2
