# Plan

Working name for the system: **notekit** — a set of small, composable Go libraries for organizing, indexing, ingesting, triaging, and maintaining an Obsidian vault. The name is a placeholder; nothing in the design depends on it.

## 1. Purpose

notekit helps a single user (and AI agents acting on their behalf) manage a local Obsidian vault of Markdown notes. It solves six concrete problems:

1. **Inbox triage** — review rough notes and propose keep/merge/move/expand/link/categorize/prune actions.
2. **Retrieval** — find notes by title, path, body text, tags, frontmatter, taxonomy, and link relationships.
3. **URL ingestion** — turn link-only notes into summary notes with provenance, via a pluggable fetch/extract/summarize pipeline.
4. **Hierarchical categorization** — assign multi-path, three-level taxonomy (`domain / topic / subtopic`) using deterministic validation and optional LLM assistance against a soft-closed vocabulary.
5. **Orphan and island detection** — find weakly connected, duplicated, or stale notes and propose remediation.
6. **Index notes** — generate and idempotently maintain index notes with managed blocks, preserving human-authored content.

The vault is the user's data. The system is **non-destructive by default**: it reads freely, but every mutation is a validated, auditable *proposal* that requires approval before it is applied. Every AI-assisted output is deterministically validated, retryable on failure, and traceable end to end.

## 2. Scope

### In scope (MVP)

- Vault scanning, note parsing, sidecar metadata, and stable note identity.
- Link extraction, link graph, backlinks, cluster/orphan analysis, near-duplicate detection.
- Keyword-oriented search (title/path/body/frontmatter/category/link fields) with prefix matching, boolean filters, snippets, and explainable scoring. **Not semantic search.**
- Soft-closed taxonomy: canonical taxonomy file, deterministic validation, LLM classification into accepted paths, and taxonomy-change proposals for new paths.
- Inbox triage engine producing structured proposals.
- URL detection and a pluggable ingestion pipeline with a default stdlib HTTP fetcher, typed failure taxonomy, and an "externally provided content" path for agent-fetched content.
- Summary note composition (draft form, proposal-only).
- Proposal/review queue persisted as JSONL/JSON under `.notekit/proposals/`, with optional read-only Markdown review reports.
- Index note generation with managed blocks.
- LLM client adapter (OpenRouter / OpenAI-compatible), prompt versioning, structured output contracts, hand-written typed validators, retry loop, budgets, tracing.
- Audit log, structured logging, trace/correlation IDs, requirement traceability.
- Configuration loading and validation.

### Out of scope (MVP) / future extensions

- **Future:** embeddings/semantic search, fuzzy matching, external search engines (as `SearchProvider` implementations); a `ReviewAdapter` that parses approvals from Obsidian notes; file watching / daemon mode; deep media analysis of attachments; X/Twitter fetching beyond a stub adapter; multi-user or sync-aware coordination; databases, vector stores, queues, or web services.
- **Never (by design):** automatic deletion, automatic overwrite of human content, silent frontmatter rewriting.

## 3. Assumptions

| ID | Assumption |
| --- | --- |
| `ASM-001` | Primary implementation is Go with stdlib-first dependency posture; Python is optional for prototyping only. |
| `ASM-002` | Vault scale ≤ ~50k notes. Full rescan per run is acceptable; an mtime+size fast-path cache skips re-parsing unchanged files. Incremental indexing is a future extension. |
| `ASM-003` | **Identity:** vault-relative path is the canonical human-facing note address. An internal stable note ID lives in sidecar state. Frontmatter IDs are honored only when explicitly configured *and* unique. Content hashes are used for change detection and best-effort rename/move inference, never as the sole identity source. |
| `ASM-004` | **State location:** metadata/state defaults to `<vault>/.notekit/`. Raw/cache payload storage defaults to disabled, or to an external cache directory when enabled. The scanner always ignores `.notekit/`, `.obsidian/`, trash folders, and configured ignore paths. |
| `ASM-005` | **Proposals:** canonical proposal records are JSONL/JSON in `.notekit/proposals/`. Markdown review notes rendered into the vault are read-only reports unless a future `ReviewAdapter` explicitly supports parsing approvals from Obsidian notes. |
| `ASM-006` | **Metadata writes:** sidecar metadata is canonical by default. Frontmatter writes are opt-in projections limited to configured managed keys, only after validation, and only in dry-run/proposal-first mode unless explicitly approved. |
| `ASM-007` | **Taxonomy:** soft-closed. Accepted paths live in a canonical taxonomy file. LLMs may classify into accepted paths and may propose new paths; new paths are not accepted until approved via a taxonomy-change proposal. |
| `ASM-008` | **Fetching:** the `Fetcher` interface is pluggable; a default stdlib `net/http` fetcher is provided. An agent may inject pre-fetched content via the external-content path; it still flows through extraction and validation. The X/Twitter adapter is a stub returning a typed `requires-external-fetch` error. |
| `ASM-009` | **Validation:** structured-output validation uses hand-written typed Go validators (zero third-party deps) emitting the structured `ValidationError` format. Contracts are documented as normative JSON examples, not enforced schema files. |
| `ASM-010` | **Retention:** ingestion retains extracted text, normalized metadata, fetch timestamp, source URL, content hash, content type, extractor version, and summary provenance by default. Raw payload retention is opt-in, size-capped, TTL-controlled, and preferably stored outside the vault. |
| `ASM-011` | Single-writer assumption: at most one notekit process mutates state at a time, enforced by a lock file in `.notekit/`. Obsidian may edit notes concurrently; conflicts are detected via content hash at apply time. |
| `ASM-012` | Notes are UTF-8. Non-UTF-8 files are recorded with a validation warning and excluded from body indexing. |

## 4. Requirements

All requirements are testable. Verification methods: **U** = unit test, **I** = integration test (fixture vault), **C** = contract test, **A** = acceptance scenario (see `AcceptanceTests.md`), **G** = golden file.

### Vault, files, identity (`REQ-VLT-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-VLT-001` | Scan a vault directory tree and identify Markdown notes (`.md`), producing a `VaultSnapshot`. | Foundation for all workflows. | U, I, A |
| `REQ-VLT-002` | The scanner must always exclude `.notekit/`, `.obsidian/`, trash folders (`.trash/`, configured), and configured ignore paths. | State and app internals are not notes. | U, I, A |
| `REQ-VLT-003` | An invalid, missing, or non-directory vault path must fail fast with a structured error; no state is created. | Safety and clear diagnostics. | U, A |
| `REQ-VLT-004` | All file writes use atomic write (temp file in same directory + fsync + rename); partial files are never left at the destination path. | Vault integrity under crash. | U, I |
| `REQ-VLT-005` | Before applying any change to a note, the note's current content hash must match the hash recorded when the proposal was created; mismatch aborts the apply as a write conflict. | Obsidian edits concurrently. | U, I, A |
| `REQ-VLT-006` | Symlinks are not followed outside the vault root by default; escaping paths are rejected. | Path traversal safety. | U |
| `REQ-VLT-007` | Unchanged files (same mtime+size as cached) may skip re-parsing; hash verification is still available on demand. | Performance at ≤50k notes. | U, I |
| `REQ-VLT-008` | Each note carries a stable internal note ID stored in sidecar state, assigned on first sight and persisted across runs. | Durable references for proposals/audit. | U, I |
| `REQ-VLT-009` | Frontmatter IDs are used as identity input only when `identity.frontmatter_key` is configured; duplicate frontmatter IDs produce a validation error and fall back to sidecar identity. | User override with safety. | U, A |
| `REQ-VLT-010` | Rename/move inference: a disappeared path plus an appeared path with identical content hash is recorded as a candidate rename (audited, best-effort), preserving the stable note ID; ambiguous matches are surfaced, not guessed. | Continuity without magic. | U, I, A |

### Markdown parsing and metadata (`REQ-MD-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-MD-001` | Parse YAML frontmatter delimited by `---` fences; extract configured keys, tags, aliases, and title. | Metadata workflows. | U, G, A |
| `REQ-MD-002` | Any write to a note must preserve every byte outside the explicitly managed region (managed frontmatter keys or managed index block); if the managed region cannot be safely located, the write is refused with a validation error. | Never damage human content. | U, G, A |
| `REQ-MD-003` | Notes with invalid or missing frontmatter are still scanned and indexed by body/path, with a validation warning attached. | Robustness on messy vaults. | U, I |
| `REQ-MD-004` | Extract tags from both frontmatter (`tags:`) and inline `#tag` syntax, normalized to a canonical form. | Tag search and orphan checks. | U, G |
| `REQ-MD-005` | Sidecar metadata is the canonical store; frontmatter projection is opt-in, limited to configured managed keys, validated first, and proposal-first unless explicitly approved. | User-locked decision (ASM-006). | U, I, A |

### Search and indexing (`REQ-IDX-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-IDX-001` | Build a lexical index over fields: title, path, body, frontmatter keys/values, tags, taxonomy paths, and link targets. | Multi-path retrieval. | U, I, A |
| `REQ-IDX-002` | Token normalization: Unicode-aware lowercasing and basic folding; prefix matching supported per token. No stemming in MVP. | Keyword search quality with stdlib. | U |
| `REQ-IDX-003` | Query language supports field-scoped terms (`tag:x`, `path:inbox/`, `taxonomy:eng/...`, `link:NoteTitle`), AND/OR/NOT combination, and phrase matching. | Explainable filtering. | U, C, A |
| `REQ-IDX-004` | Results include snippets with match offsets for body hits. | Human and agent review. | U, G |
| `REQ-IDX-005` | Scoring is explainable: each result carries per-field score contributions and the matched terms. | Debuggability; agent trust. | U, C |
| `REQ-IDX-006` | Search is exposed behind a `SearchProvider` interface so semantic/embedding/fuzzy/external providers can be added without changing callers. | Future extension. | C |
| `REQ-IDX-007` | The index is a rebuildable sidecar artifact (JSON/JSONL under `.notekit/index/`); deleting it and re-scanning fully reconstructs it. | Vault remains source of truth. | I |

### URL ingestion (`REQ-ING-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-ING-001` | Detect URLs in note bodies and frontmatter, with position, scheme, and surrounding context; identify "sparse" notes (little content beyond title + URL). | Ingestion trigger. | U, G, A |
| `REQ-ING-002` | Fetching is behind a `Fetcher` interface; the default implementation uses `net/http` only. | Pluggability, minimal deps. | C, U |
| `REQ-ING-003` | Fetch policy enforces: allowed/blocked domain lists, allowed schemes (http/https only by default), robots.txt respect (configurable), max body size, timeout, and a redirect cap with each hop re-validated against policy. | Safety and predictability. | U, I, A |
| `REQ-ING-004` | Unsafe URLs are rejected before any network I/O: disallowed schemes (`file:`, `javascript:`, etc.), private/loopback/link-local IPs and DNS results resolving to them (SSRF defense), and blocked domains. | Security. | U, A |
| `REQ-ING-005` | Fetch failures are typed: `dns`, `timeout`, `tls`, `http-4xx`, `http-5xx`, `auth-required`, `paywall-suspected`, `rate-limited`, `robots-disallowed`, `too-large`, `gone`, `redirect-loop`, `requires-external-fetch`. Each carries retryability and retry-after where known. | Precise agent feedback. | U, C, A |
| `REQ-ING-006` | The `Extractor` interface converts fetched payloads to readable text + normalized metadata; extraction yielding empty/whitespace-only text is a validation error, not a silent success. | No garbage summaries. | U, A |
| `REQ-ING-007` | Default retention per successful ingestion: extracted text, normalized metadata, fetch timestamp, source URL, content hash, content type, extractor name+version, and summary provenance. | User-locked decision (ASM-010). | U, I |
| `REQ-ING-008` | Raw payload retention is opt-in, size-capped, TTL-controlled, and stored outside the vault by default (external cache dir). Expired entries are purged on run. | Disk and privacy control. | U, I |
| `REQ-ING-009` | Externally provided content (e.g., agent-fetched) can enter the pipeline via a dedicated input type carrying its own provenance; it passes through the same extraction/validation as fetched content. | Agent tooling interop. | C, I, A |
| `REQ-ING-010` | The X/Twitter adapter is a stub that returns the typed `requires-external-fetch` error with guidance metadata. | Realistic platform limits. | U |

### Taxonomy (`REQ-TAX-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-TAX-001` | A taxonomy path has exactly three levels (`domain/topic/subtopic`); each segment matches a validated slug grammar (non-empty, normalized case, no path separators). | Structural determinism. | U, A |
| `REQ-TAX-002` | Accepted paths live in a canonical taxonomy file (`.notekit/taxonomy.yaml` by default) with stable path IDs, descriptions, and status (`accepted`, `deprecated`). | Soft-closed vocabulary. | U, I |
| `REQ-TAX-003` | Classification results are deterministically validated: structural grammar first, then membership against accepted paths. | LLM output cannot bypass rules. | U, C, A |
| `REQ-TAX-004` | A classifier may propose new paths; these become `taxonomy-change` proposals and are not usable for note classification until approved. | Controlled vocabulary growth. | U, I, A |
| `REQ-TAX-005` | Classification produces 3–7 candidate paths per note (configurable), each with confidence, rationale, provenance, and validation status. | Reviewable AI output. | U, C, A |
| `REQ-TAX-006` | Malformed candidate paths (wrong depth, empty segments, bad characters, unknown paths presented as accepted) are rejected with structured `ValidationError`s referencing the offending candidate. | Retry loop input. | U, A |
| `REQ-TAX-007` | Manually curated taxonomy assignments on a note are never modified or removed without an explicitly approved proposal; classifier output can only add candidates. | Preserve human curation. | U, I, A |

### Links and graph (`REQ-LNK-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-LNK-001` | Extract wikilinks (`[[Target]]`, `[[Target|Alias]]`, `[[Target#Heading]]`, embeds `![[...]]`) and Markdown links, with byte offsets. | Graph foundation. | U, G, A |
| `REQ-LNK-002` | Resolve link targets using Obsidian-like resolution (exact path, then title/alias match, shortest-path disambiguation); unresolved links are recorded as broken-link records, not errors. | Faithful graph. | U, G, A |
| `REQ-LNK-003` | Build a link graph with forward links and derived backlinks per note. | Backlink search, orphan analysis. | U, I, A |
| `REQ-LNK-004` | Compute connected components over the (undirected) link graph to identify isolated clusters below a configurable size threshold. | Island detection. | U, A |

### Inbox triage (`REQ-INB-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-INB-001` | Enumerate notes in the configured inbox path that lack a `triaged` marker in sidecar state. | Work queue definition. | U, I, A |
| `REQ-INB-002` | For each inbox note, produce a `TriageSuggestion`: recommended action (keep/merge/move/expand/link/categorize/prune), candidate destination, candidate related notes (via search + graph), candidate links, missing metadata, and a human-review-required flag with reason. | Core triage value. | U, C, A |
| `REQ-INB-003` | Triage never mutates notes directly; all recommendations are emitted as proposals. | Non-destructive default. | I, A |

### Orphans, islands, duplicates (`REQ-ORP-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-ORP-001` | Detect notes with no incoming links, no outgoing links, no tags, and/or no taxonomy paths, reporting which criteria matched. | Weak-connection surfacing. | U, I, A |
| `REQ-ORP-002` | Detect isolated graph clusters (REQ-LNK-004) and stale notes (no modification since a configurable age, never reviewed). | Maintenance signal. | U, A |
| `REQ-ORP-003` | Detect exact duplicates (content hash) and near-duplicates (normalized-shingle Jaccard similarity above a configurable threshold). | Merge candidates. | U, G |
| `REQ-ORP-004` | Remediation output (link/categorize/merge/archive/prune) is suggestion-only; no automatic deletion or archival ever occurs. | Safety invariant. | I, A |

### Index notes (`REQ-NDX-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-NDX-001` | Build index notes from a source spec: taxonomy path, tag, folder, graph cluster, saved query, or manual rule. | Flexible grouping. | U, I, A |
| `REQ-NDX-002` | Generated content lives only between managed block markers (`<!-- notekit:begin ... -->` / `<!-- notekit:end -->`); all bytes outside are preserved verbatim. | Human content safety. | U, G, A |
| `REQ-NDX-003` | Regeneration is idempotent: same inputs produce byte-identical managed blocks; a no-change regeneration produces no write and no proposal. | Stable diffs. | U, G, A |
| `REQ-NDX-004` | Missing, duplicated, or corrupted markers cause the write to be refused with a validation error identifying the marker problem. | Fail closed. | U, A |

### LLM integration (`REQ-LLM-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-LLM-001` | All LLM access goes through an `LLMClient` interface; the default adapter targets OpenAI-compatible APIs (OpenRouter). Core modules depend only on the interface. | Provider independence. | C |
| `REQ-LLM-002` | Prompt templates are versioned; every request records template ID + version in provenance. | Reproducibility, audit. | U, I |
| `REQ-LLM-003` | Every LLM task declares a structured output contract; responses are parsed and validated by hand-written typed validators before use. | Determinism boundary. | U, C, A |
| `REQ-LLM-004` | On validation failure, the system returns structured errors to the caller/agent and supports bounded retries (configurable max attempts) with the errors included as feedback context. | Self-correcting loop. | U, I, A |
| `REQ-LLM-005` | Token and cost budgets are enforced per request and per run; exceeding a budget aborts remaining LLM work with a typed `budget-exhausted` error while preserving completed results. | Cost control. | U, A |
| `REQ-LLM-006` | Timeouts and rate limits are handled with typed errors and exponential backoff honoring `retry-after` where present. | Robustness. | U, A |
| `REQ-LLM-007` | Untrusted content (web extractions, note bodies) is passed to prompts only inside clearly delimited data fences with instructions treated as system-owned; instructions found inside data are never executed, and prompts include an injection-resistance preamble. Validators reject outputs referencing capabilities outside the contract. | Prompt-injection resistance. | U, G, A |
| `REQ-LLM-008` | A redaction/minimization hook can strip or mask configured patterns (e.g., secrets, emails) from content before it is sent to a provider. | Privacy. | U |
| `REQ-LLM-009` | Every LLM request/response pair gets a trace ID and an audit record: model, template version, token counts, cost estimate, latency, validation outcome. Raw prompt/response retention follows the retention config. | Auditability. | U, I, A |

### Proposals and review (`REQ-PRP-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-PRP-001` | Proposals are persisted canonically as JSON/JSONL records under `.notekit/proposals/`, one file per proposal (or batched JSONL per run), append-safe. | User-locked decision (ASM-005). | U, I, A |
| `REQ-PRP-002` | Proposal lifecycle: `draft → pending → approved | rejected → applied | failed | conflicted`. Transitions are validated and audited. | Controlled mutation. | U, C, A |
| `REQ-PRP-003` | Only `approved` proposals can be applied; apply re-verifies preconditions (base content hash, target existence rules) immediately before writing. | Safety at the write boundary. | U, I, A |
| `REQ-PRP-004` | Optional Markdown review reports rendered into the vault are read-only projections; the system never parses them for decisions (until a future `ReviewAdapter` exists). | User-locked decision (ASM-005). | I |
| `REQ-PRP-005` | A proposal that would create a note at an existing path, or modify a note whose hash changed, fails as `conflicted` without writing, and the conflict is audited. | Never overwrite silently. | U, I, A |
| `REQ-PRP-006` | Batch application is per-proposal atomic: one failure marks that proposal `failed`/`conflicted` and continues or halts per config (`halt_on_error`), with a batch summary reporting per-item outcomes. | Partial-failure clarity. | I, A |

### Validation (`REQ-VAL-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-VAL-001` | Every AI or ingestion output is validated deterministically before it can become a proposal. | Trust boundary. | U, C, A |
| `REQ-VAL-002` | `ValidationError` is structured: stable `code`, JSON-path-like `field`, human message, machine `hint` for correction, and severity. Multiple errors are returned together. | Agent-correctable feedback. | U, C, A |
| `REQ-VAL-003` | Validation is pure and repeatable: same input yields the same errors, with no I/O side effects. | Deterministic verification. | U |

### Observability (`REQ-OBS-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-OBS-001` | Structured logs (JSON lines) carry trace ID, correlation ID, module, and event fields; log level is configurable. | Debuggability. | U, I |
| `REQ-OBS-002` | Audit events are append-only JSONL under `.notekit/audit/`, covering: scan summaries, proposal creation/decision/apply, LLM requests, ingestion outcomes, conflicts, and rename inferences. | Accountability. | U, I, A |
| `REQ-OBS-003` | A run produces basic metrics (counts, durations, token/cost totals) in the run summary. | Operational insight. | U |
| `REQ-OBS-004` | Requirement-to-module and requirement-to-test mappings are maintained in this plan's traceability matrix and verified by a doc-consistency test (every `REQ-*` appears in ≥1 module and ≥1 acceptance scenario). | Traceability. | U, A |

### Security and safety (`REQ-SEC-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-SEC-001` | All write paths are confined to the vault root, `.notekit/`, and the configured external cache dir; any resolved path escaping these roots is rejected. | Traversal defense. | U |
| `REQ-SEC-002` | API keys come from environment or config references, are never persisted in state or proposals, and are redacted from logs and audit events. | Secret hygiene. | U |
| `REQ-SEC-003` | Extracted HTML is sanitized: scripts, styles, and event handlers stripped before text extraction; extracted text is treated as untrusted data everywhere (REQ-LLM-007). | Untrusted content. | U, G |
| `REQ-SEC-004` | Per-file and per-fetch size limits are enforced (configurable); oversized inputs produce typed errors. | Resource safety. | U |
| `REQ-SEC-005` | Deletion/pruning/archival can only occur through an approved proposal whose diff explicitly shows the destructive action; there is no code path for unproposed deletion. | Destructive-action prevention. | I, A |
| `REQ-SEC-006` | Before applying any proposal that modifies an existing note, the prior content is preserved in `.notekit/backups/` (content-addressed), enabling rollback. | Recoverability. | U, I |

### Configuration (`REQ-CFG-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-CFG-001` | Configuration loads from a file (`.notekit/config.yaml` or explicit path) with environment-variable overrides; unknown keys and invalid values fail with structured errors listing every problem. | Predictable setup. | U, C |
| `REQ-CFG-002` | `dry_run` defaults to `true`; apply operations require both `dry_run=false` (or per-call override) and an approved proposal. | Safe by default. | U, A |

### Extensibility (`REQ-EXT-*`)

| ID | Requirement | Rationale | Verification |
| --- | --- | --- | --- |
| `REQ-EXT-001` | Fetchers, extractors, search providers, LLM providers, validators, taxonomy strategies, index-note source specs, and review adapters are registered against small interfaces via an explicit registry; adding an implementation requires no changes to core modules. | Growth without churn. | C |

## 5. High-Level Architecture

### Summary

notekit is a set of small Go libraries composed around one invariant: **reads are free; writes are proposals.** A pure read path builds a `VaultSnapshot`, indexes, and a link graph. Analysis engines (triage, taxonomy, orphan, ingestion, index-note) consume the snapshot and emit `ProposedChange` records. AI assistance is an internal detail of some engines, always mediated by the `LLMClient` adapter and the validator layer. Only the `Applier` writes to notes, and only for approved proposals whose preconditions still hold.

### Component list

- **Core read path:** VaultRepository, MarkdownParser, MetadataManager, IdentityResolver, LinkExtractor, LinkGraphBuilder, SearchIndex.
- **Analysis engines:** InboxTriageEngine, TaxonomyClassifier (+ TaxonomyManager), OrphanIslandAnalyzer, IndexNoteBuilder, IngestionPipeline (URLDetector, Fetcher, Extractor, SummaryComposer).
- **AI layer:** LLMClient adapter, PromptTemplateStore, StructuredOutputValidator, BudgetTracker.
- **Mutation path:** ProposalStore, ReviewQueue, Applier, BackupStore.
- **Cross-cutting:** ConfigLoader, AuditLog, TraceContext, Registry, AgentCoordinator.

### Data flow

```
                         READ PATH (pure)
  vault/ ──▶ VaultRepository ──▶ MarkdownParser ──▶ VaultSnapshot
                                     │                   │
                              LinkExtractor        MetadataManager
                                     │                   │(sidecar .notekit/meta/)
                              LinkGraphBuilder ──▶ SearchIndex (.notekit/index/)

                         ANALYSIS / AI-ASSISTED PATH
  VaultSnapshot + Graph + Index
        │
        ├─▶ InboxTriageEngine ─┐
        ├─▶ TaxonomyClassifier ─┤        ┌──────────────┐
        ├─▶ OrphanAnalyzer ─────┼──▶ StructuredOutputValidator
        ├─▶ IngestionPipeline ──┤        └──────┬───────┘
        └─▶ IndexNoteBuilder ───┘        valid  │  invalid ──▶ ValidationError[] ──▶ agent retry
                                                ▼
                                       ProposalStore (.notekit/proposals/)
                         WRITE PATH (gated)
  ReviewQueue ──approve──▶ Applier ──precondition check──▶ atomic write ──▶ AuditLog
                                └─ hash mismatch ──▶ conflicted (no write)
```

- **Read path:** never writes to the vault; may refresh sidecar caches under `.notekit/`.
- **Write/proposal path:** engines → validator → ProposalStore → human/agent decision → Applier → backup + atomic write + audit.
- **AI-assisted path:** engine builds prompt from versioned template + fenced data → LLMClient → validator → typed result or `ValidationError[]` → bounded retry with error feedback.
- **Failure path:** every failure is a typed error with a trace ID; batch operations report per-item outcomes; conflicts and budget exhaustion stop writes, never corrupt them.

## 6. Module Design

Each module lists: purpose, responsibilities, contract sketch, inputs/outputs, failure modes, validation, test strategy, requirement IDs. Interface sketches are Go-flavored pseudocode, not implementation.

### 6.1 VaultRepository

- **Purpose:** safe filesystem access to the vault.
- **Responsibilities:** scan (with ignore rules), read note bytes, atomic write, path confinement, mtime+size cache, lock file.
- **Must not own:** parsing, identity semantics, proposal logic.
- **Contract:** `Scan(ctx) (VaultSnapshot, error)`; `Read(relPath) ([]byte, FileStat, error)`; `AtomicWrite(relPath, data, expectedHash) error`.
- **Inputs:** vault path, ignore config. **Outputs:** snapshot, file bytes.
- **Failure modes:** invalid vault path, permission denied, path escape, lock contention, hash-mismatch on write.
- **Validation:** path confinement (REQ-SEC-001); expectedHash precondition (REQ-VLT-005).
- **Tests:** unit with temp dirs; fixture vaults; symlink-escape cases; crash-safety via write interruption simulation.
- **Requirements:** REQ-VLT-001..007, REQ-SEC-001, REQ-SEC-004.

### 6.2 MarkdownParser

- **Purpose:** structural parse of a note: frontmatter block boundaries, body, headings, inline tags.
- **Responsibilities:** locate `---` fences byte-exactly; parse YAML frontmatter (minimal YAML subset parser or one vetted dep — see §16); tolerate invalid frontmatter.
- **Must not own:** writing, link semantics beyond raw extraction handoff.
- **Contract:** `Parse(raw []byte) (ParsedNote, []ValidationError)`.
- **Failure modes:** unterminated fence, invalid YAML, non-UTF-8.
- **Tests:** golden files of tricky frontmatter; fuzz on fence detection.
- **Requirements:** REQ-MD-001, REQ-MD-003, REQ-MD-004.

### 6.3 MetadataManager

- **Purpose:** canonical sidecar metadata per note (`.notekit/meta/<noteID>.json`), plus opt-in frontmatter projection.
- **Responsibilities:** read/merge sidecar + frontmatter views; produce frontmatter-projection proposals for managed keys only; never write frontmatter directly.
- **Contract:** `Get(noteID) (NoteMetadata, error)`; `ProposeProjection(noteID, keys []string) (ProposedChange, error)`.
- **Failure modes:** managed region not locatable (refuse), key not in managed set (refuse).
- **Requirements:** REQ-MD-002, REQ-MD-005.

### 6.4 IdentityResolver

- **Purpose:** stable identity per ASM-003.
- **Responsibilities:** assign/persist internal note IDs; honor configured frontmatter ID key when unique; content-hash change detection; best-effort rename inference on snapshot diff; surface ambiguity.
- **Contract:** `Resolve(snapshot, prevState) (IdentityMap, []RenameCandidate, []ValidationError)`.
- **Failure modes:** duplicate frontmatter IDs; ambiguous rename (one hash, multiple candidates).
- **Requirements:** REQ-VLT-008..010.

### 6.5 LinkExtractor

- **Purpose:** extract wikilinks, embeds, and Markdown links with byte offsets.
- **Contract:** `Extract(parsed ParsedNote) []RawLink`.
- **Tests:** golden files covering aliases, headings, embeds, code-fence exclusion.
- **Requirements:** REQ-LNK-001.

### 6.6 LinkGraphBuilder

- **Purpose:** resolve raw links to note IDs; build graph, backlinks, components.
- **Contract:** `Build(snapshot, identityMap, rawLinks) (LinkGraph, []BrokenLink)`.
- **Failure modes:** none fatal; unresolved → BrokenLink records.
- **Requirements:** REQ-LNK-002..004.

### 6.7 SearchIndex (default `SearchProvider`)

- **Purpose:** keyword search per REQ-IDX-*.
- **Contract:** `type SearchProvider interface { Index(snapshot) error; Search(q SearchQuery) (SearchResultSet, error) }`.
- **Failure modes:** malformed query (typed parse error with position), stale index (detected via snapshot hash, auto-rebuild).
- **Requirements:** REQ-IDX-001..007.

### 6.8 TaxonomyManager

- **Purpose:** own the canonical taxonomy file: load, validate grammar, path IDs, status; apply approved taxonomy-change proposals.
- **Contract:** `Accepted() []TaxonomyPath`; `ValidatePath(p) []ValidationError`; `ProposeAddition(p, rationale) ProposedChange`.
- **Requirements:** REQ-TAX-001, REQ-TAX-002, REQ-TAX-004.

### 6.9 TaxonomyClassifier

- **Purpose:** produce 3–7 candidate paths per note using deterministic signals (existing tags, folder, link neighbors) and optional LLM assistance; route new-path suggestions to TaxonomyManager proposals.
- **Contract:** `Classify(note, ctx) (ClassificationResult, []ValidationError)`.
- **Failure modes:** LLM malformed output (validator rejects → retry loop), budget exhausted.
- **Requirements:** REQ-TAX-003..007, REQ-LLM-003..005.

### 6.10 InboxTriageEngine

- **Purpose:** produce `TriageSuggestion` proposals for untriaged inbox notes.
- **Responsibilities:** gather candidates via SearchIndex + LinkGraph; optionally consult LLM for action recommendation and rationale; flag human review (e.g., prune/merge always requires it).
- **Contract:** `Triage(noteID) (TriageSuggestion, []ValidationError)`.
- **Requirements:** REQ-INB-001..003.

### 6.11 URLDetector

- **Purpose:** find URLs and classify note sparseness.
- **Contract:** `Detect(parsed ParsedNote) URLReport`.
- **Requirements:** REQ-ING-001.

### 6.12 Fetcher (interface) + DefaultHTTPFetcher

- **Purpose:** retrieve remote content under policy.
- **Contract:** `type Fetcher interface { Fetch(ctx, req FetchRequest) (FetchResult, error) }` — errors are the typed taxonomy of REQ-ING-005. Policy checks (scheme, domain lists, SSRF, robots, size, redirects) run before and during fetch.
- **Requirements:** REQ-ING-002..005, REQ-ING-010, REQ-SEC-004.

### 6.13 ContentExtractor (interface)

- **Purpose:** payload → readable text + normalized metadata. Default extractors: HTML (sanitize then text), plain text, PDF (future/optional adapter).
- **Contract:** `Extract(payload FetchResult) (SourceDocument, []ValidationError)`.
- **Failure modes:** empty text (validation error), unsupported content type (typed).
- **Requirements:** REQ-ING-006, REQ-ING-007, REQ-SEC-003.

### 6.14 IngestionPipeline

- **Purpose:** orchestrate detect → fetch (or accept external content) → extract → summarize → propose, as an `IngestionJob` state machine.
- **Contract:** `Run(job IngestionJob) (IngestionOutcome, error)`; `RunExternal(content ExternalContent) (IngestionOutcome, error)`.
- **Failure modes:** every stage failure recorded on the job with typed error; job is resumable/retryable per stage.
- **Requirements:** REQ-ING-001..010.

### 6.15 SummaryNoteComposer

- **Purpose:** compose a `SummaryNoteDraft` from a SourceDocument via versioned prompt + validator; attach candidate taxonomy and candidate links; emit a create-note proposal.
- **Requirements:** REQ-ING-007, REQ-LLM-002..004, REQ-PRP-005.

### 6.16 LLMClient (adapter)

- **Purpose:** single choke point for model calls.
- **Contract:** `Complete(ctx, req LLMRequest) (LLMResponse, error)` where LLMRequest = {model, template ID+version, fenced inputs, output contract ID, budget}; adapter handles auth, retries/backoff, rate limits, timeouts, token accounting.
- **Requirements:** REQ-LLM-001..002, REQ-LLM-005..009, REQ-SEC-002.

### 6.17 StructuredOutputValidator

- **Purpose:** hand-written typed validators per output contract (`ClassificationResult`, `TriageSuggestion`, `SummaryNoteDraft`, taxonomy proposals...).
- **Contract:** `Validate(contractID string, raw []byte) (any, []ValidationError)`.
- **Requirements:** REQ-VAL-001..003, REQ-LLM-003, REQ-TAX-006.

### 6.18 ProposalStore + ReviewQueue

- **Purpose:** persist proposals (JSON/JSONL under `.notekit/proposals/`), manage lifecycle transitions, render optional read-only Markdown review reports.
- **Contract:** `Create(p) (ProposalID, error)`; `Decide(id, decision ReviewDecision) error`; `ListPending(filter) []ProposedChange`; `RenderReport(runID) (reportPath, error)`.
- **Requirements:** REQ-PRP-001..004.

### 6.19 Applier + BackupStore

- **Purpose:** apply approved proposals: re-verify preconditions, back up prior content, atomic write, audit, mark applied/failed/conflicted.
- **Contract:** `Apply(ctx, proposalID) (ApplyResult, error)`; `ApplyBatch(ctx, ids, halt bool) BatchResult`.
- **Requirements:** REQ-PRP-002..006, REQ-VLT-004..005, REQ-SEC-005..006.

### 6.20 OrphanIslandAnalyzer

- **Purpose:** orphan criteria, islands, staleness, duplicate/near-duplicate detection; emit suggestion proposals.
- **Requirements:** REQ-ORP-001..004.

### 6.21 IndexNoteBuilder

- **Purpose:** generate/refresh managed blocks in index notes from source specs.
- **Requirements:** REQ-NDX-001..004, REQ-MD-002.

### 6.22 ConfigLoader / AuditLog / TraceContext / Registry / AgentCoordinator

- **ConfigLoader:** parse + validate config, env overrides, defaults (REQ-CFG-001..002).
- **AuditLog:** append-only JSONL events with trace/correlation IDs (REQ-OBS-001..003).
- **TraceContext:** trace ID per run, correlation ID per task; propagated via `context.Context`.
- **Registry:** explicit registration maps for pluggable implementations (REQ-EXT-001).
- **AgentCoordinator:** thin façade exposing agent-oriented task APIs (see §8); owns the propose→validate→feedback→retry loop wiring; owns nothing the underlying modules own.

## 7. Data Model

All persisted records are JSON with `schema_version`. Canonical IDs: note IDs are ULIDs assigned by IdentityResolver; proposal, job, and trace IDs are ULIDs. Content hashes are SHA-256 of raw note bytes. Timestamps are RFC 3339 UTC. Provenance is mandatory on anything AI-derived.

```jsonc
// NoteIdentity
{ "note_id": "01J...", "path": "projects/goeval.md", "frontmatter_id": null,
  "content_hash": "sha256:...", "first_seen": "...", "last_seen": "..." }

// Note (in-memory view) = NoteIdentity + ParsedNote + NoteMetadata
// NoteMetadata (sidecar canonical)
{ "note_id": "01J...", "tags": ["go","llm"], "taxonomy_paths": ["eng/llm/evaluation"],
  "taxonomy_provenance": {"eng/llm/evaluation": {"source":"human"}},
  "triaged": false, "reviewed_at": null, "custom": {}, "schema_version": 1 }

// VaultSnapshot
{ "snapshot_id": "01J...", "taken_at": "...", "vault_root_hash": "sha256:...",
  "notes": [/* NoteIdentity + FileStat */], "ignored_count": 42 }

// Link / Backlink / BrokenLink
{ "from": "01J...A", "to": "01J...B", "kind": "wikilink|mdlink|embed",
  "raw": "[[Target|Alias]]", "offset": 812 }
{ "target_raw": "[[Missing Note]]", "from": "01J...A", "reason": "unresolved" }

// TaxonomyPath (canonical file entry)
{ "path_id": "tx-0042", "path": "eng/llm/evaluation", "status": "accepted", "description": "..." }

// ClassificationResult (LLM output contract)
{ "note_id": "01J...", "candidates": [
    { "path": "eng/llm/evaluation", "membership": "accepted", "confidence": 0.86,
      "rationale": "...", "provenance": {"source":"llm","model":"...","template":"taxonomy-classify@v3","trace_id":"01J..."},
      "validation": "valid" },
    { "path": "eng/llm/observability", "membership": "proposed-new", "confidence": 0.55, "...": "..." }
  ], "schema_version": 1 }

// SearchQuery / SearchResult
{ "text": "eval harness", "filters": [{"field":"tag","op":"eq","value":"go"}],
  "prefix": true, "limit": 20 }
{ "note_id": "01J...", "score": 7.2,
  "explanation": [{"field":"title","term":"eval","contribution":4.0}],
  "snippets": [{"field":"body","text":"...eval harness...","offset":300}] }

// IngestionJob (state machine)
{ "job_id": "01J...", "note_id": "01J...", "url": "https://...",
  "stage": "detected|fetched|extracted|summarized|proposed|failed",
  "attempts": {"fetch": 1, "summarize": 2},
  "last_error": {"code":"llm-malformed-output","retryable":true}, "trace_id": "01J..." }

// SourceDocument (default retention set — REQ-ING-007)
{ "source_url": "https://...", "final_url": "https://...", "fetched_at": "...",
  "content_type": "text/html", "content_hash": "sha256:...",
  "extractor": {"name":"html","version":"1.2.0"},
  "title": "...", "text": "...", "metadata": {"author":"...","published":"..."},
  "raw_payload_ref": null /* or external cache key when opt-in retention enabled */ }

// SummaryNoteDraft
{ "title": "...", "body_markdown": "...", "source": {/* SourceDocument provenance subset */},
  "candidate_taxonomy": [/* ClassificationResult candidates */],
  "candidate_links": [{"note_id":"01J...","reason":"shared tag: go"}],
  "provenance": {"template":"summarize@v2","model":"...","trace_id":"01J..."} }

// ProposedChange
{ "proposal_id": "01J...", "kind": "create-note|edit-managed-block|frontmatter-projection|move-note|merge-notes|add-links|taxonomy-change|archive-note|prune-note",
  "status": "pending", "created_by": "triage-engine|agent:<id>|human",
  "target": {"note_id":"01J...","path":"inbox/foo.md","base_content_hash":"sha256:..."},
  "diff": {"format":"unified","value":"..."},  // human-reviewable, always present
  "payload": { /* kind-specific structured content */ },
  "requires_human_review": true, "provenance": {...}, "trace_id": "01J...",
  "schema_version": 1 }

// ReviewDecision
{ "proposal_id": "01J...", "decision": "approved|rejected", "decided_by": "human|agent:<id>",
  "reason": "...", "decided_at": "..." }

// ValidationError
{ "code": "taxonomy.path.depth", "field": "candidates[1].path",
  "message": "taxonomy path must have exactly 3 segments, got 2",
  "hint": "append a subtopic segment or choose an accepted path", "severity": "error" }

// AuditEvent
{ "event_id": "01J...", "at": "...", "kind": "proposal.created|proposal.applied|llm.request|ingest.failed|rename.inferred|conflict.detected|...",
  "trace_id": "01J...", "correlation_id": "01J...", "actor": "...", "detail": {...} }

// TraceContext: { "trace_id": ULID (per run), "correlation_id": ULID (per task) }

// AgentTaskResult
{ "task_id": "01J...", "status": "ok|invalid|failed|budget-exhausted",
  "output": {...}, "errors": [/* ValidationError */], "proposals": ["01J..."],
  "trace_id": "01J...", "attempts": 2 }
```

**Conflict detection:** every mutating proposal pins `base_content_hash` (or `base: absent` for creates). Apply recomputes the target's hash; mismatch → `conflicted`, no write, audit event. **Rename inference** records old/new paths + hash in an audit event and (when ambiguous or user-visible) a proposal, never a silent state mutation of human-meaningful data.

## 8. AI-Agent Interaction Model

Agents interact through `AgentCoordinator` task APIs. Every task input and output is a structured JSON document; no free-text protocol.

- **Task inputs:** `{task_kind, params, budget, trace hints}` — e.g., `{"task_kind":"classify-taxonomy","params":{"note_id":"01J..."},"budget":{"max_tokens":4000,"max_usd":0.02}}`.
- **Task outputs:** `AgentTaskResult` (see §7) — status, typed output, `ValidationError[]`, proposal IDs, trace ID, attempt count.
- **Validation signals:** `status:"invalid"` + errors with machine `hint`s. Errors are stable-coded so agents can branch on them.
- **Retry loop (the contract):**
  1. Agent (or engine on the agent's behalf) proposes an action or structured output.
  2. System validates deterministically (StructuredOutputValidator + domain rules).
  3. If invalid, system returns `ValidationError[]` — no partial acceptance.
  4. Agent adjusts and resubmits (attempt count increments; bounded by config).
  5. System re-validates.
  6. Valid outputs become `pending` proposals; nothing is applied without an explicit `approved` ReviewDecision. Proposal kinds flagged `requires_human_review` (prune, merge, taxonomy-change by default) cannot be approved by agents.
- **Error feedback format:** exactly the `ValidationError` structure; ingestion adds the typed fetch-failure taxonomy with `retryable` and `retry_after`.
- **Trace/correlation:** each agent session gets a trace ID; each task a correlation ID; both appear in every log line, audit event, LLM record, and proposal, enabling end-to-end reconstruction.
- **Human review handoff:** `ListPending(requires_human_review=true)` plus the rendered Markdown report gives the human a review surface; decisions re-enter via `Decide`.

**Example — success:** agent submits `classify-taxonomy` for note N → classifier returns 5 candidates → validator passes → 1 candidate is `proposed-new` → system creates one note-classification proposal + one taxonomy-change proposal → `AgentTaskResult{status:ok, proposals:[p1,p2]}` → human approves p2, then p1 → Applier projects taxonomy into sidecar metadata → audit trail links all of it by trace ID.

**Example — failure and adjustment:** agent submits a summary draft whose `candidate_taxonomy[0].path` is `"golang/llm"` (2 segments) → validator returns `taxonomy.path.depth` with hint → agent resubmits with `"eng/llm/evaluation"` → valid → pending proposal. If the agent exhausts max attempts, the task ends `failed` with the last error set, audited, and no proposal is created.

## 9. Verification Strategy

- **Deterministic checks:** all structural validation (paths, taxonomy grammar, managed blocks, proposal preconditions) is pure-function and table-tested.
- **Rule-based validation:** taxonomy membership, fetch policy, managed-key allowlists.
- **Contract validation:** every module boundary type has a validator + contract tests shared by producer and consumer; LLM output contracts are the same validators.
- **Test fixtures:** small fixture vaults (clean, messy-frontmatter, broken-links, inbox-heavy, duplicate-heavy) checked into the repo.
- **Golden files:** parser output, link extraction, index-note managed blocks, snippets, prompt renderings.
- **Fuzz/property tests:** frontmatter fence detection, wikilink parsing, query parser, path-confinement (property: no resolved write path escapes roots; managed-block replacement preserves all outside bytes byte-for-byte).
- **Human review points:** proposal approval; taxonomy-change approval; any `requires_human_review` kind.
- **Requirement-to-test mapping:** the traceability matrix (§18) plus a consistency test that fails the build if a `REQ-*` lacks module or scenario coverage (REQ-OBS-004).

## 10. Observability and Traceability

- **Logs:** JSON lines to stderr/file; fields: `ts, level, module, event, trace_id, correlation_id, note_id?, proposal_id?, detail`. Level configurable.
- **Metrics:** run summary counters (notes scanned, proposals created/applied/conflicted, LLM calls, tokens, estimated cost, durations per stage).
- **Trace/correlation IDs:** per §8; carried in `context.Context` in Go.
- **Audit events:** `.notekit/audit/YYYY-MM.jsonl`, append-only; kinds listed in §7.
- **AI request metadata:** model, provider, template ID+version, token in/out, cost estimate, latency, validation outcome, retry index (REQ-LLM-009). Raw prompt/response retention follows retention config, external dir preferred.
- **Requirement-to-module / requirement-to-test mapping:** §18 matrix, enforced by the consistency test.

## 11. Error Handling

All errors are typed, carry trace IDs, and are audited when they affect state. Batch operations report per-item outcomes.

| Error | Behavior |
| --- | --- |
| Invalid vault path | Fail fast, structured error, no state created (REQ-VLT-003). |
| Invalid Markdown / frontmatter | Note indexed with warnings; frontmatter-dependent features skip it (REQ-MD-003). |
| Ambiguous note identity (dup frontmatter IDs, ambiguous rename) | ValidationError; fall back to sidecar identity; surface candidates for human resolution (REQ-VLT-009..010). |
| Broken wikilink | BrokenLink record + optional fix-link suggestion proposal; never an abort (REQ-LNK-002). |
| URL fetch failure | Typed taxonomy (REQ-ING-005); retryable ones re-attempted with backoff up to config; job marked failed with last error otherwise. |
| Unsafe URL | Rejected pre-I/O with `unsafe-url` code; audited (REQ-ING-004). |
| Empty extracted content | ValidationError `ingest.empty-content`; job fails at extract stage; no summary attempted (REQ-ING-006). |
| LLM timeout / rate limit | Typed error; exponential backoff honoring retry-after; bounded attempts (REQ-LLM-006). |
| LLM malformed output | Validator errors returned as feedback; bounded retry; final failure audited with last errors (REQ-LLM-004). |
| Validation failure (any) | `ValidationError[]` returned; nothing persisted as a proposal (REQ-VAL-001). |
| Write conflict | Apply aborts, proposal → `conflicted`, audit event, prior content untouched (REQ-PRP-005). |
| Budget exhausted | Remaining LLM work aborted with `budget-exhausted`; completed results preserved (REQ-LLM-005). |
| Partial batch failure | Per-item outcomes; continue or halt per `halt_on_error`; batch summary emitted (REQ-PRP-006). |

## 12. Configuration

`.notekit/config.yaml`, env overrides `NOTEKIT_*`. Selected keys and defaults:

| Key | Default |
| --- | --- |
| `vault.path` | required |
| `vault.inbox_path` | `inbox/` |
| `vault.ignore_paths` | `[".notekit/", ".obsidian/", ".trash/"]` (always enforced even if overridden) |
| `state.dir` | `<vault>/.notekit/` |
| `state.external_cache_dir` | unset (raw retention disabled) |
| `identity.frontmatter_key` | unset (disabled) |
| `index_notes.output_path` | `indexes/` |
| `index_notes.block_markers` | `<!-- notekit:begin --> / <!-- notekit:end -->` |
| `taxonomy.file` | `.notekit/taxonomy.yaml` |
| `taxonomy.candidates_min/max` | 3 / 7 |
| `llm.provider` | `openrouter` |
| `llm.base_url` | `https://openrouter.ai/api/v1` |
| `llm.model` | user-set; per-task overrides allowed |
| `llm.api_key_env` | `OPENROUTER_API_KEY` |
| `llm.max_attempts` | 3 |
| `llm.budget.per_request_tokens` | 8000 |
| `llm.budget.per_run_usd` | 1.00 |
| `fetch.allowed_schemes` | `["https","http"]` |
| `fetch.allowed_domains` / `blocked_domains` | empty allowlist = all except blocked |
| `fetch.respect_robots` | true |
| `fetch.max_body_bytes` | 10 MiB |
| `fetch.timeout` | 30s |
| `fetch.max_redirects` | 5 |
| `retention.raw_payloads.enabled` | false |
| `retention.raw_payloads.max_bytes_total` / `ttl_days` | 512 MiB / 30 |
| `dry_run` | true |
| `review.render_markdown_report` | false |
| `review.report_path` | `notekit-reports/` |
| `log.level` | `info` |
| `managed.frontmatter_keys` | `[]` (projection disabled) |
| `apply.halt_on_error` | false |

## 13. Security and Safety Considerations

- **Path traversal / symlinks:** all paths resolved and confined to vault, state dir, external cache; symlinks not followed outside vault root (REQ-VLT-006, REQ-SEC-001).
- **Secrets / API keys:** env-sourced, never persisted, redacted in logs/audit (REQ-SEC-002).
- **Prompt injection:** untrusted content fenced as data; instructions system-owned; outputs must satisfy typed contracts, so injected "instructions" cannot expand capability; extraction sanitizes HTML first (REQ-LLM-007, REQ-SEC-003).
- **SSRF / unsafe URLs:** scheme allowlist, private/loopback IP rejection after DNS resolution, per-hop redirect re-validation (REQ-ING-004).
- **Untrusted remote content:** size caps, content-type checks, sanitization, no execution of any fetched content (REQ-SEC-003..004).
- **Backup and rollback:** content-addressed backups before every note modification; restore is itself a proposal (REQ-SEC-006).
- **Destructive action prevention:** no unproposed deletion path exists; prune/merge require human review by default (REQ-SEC-005, REQ-ORP-004).
- **Privacy to LLMs:** redaction hook, content minimization (send excerpts where sufficient), raw prompt retention off by default and external when on (REQ-LLM-008, ASM-010).

## 14. Extensibility

Every extension point is a small interface plus an explicit registry entry (no `init()`-side-effect magic across module boundaries; registration is explicit at composition time):

- `Fetcher` (per scheme/domain class), `ContentExtractor` (per content type), `SearchProvider`, `LLMClient` (per provider), output-contract validators (per contract ID), taxonomy strategies (deterministic vs LLM-assisted mixes), index-note source specs (per spec kind), and future `ReviewAdapter` (per approval channel).

Adding an implementation = implement interface + register + config reference. Contract tests are provided per interface so third-party implementations can self-verify.

## 15. Testing Plan

- **Unit tests:** every module; table-driven; pure validators exhaustively.
- **Integration tests:** fixture vaults through full read path, engines, proposal store, applier; temp-dir isolation.
- **Contract tests:** shared suites per interface (Fetcher, Extractor, SearchProvider, LLMClient) run against every implementation, including fakes.
- **Acceptance tests:** `AcceptanceTests.md` scenarios implemented as integration tests over fixture vaults with a scripted fake LLM.
- **Regression tests:** every fixed bug adds a fixture or golden file.
- **Golden-file tests:** parser, link extraction, snippets, managed blocks, prompt renderings.
- **Fixture vaults:** `testdata/vaults/{clean,messy,broken-links,inbox,duplicates,islands}`.
- **AI-output validation tests:** corpus of valid/invalid/adversarial (prompt-injected) fake LLM outputs per contract.
- **Failure injection:** fault-injecting Fetcher/LLMClient fakes (timeouts, rate limits, malformed JSON, truncation); interrupted-write simulation for atomicity.

## 16. Implementation Notes

**Go (primary).**

- **Packages:** `vault`, `mdparse`, `identity`, `meta`, `links`, `graph`, `search`, `taxonomy`, `triage`, `ingest` (with `ingest/fetch`, `ingest/extract`), `summarize`, `llm`, `validate`, `proposal`, `apply`, `orphan`, `indexnote`, `config`, `audit`, `trace`, `registry`, `agent`. One interface-heavy `core` package for shared types is acceptable; avoid a god-package.
- **Dependency posture:** stdlib first. Likely exceptions to weigh explicitly: YAML parsing (frontmatter + config) — either a minimal internal YAML-subset parser (frontmatter is usually flat maps/lists) or one vetted dependency (`gopkg.in/yaml.v3`); accept the dep for correctness, isolate it behind `mdparse`/`config` so nothing else imports it. ULIDs are small enough to implement internally. No web framework, no DB, no search library.
- **Filesystem safety:** resolve with `filepath.Clean` + `filepath.Rel` against the root; reject `..` escapes; `os.Root` (Go 1.24+) is a good confinement primitive if the toolchain allows.
- **Atomic writes:** temp file in the same directory, `fsync` file, rename, `fsync` directory. Never write across filesystems.
- **Concurrency:** read path parallelizable with a bounded worker pool (parse/hash); write path strictly serialized behind the `.notekit/lock` file; all APIs take `context.Context`.
- **Interfaces:** small, consumer-defined, named by capability (`Fetcher`, `SearchProvider`); return typed errors implementing a common `Code() string` interface for stable error codes.
- **JSON/frontmatter:** `encoding/json` with `schema_version` on every persisted type; frontmatter writes only via the managed-keys byte-splice in `meta` (never re-serialize whole frontmatter).
- **Testing:** stdlib `testing` + golden files + `testing/fstest`/temp dirs; fuzz via `go test -fuzz`.

**Python (optional prototyping).** Mirror the interfaces as `Protocol` classes; `pyyaml` for frontmatter, `dataclasses`/`pydantic`-free typed dicts to stay contract-compatible; keep JSON contracts identical so a Python prototype's proposals/validators are interchangeable with Go's. Prototype candidates: extractor heuristics, taxonomy prompts, near-duplicate thresholds.

## 17. Open Questions

| ID | Question | Why it matters | Recommended default | Impact if changed later |
| --- | --- | --- | --- | --- |
| `OPQ-001` | YAML dependency: internal subset parser vs `yaml.v3`? | Correctness vs zero-dep purity for frontmatter and config. | `yaml.v3`, isolated behind two packages. | Low — swap is localized. |
| `OPQ-002` | Near-duplicate algorithm and threshold (shingle size, Jaccard cutoff)? | Precision/recall of merge suggestions. | 5-word shingles, Jaccard ≥ 0.7, tune on fixtures. | Low — config + fixture retuning. |
| `OPQ-003` | Should triage `merge` proposals include an LLM-drafted merged body, or only identify the pair? | Bigger value vs bigger review burden and token cost. | MVP: identify pair + rationale only; merged-body drafting is a later task kind. | Medium — new contract + validator. |
| `OPQ-004` | Obsidian link-resolution fidelity: replicate shortest-path disambiguation exactly, or document a simplified rule? | Broken-link accuracy on vaults with duplicate titles. | Simplified rule (exact path → unique title/alias → ambiguous flagged), documented divergence. | Medium — graph edges change; re-index fixes it. |
| `OPQ-005` | PDF extraction in MVP or stub? | PDFs are common ingestion targets but pull dependencies. | Stub returning `unsupported-content-type` guidance; PDF extractor as first post-MVP adapter. | Low — new Extractor registration. |
| `OPQ-006` | Per-task model routing (cheap model for triage, better model for summaries)? | Cost/quality tuning on OpenRouter. | Support `llm.model` global + per-task override keys from day one; defaults all-global. | Low — config only if interface carries model per request (it does). |
| `OPQ-007` | Where do saved queries for index notes live — taxonomy file, config, or their own `.notekit/queries.yaml`? | Affects IndexNoteBuilder spec format ownership. | Separate `queries.yaml` with stable query IDs. | Low. |

## 18. Traceability Matrix

Compact mapping (Requirement → Module → Acceptance scenario → Method). Scenarios are the stable names in `AcceptanceTests.md`; U/I/C/G as defined in §4.

| Requirement | Module | Acceptance scenario | Method |
| --- | --- | --- | --- |
| REQ-VLT-001 | VaultRepository | AT-VLT-01 Scan a valid vault | I, A |
| REQ-VLT-002 | VaultRepository | AT-VLT-01; AT-VLT-03 State folders are ignored | I, A |
| REQ-VLT-003 | VaultRepository | AT-VLT-02 Reject an invalid vault path | U, A |
| REQ-VLT-004 | VaultRepository, Applier | AT-PRP-02 Apply an approved proposal safely | U, I, A |
| REQ-VLT-005 | Applier | AT-PRP-03 Detect a write conflict | U, I, A |
| REQ-VLT-006 | VaultRepository | (unit/property only) | U |
| REQ-VLT-007 | VaultRepository | AT-VLT-01 | U, I |
| REQ-VLT-008 | IdentityResolver | AT-IDN-01 Stable identity across runs | U, I, A |
| REQ-VLT-009 | IdentityResolver | AT-IDN-02 Duplicate frontmatter IDs | U, A |
| REQ-VLT-010 | IdentityResolver | AT-IDN-03 Rename inference | U, I, A |
| REQ-MD-001 | MarkdownParser | AT-MD-01 Parse frontmatter | U, G, A |
| REQ-MD-002 | MetadataManager, IndexNoteBuilder | AT-MD-02 Preserve human content; AT-NDX-02 Update only a managed block | U, G, A |
| REQ-MD-003 | MarkdownParser | AT-MD-03 Tolerate invalid frontmatter | U, I, A |
| REQ-MD-004 | MarkdownParser | AT-IDX-02 Find notes by multiple paths | U, G |
| REQ-MD-005 | MetadataManager | AT-MD-04 Frontmatter projection is opt-in | U, I, A |
| REQ-IDX-001..003 | SearchIndex | AT-IDX-01 Index notes; AT-IDX-02 Find notes by multiple paths | U, C, A |
| REQ-IDX-004..005 | SearchIndex | AT-IDX-02 | U, G, C |
| REQ-IDX-006..007 | SearchIndex, Registry | AT-IDX-01 | C, I |
| REQ-ING-001 | URLDetector | AT-ING-01 Detect URLs in sparse notes | U, G, A |
| REQ-ING-002..003 | Fetcher | AT-ING-02 Successful URL ingestion | C, I, A |
| REQ-ING-004 | Fetcher | AT-ING-03 Handle an inaccessible or unsafe URL | U, A |
| REQ-ING-005 | Fetcher, IngestionPipeline | AT-ING-04 Typed fetch failures with retry semantics | U, C, A |
| REQ-ING-006 | ContentExtractor | AT-ING-05 Handle empty extracted content | U, A |
| REQ-ING-007 | IngestionPipeline | AT-ING-02 | U, I, A |
| REQ-ING-008 | IngestionPipeline | AT-ING-06 Raw payload retention is opt-in | U, I, A |
| REQ-ING-009 | IngestionPipeline | AT-ING-07 Externally provided content | C, I, A |
| REQ-ING-010 | Fetcher (X stub) | AT-ING-08 X/Twitter stub adapter | U, A |
| REQ-TAX-001 | TaxonomyManager | AT-TAX-02 Reject malformed taxonomy paths | U, A |
| REQ-TAX-002 | TaxonomyManager | AT-TAX-01 Generate candidate taxonomy paths | U, I, A |
| REQ-TAX-003 | TaxonomyClassifier, Validator | AT-TAX-01; AT-TAX-02 | U, C, A |
| REQ-TAX-004 | TaxonomyManager | AT-TAX-04 New paths require approval | U, I, A |
| REQ-TAX-005 | TaxonomyClassifier | AT-TAX-01 | U, C, A |
| REQ-TAX-006 | Validator | AT-TAX-02; AT-TAX-03 Retry classification after failure | U, A |
| REQ-TAX-007 | TaxonomyClassifier, Applier | AT-TAX-05 Preserve manual categories | U, I, A |
| REQ-LNK-001 | LinkExtractor | AT-LNK-01 Extract links | U, G, A |
| REQ-LNK-002 | LinkGraphBuilder | AT-LNK-02 Detect broken links | U, G, A |
| REQ-LNK-003 | LinkGraphBuilder | AT-LNK-03 Build a link graph | U, I, A |
| REQ-LNK-004 | LinkGraphBuilder | AT-ORP-02 Detect isolated clusters | U, A |
| REQ-INB-001 | InboxTriageEngine | AT-INB-01 Detect an inbox note needing review | U, I, A |
| REQ-INB-002 | InboxTriageEngine | AT-INB-02 Suggest destination and related notes | U, C, A |
| REQ-INB-003 | InboxTriageEngine | AT-INB-02; AT-PRP-01 Proposals instead of direct changes | I, A |
| REQ-ORP-001 | OrphanIslandAnalyzer | AT-ORP-01 Detect orphan notes | U, I, A |
| REQ-ORP-002 | OrphanIslandAnalyzer | AT-ORP-02 | U, A |
| REQ-ORP-003 | OrphanIslandAnalyzer | AT-ORP-03 Suggest links for weak notes | U, G |
| REQ-ORP-004 | OrphanIslandAnalyzer | AT-ORP-03; AT-PRP-01 | I, A |
| REQ-NDX-001 | IndexNoteBuilder | AT-NDX-01 Build an index note from taxonomy | U, I, A |
| REQ-NDX-002..003 | IndexNoteBuilder | AT-NDX-02 | U, G, A |
| REQ-NDX-004 | IndexNoteBuilder | AT-NDX-03 Refuse write on corrupted markers | U, A |
| REQ-LLM-001..002 | LLMClient | AT-LLM-01 Timeout and malformed output; AT-OBS-01 | C, U, I |
| REQ-LLM-003..004 | Validator, engines | AT-TAX-03; AT-AGT-01 Structured errors to an agent | U, C, A |
| REQ-LLM-005 | BudgetTracker | AT-LLM-02 Respect token and cost budgets | U, A |
| REQ-LLM-006 | LLMClient | AT-LLM-01 | U, A |
| REQ-LLM-007 | Prompting, Validator | AT-LLM-03 Resist prompt injection | U, G, A |
| REQ-LLM-008 | LLMClient hook | (unit only) | U |
| REQ-LLM-009 | LLMClient, AuditLog | AT-OBS-01 Audit events with trace IDs | U, I, A |
| REQ-PRP-001..002 | ProposalStore | AT-PRP-01 | U, C, A |
| REQ-PRP-003 | Applier | AT-PRP-02 | U, I, A |
| REQ-PRP-004 | ProposalStore | AT-PRP-05 Review report is read-only | I, A |
| REQ-PRP-005 | Applier | AT-PRP-03; AT-PRP-04 Prevent overwrite of existing notes | U, I, A |
| REQ-PRP-006 | Applier | AT-PRP-06 Partial batch failure | I, A |
| REQ-VAL-001..003 | Validator | AT-AGT-01; AT-AGT-02 Agent adjusts after failure | U, C, A |
| REQ-OBS-001..003 | AuditLog, TraceContext | AT-OBS-01 | U, I, A |
| REQ-OBS-004 | (doc consistency test) | AT-OBS-02 Requirement-to-test traceability | U, A |
| REQ-SEC-001 | VaultRepository | (unit/property only) | U |
| REQ-SEC-002 | LLMClient, AuditLog | (unit only) | U |
| REQ-SEC-003 | ContentExtractor | AT-LLM-03 | U, G |
| REQ-SEC-004 | Fetcher, VaultRepository | AT-ING-04 | U |
| REQ-SEC-005 | Applier | AT-PRP-01; AT-ORP-03 | I, A |
| REQ-SEC-006 | BackupStore | AT-PRP-02 | U, I |
| REQ-CFG-001 | ConfigLoader | AT-CFG-01 Invalid configuration is rejected | U, C, A |
| REQ-CFG-002 | ConfigLoader, Applier | AT-PRP-01 | U, A |
| REQ-EXT-001 | Registry | AT-ING-07 (external content adapter path) | C, A |
