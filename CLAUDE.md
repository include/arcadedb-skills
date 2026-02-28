# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is a **Claude Skill package** for ArcadeDB — not a traditional software project. It provides expert guidance for building applications with ArcadeDB, a multi-model database engine (Graph, Document, K/V, Search, Time Series, Vector).

There is no build system, no dependencies, and no executable code. The deliverable is the `.skill` file (a ZIP archive) and the source knowledge base it's built from.

## Repository Structure

```
arcadedb-skill/                  # Skill source
  SKILL.md                       # Master skill definition (YAML frontmatter + methodology)
  evals/evals.json               # 3 evaluation scenarios (social network, Neo4j migration, production setup)
  references/                    # Knowledge base extracted from ArcadeDB Manual v26.2.1
    core-concepts.md             # Records, Types, Buckets, Relationships, Transactions, Indexes
    sql-reference.md             # ArcadeDB SQL syntax & query language
    cypher-reference.md          # OpenCypher support & Neo4j migration
    api-reference.md             # HTTP, Java, Python, Node.js, JDBC, PostgreSQL, BOLT APIs
    graph-algorithms.md          # Path finding, centrality, community detection
    admin-reference.md           # Docker, K8s, HA, security, backups
    query-languages-reference.md # Extended query language reference
    tools-developer-reference.md # Developer tools reference
arcadedb.skill                   # Packaged skill (ZIP of above)
ArcadeDB-Manual.pdf              # Official ArcadeDB v26.2.1 manual (https://docs.arcadedb.com/ArcadeDB-Manual.pdf)
```

## Key Architecture Decisions

- **SKILL.md is the entry point** — it contains YAML frontmatter (trigger conditions) and a 5-step methodology that dispatches to the appropriate reference file based on the user's question.
- **Reference files are modular** — designed to be loaded selectively (not all at once) to keep context efficient. Each covers one domain.
- **All reference content is extracted from the official manual** — treat the PDF as the canonical source when updating references.
- **Evals test with/without skill** — each scenario is run twice to measure skill effectiveness.

## Working With This Repo

### Updating reference content

1. Read the relevant section of `ArcadeDB-Manual.pdf`
2. Update the corresponding `references/*.md` file
3. Ensure SKILL.md's reference index stays accurate

### Packaging the skill

The `.skill` file is a ZIP archive. To repackage after changes:
```bash
cd arcadedb-skill && zip -r ../arcadedb.skill . && cd ..
```

### Evaluations

Eval scenarios are in `arcadedb-skill/evals/evals.json`. Each has an `id`, `prompt`, and `expected_output`. Results are reviewed via the HTML report.

## ArcadeDB Domain Knowledge

Key concepts that affect how content is structured in this skill:

- **RID** (Record ID, e.g. `#12:0`) — every record's unique identifier
- **Types** = schema (Vertex, Edge, Document); **Buckets** = physical storage
- **Native graph** — edges are physical pointers, O(1) traversal regardless of DB size
- **Query languages**: SQL (primary), OpenCypher (recommended for graph), Gremlin, GraphQL, MongoDB QL, Redis
- **Wire protocols**: HTTP/JSON, PostgreSQL, MongoDB, Neo4j BOLT, Redis
- **Schema modes**: schema-full, schema-less, schema-mixed
- **Index types**: LSM-Tree, Hash, Full-Text (Lucene), Geospatial, HNSW Vector
- **ACID with MVCC** — optimistic concurrency, auto-retry on conflict
