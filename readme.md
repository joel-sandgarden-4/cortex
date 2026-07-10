<p align="center">
    <img src="docs/cortex.png" width="800px" alt="Cortex - memory layer for AI">
</p>

# Cortex

Cortex is a persistent memory system for LLM co-thinking. It gives agents a durable place to retrieve context, reason with it, and keep the memory store healthy over time.

## Contents

- [Getting started](#getting-started)
- [How Cortex works](#how-cortex-works)
- [Interfaces](#interfaces)
- [Storage model](#storage-model)
- [Browser viewer](#browser-viewer)
- [Development and troubleshooting](#development-and-troubleshooting)
- [Reference](#reference)
- [License](#license)
- [Contact](#contact)

## Getting started

### Requirements

| Item | Details |
|---|---|
| Runtime | Node.js 18 or newer |
| Package manager | `npm` for dependency installation; `pnpm` or Corepack for the `start` convenience script |
| Client | An MCP-compatible client such as Claude Desktop or Cursor |

### Environment

| Variable | Purpose | Default |
|---|---|---|
| `PORT` | HTTP port for the Express server | `8010` |
| `MEMORY_DATA_DIR` | Directory that holds the memory store | repo-local `data/` |

`src/server.ts` loads `.env` at startup, so local overrides belong there.

### Install and run

1. Install dependencies.

   ```bash
   npm install
   ```

2. Build the browser bundle for the first viewer load.

   ```bash
   npm run build:web
   ```

3. Start the server.

   ```bash
   npm run serve
   ```

4. For local development with live rebuilds, enable `pnpm` or Corepack, then run the combined start script.

   ```bash
   npm run start
   ```

| Script | Command | Use |
|---|---|---|
| `build:web` | `npm run build:web` | Build the viewer bundle once into `web/dist/`. |
| `watch:web` | `npm run watch:web` | Rebuild the viewer bundle as files change. |
| `serve` | `npm run serve` | Start the Express server, API, MCP endpoint, and static viewer routes. |
| `start` | `npm run start` | Run `watch:web` and `serve` together through `pnpm`; this requires `pnpm` or Corepack. |

### Verify

Build the web bundle before opening `http://localhost:8010/memory`.

## How Cortex works

Cortex runs as a standalone Express server. It loads environment config from `.env`, opens the file-backed memory store, serves the browser viewer at `/memory`, exposes the web API under `/api/memory`, and mounts MCP at `POST /mcp/memory`.

### Architecture

The runtime combines five pieces:

- `src/memory.embedding.ts` builds the semantic index with `Xenova/bge-small-en-v1.5` and HNSW search.
- `src/memory.storage.ts` stores chunks as Markdown files with YAML frontmatter.
- `src/memory.audit.ts` writes an append-only `audit.log`.
- `src/memory.index.ts` coordinates storage, embeddings, audit logging, and file watching.
- `web/src/MemoryApp.tsx` renders the browser viewer.

![Cortex architecture diagram](docs/diagram.drawio.png)

### Workflow

The prompt-driven loop follows retrieve → think → garden → respond.

1. **Retrieve**
   - Query memory metadata first.
   - Review summaries, tags, related links, and metrics.
   - Call `get_chunks` for the IDs that matter.
   - Follow related links or issue another query when needed.

2. **Think**
   - Use the selected chunks to reason about the current task.
   - Keep the retrieved set small and focused.

3. **Garden**
   - Store new durable insights.
   - Update chunks when summaries, tags, or relationships change.
   - Mark chunks relevant when they shape the answer.
   - Mark chunks obsolete when newer context replaces them.

4. **Respond**
   - Cite useful chunks with markdown links back to the viewer, for example `[ref:chunk_id](http://localhost:8010/memory/chunk_id)`.

![Retrieve-think-garden-respond workflow diagram](docs/agent-diagram.drawio.png)

### Relevance metrics

| Metric | Meaning | Operational use |
|---|---|---|
| `retrieved_count` | Search matched the chunk. | High values with low relevance point to broad tags or stale content. |
| `relevant_count` | A response used the chunk. | Track whether the chunk actually helps. |
| `last_relevant_date` | Last useful moment. | Identify content that has gone stale. |
| `relevant_count / retrieved_count` | Relevance ratio. | Decide whether to sharpen tags, update content, or archive the chunk. |

## Interfaces

### HTTP and MCP

| Interface | Route | Notes |
|---|---|---|
| Browser viewer | `GET /memory` | Searchable chunk browser. |
| Chunk list | `GET /api/memory/chunks` | Returns sidebar metadata. |
| Chunk detail | `GET /api/memory/chunks/:id` | Returns one full chunk. |
| Search | `GET /api/memory/search?q=...` | Returns full chunks after metadata search. |
| MCP transport | `POST /mcp/memory` | Streamable HTTP endpoint for MCP clients. |

### MCP tools

| Tool | Description |
|---|---|
| `store_chunk` | Store a new chunk with content and metadata. |
| `update_chunk` | Update chunk content, metadata, or both. |
| `get_chunks` | Fetch full chunks by ID. |
| `query` | Search by meaning and return metadata only. |
| `mark_relevant_chunks` | Increment relevance counters for selected chunks. |
| `mark_obsolete` | Archive a chunk and record the reason. |
| `get_audit_log` | Read the operation log. |
| `memory_stats` | Return store totals. |

`query` returns metadata only. Use `get_chunks` when the full content matters.

## Storage model

### Chunk schema

| Field | Purpose |
|---|---|
| `id` | 6-character hex identifier. |
| `content` | Markdown body stored after frontmatter. |
| `summary` | Short description used for scanning and filenames. |
| `type` | One of `framework`, `insight`, `fact`, `log`, `emotional`, `goal`, or `question`. |
| `epistemic` | One of `established`, `working`, `speculative`, or `deprecated`. |
| `status` | One of `active`, `dormant`, `review`, or `archived`. |
| `surface_tags` | Free-form retrieval tags. |
| `related` | Related chunks and the reason each link exists. |
| `created` | Creation timestamp in ISO format. |
| `updated` | Last update timestamp in ISO format. |
| `accessed` | Last access timestamp in ISO format. |
| `retrieved_count` | Count of metadata-first retrieval matches. |
| `relevant_count` | Count of times the chunk proved useful. |
| `last_relevant_date` | Timestamp for the last relevance mark. |
| `expires` | Optional ISO expiry date. |
| `context_notes` | Optional notes about the source or the archival reason. |

### File layout

Cortex stores memory in the data directory as Markdown files with YAML frontmatter.

```text
data/
├── audit.log
└── chunks/
    ├── a1b2c3-short-summary.md
    ├── d4e5f6-another-note.md
    └── ...
```

Each chunk file uses a 6-character hex ID and a slugged filename. The Markdown body holds the content, and the YAML frontmatter holds the metadata.

### Storage behavior

- Saving a new chunk writes a new Markdown file and adds an entry to `audit.log`.
- Updating a chunk rewrites the file in place. If the summary changes, Cortex also renames the file to match the new slug.
- Marking a chunk relevant increments `relevant_count` and refreshes `last_relevant_date`.
- Marking a chunk obsolete sets `status` to `archived`, appends the reason to `context_notes`, and logs the change.

## Browser viewer

The browser viewer gives a searchable view of the memory store.

- The sidebar lists every chunk with its ID, type, summary, and status.
- The search bar drives semantic search queries.
- Search results show the full chunk, not just metadata.
- The detail pane exposes classification, tags, related chunks, timestamps, relevance metrics, optional context notes, and content.

## Development and troubleshooting

- Short searches do nothing. Enter at least three characters before expecting results.
- If `/memory` loads without the viewer bundle, run `npm run build:web` or `npm run watch:web` first.
- The server watches external edits under `data/chunks/` and reindexes them automatically.
- If `npm run start` fails with a missing command, install `pnpm` or enable Corepack.

## Reference

- Supplemental design paper: [LLM-first personal knowledge management](https://www.linkedin.com/pulse/llm-first-personal-knowledge-management-joel-solymosi-n47qc)

## License

MIT

## Contact

Questions: joel@custlabs.com