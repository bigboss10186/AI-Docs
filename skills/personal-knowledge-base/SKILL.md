---
name: personal-knowledge-base
description: Maintain and extend the user's docs personal knowledge base. Use when Codex is asked to add, organize, classify, rewrite, or explain notes in /Users/bigboss/code/docs; choose where a document belongs; use the repository's note templates; update indexes; or describe what each docs directory is for.
---

# Personal Knowledge Base

Use this skill to work inside the user's docs knowledge base at `/Users/bigboss/code/docs`.

## Core Workflow

1. Inspect the current structure before editing:
   - `docs/README.md`
   - relevant topic README files
   - relevant files under `docs/indexes/`
2. Classify the request by knowledge type:
   - concept explanation
   - source-map / source reading
   - debug / incident analysis
   - study-note / ordinary note
   - summary
   - question list
3. Choose the target directory using the repository rules:
   - shared database concepts -> `docs/postgres/`
   - GaussDB/openGauss-specific behavior -> `docs/gaussdb/`
   - concrete troubleshooting cases -> `docs/diagnostics/`
   - Codex or AI agent internals -> `docs/codex/`
   - algorithms and complexity patterns -> `docs/algorithms/`
   - cross-topic indexes -> `docs/indexes/`
   - unsorted fragments -> `docs/inbox/`
4. Use a template from `docs/templates/` when creating a new note.
5. After creating or moving a note, update the closest README or index so the note is discoverable.

## References

Read `references/knowledge-base.md` when you need the full directory contract, template mapping, naming rules, or examples of where a note belongs.
