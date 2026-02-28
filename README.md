# ArcadeDB - AI Skills Package

A Claude Skill package providing expert guidance for building applications with [ArcadeDB](https://arcadedb.com), a multi-model database engine supporting Graph, Document, Key/Value, Search, Time Series, and Vector models.

## What It Does

This skill gives Claude deep knowledge of ArcadeDB including:

- Core concepts (Records, Types, Buckets, Relationships, Transactions, Indexes)
- SQL and OpenCypher query languages
- HTTP, Java, Python, Node.js, JDBC, PostgreSQL, and BOLT APIs
- Graph algorithms (path finding, centrality, community detection)
- Administration (Docker, Kubernetes, HA, security, backups)

## Reference

The official ArcadeDB Manual can be found at https://docs.arcadedb.com/ArcadeDB-Manual.pdf

## Installation

### Claude Code

Copy the skill directory into your project or personal skills folder:

```bash
# Project-level (this project only)
cp -r arcadedb-skill .claude/skills/arcadedb-skill

# Personal (all your projects)
cp -r arcadedb-skill ~/.claude/skills/arcadedb-skill
```

Claude discovers skills automatically. Once installed, the skill activates when ArcadeDB-related questions are detected, or you can invoke it directly with `/arcadedb-skill`.

### Claude.ai

1. Go to **Settings > Features**
2. Upload the `arcadedb.skill` file (it's a ZIP archive)

Requires a Pro, Max, Team, or Enterprise plan with code execution enabled.

### Claude API

Upload via the [Skills API](https://platform.claude.com/docs/en/build-with-claude/skills-guide) (`/v1/skills` endpoints), then reference the `skill_id` in your `container` parameter.

For more details, see the [Agent Skills documentation](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview).

## Packaging

```bash
cd arcadedb-skill && zip -r ../arcadedb.skill . && cd ..
```

## Author

**Francisco Cabrita** — [GitHub](https://github.com/include) · [LinkedIn](https://www.linkedin.com/in/franciscocabrita/)

## License

[Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
