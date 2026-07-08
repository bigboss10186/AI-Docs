# Knowledge Base Reference

## Root

Default root: `/Users/bigboss/code/docs`.

If the repository has moved, locate the knowledge base by going two directories up from this skill folder:

```text
docs/skills/personal-knowledge-base -> docs
```

## Directory Contract

- `README.md`: global entry for the knowledge base; keep it as an index and policy file.
- `indexes/`: cross-topic indexes and relationship pages. Use this when a note connects multiple domains.
- `postgres/`: PostgreSQL mechanisms, general database kernel concepts, and source-coordinate notes.
- `gaussdb/`: GaussDB/openGauss-specific mechanisms, extensions, behavior differences, and product-specific source notes.
- `diagnostics/`: concrete problem investigations, incident notes, root-cause analysis, and troubleshooting records.
- `codex/`: OpenAI Codex source notes, agent mechanisms, tool execution, context management, and app/TUI internals.
- `algorithms/`: algorithms, data structures, complexity-saving patterns, and engineering optimization techniques.
- `inbox/`: temporary fragments, questions, links, and half-formed notes waiting for classification.
- `templates/`: reusable Markdown templates.
- `skills/`: Codex skills that describe how to operate this knowledge base or future reusable workflows.

## Template Mapping

- Concept or mechanism explanation: `templates/concept.md`
- Source reading, call chains, and source coordinates: `templates/source-map.md`
- Troubleshooting or root-cause analysis: `templates/debug.md`
- Stage-level synthesis or periodic consolidation: `templates/summary.md`

When creating a note from a template, preserve the top metadata block and fill:

```md
---
topic:
type:
status: draft
created:
updated:
---
```

Use these `type` values unless there is a good reason to add another:

- `concept`
- `source-map`
- `debug`
- `study-note`
- `summary`
- `question`

Use these `status` values:

- `draft`: still incomplete.
- `usable`: complete enough for future review.
- `solid`: mature and stable.

## Placement Rules

Use `postgres/` for concepts shared by PostgreSQL and openGauss/GaussDB:

- page
- tuple
- heap
- btree
- WAL/XLOG
- VACUUM
- rmgr
- smgr
- checkpoint
- redo/replay

Use `gaussdb/` for openGauss/GaussDB-specific content:

- ASTORE
- USTORE
- CStore
- D-Store
- uheap
- ubtree
- undo worker
- node directory isolation
- GaussDB deployment behavior
- replay differences from PostgreSQL

Use `diagnostics/` for concrete cases:

- a failure or confusing behavior was observed
- there are logs, configs, directories, commands, or reproduction steps
- the note explains symptoms, evidence, root cause, and follow-up checks

Use `indexes/` when a note should remain in its natural directory but must be discoverable from other angles.

## Naming Rules

- Use lowercase kebab-case file names.
- Prefer semantic names over dates: `tablespace-symlink-replay.md` instead of `note-2026-07-06.md`.
- Keep README files as indexes for their directory; do not put long-form topic content in README unless the directory itself is tiny.
- Add a link to the nearest README or index after adding a new note.

## New Note Checklist

1. Identify the topic and type.
2. Pick the target directory.
3. Copy the closest template from `docs/templates/`.
4. Fill metadata, conclusion, background, evidence, and related notes.
5. Add source coordinates when the note is based on code.
6. Link the new note from the nearest README.
7. Update `docs/indexes/` if the note crosses domains.

## Existing Indexes

- `indexes/database.md`: database kernel concepts, PostgreSQL, GaussDB/openGauss, replay, storage, diagnostics.
- `indexes/ai-codex.md`: Codex, AI coding agent internals, tools, context, plugin/skill/MCP.
- `indexes/source-reading.md`: source-reading notes across database and Codex.

## Examples

- A note explaining WAL record dispatch belongs in `postgres/replay/` and should be linked from `postgres/README.md` and `indexes/database.md`.
- A note comparing ASTORE and USTORE belongs in `gaussdb/storage-engines/` and should be linked from `gaussdb/README.md` and `indexes/database.md`.
- A note about a tablespace symlink conflict during replay belongs in `diagnostics/gaussdb/` and can link back to related replay notes.
- A note explaining Codex shell tool approval belongs in `codex/tool-execution/` and should be linked from `codex/README.md` and `indexes/ai-codex.md`.
