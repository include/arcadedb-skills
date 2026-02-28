---
name: arcadedb-skill
description: "Expert guidance for building applications with ArcadeDB, a multi-model database engine supporting Graph, Document, Key/Value, Search, Time Series, and Vector models. Use this skill whenever the user mentions ArcadeDB, or is working with ArcadeDB SQL, Cypher queries against ArcadeDB, ArcadeDB graph schemas, ArcadeDB HTTP API, or building any application that uses ArcadeDB as its database. Also trigger when the user needs help with graph database design patterns in ArcadeDB, migrating from Neo4j or OrientDB to ArcadeDB, writing ArcadeDB traversal queries, setting up ArcadeDB in Docker/Kubernetes, or working with ArcadeDB's vector search, time series, or full-text search features. Even if the user just says 'graph database' and context suggests ArcadeDB, use this skill."
---

# ArcadeDB GraphDB Development Skill

You are an expert ArcadeDB developer and architect. ArcadeDB is a next-generation multi-model DBMS that natively supports Graph, Document, Key/Value, Search-Engine, Time-Series, and Vector-Embedding models in a single engine. It supports multiple query languages (SQL, OpenCypher, Gremlin, GraphQL, MongoDB QL, Redis) and multiple wire protocols (HTTP/JSON, PostgreSQL, MongoDB, Neo4j BOLT, Redis).

## When to use this skill

Use this skill for any task involving ArcadeDB — schema design, query writing, API integration, deployment, migration, or architecture decisions. ArcadeDB is written in LLJ ("Low-Level-Java") for minimal GC pressure and runs on Java 21+.

## Key ArcadeDB concepts you should know

Before diving into reference files, here's a condensed mental model of how ArcadeDB works:

**Data organization**: Everything is a *Record* identified by a *RID* (Record ID, e.g., `#12:0`). Records live in *Types* (schema-level, like tables) which map to physical *Buckets* (storage-level). Types can be Vertex types, Edge types, or Document types. Types support *Inheritance* — a `Person` type can extend a base `V` (vertex) type, and queries on the parent automatically include children.

**Graph model**: ArcadeDB is a native graph database — edges are physical pointers, not JOINs, so traversal speed is independent of database size. Vertices have incoming/outgoing edge sets. Edges link a head vertex to a tail vertex and can carry properties.

**Multi-model**: The same database can store graphs, documents, key/value pairs, time series data, vector embeddings, and full-text searchable content. This isn't an emulation layer — the engine was built from the ground up to support all models natively.

**Schema modes**: Supports schema-full (all properties defined), schema-less (any property allowed), and schema-mixed (some properties defined, extras allowed).

**Indexes**: Uses LSM-Tree (default, write-optimized), Hash (exact-match), Full-Text (Lucene-based), Geospatial (GeoHash + R-Tree), and LSMVectorIndex (HNSW for ANN search). Indexes are created with `CREATE INDEX`.

**Transactions**: Full ACID with MVCC (Multi-Version Concurrency Control). Optimistic concurrency — no locks during reads. Transactions auto-retry on conflict.

**Query languages**: The primary query language is ArcadeDB SQL (extended SQL with graph traversal). OpenCypher is the recommended graph query language (97.8% TCK compliance). Gremlin, GraphQL, MongoDB QL, and Redis commands are also supported.

## How to help the user

### Step 1: Understand what they need

Determine which aspect of ArcadeDB the user needs help with:
- **Schema design** → Read `references/core-concepts.md` for types, relationships, indexes
- **SQL queries** → Read `references/sql-reference.md` for ArcadeDB SQL syntax
- **Cypher queries** → Read `references/cypher-reference.md` for OpenCypher support
- **API/driver integration** → Read `references/api-reference.md` for HTTP API, Java, Python, Node.js, JDBC, BOLT
- **Graph algorithms** → Read `references/graph-algorithms.md` for path finding, centrality, community detection
- **Administration/deployment** → Read `references/admin-reference.md` for Docker, K8s, HA, backup, security
- **Other query languages (Gremlin, GraphQL, MongoDB, Redis)** → Read `references/query-languages-reference.md` for Gremlin Server, GraphQL directives, MongoDB QL, Redis commands
- **Tools, embedded mode, functions, triggers** → Read `references/tools-developer-reference.md` for Studio, Console, Importer, Embedded Server, User Functions, Triggers

### Step 2: Load the right reference

Read only the reference file(s) relevant to the user's question. Each reference file is a comprehensive extraction from the official ArcadeDB Manual. These are your authoritative source — prefer them over general knowledge.

### Step 3: Apply ArcadeDB-specific patterns

When writing code or queries, follow these ArcadeDB-specific patterns:

#### Schema design patterns

```sql
-- Always create vertex and edge types explicitly
CREATE VERTEX TYPE Person;
CREATE VERTEX TYPE Company;
CREATE EDGE TYPE WorksAt;

-- Add properties with types
CREATE PROPERTY Person.name STRING;
CREATE PROPERTY Person.age INTEGER;
CREATE PROPERTY WorksAt.since DATE;

-- Create indexes for query performance
CREATE INDEX ON Person (name) UNIQUE;
CREATE INDEX ON Person (age) NOTUNIQUE;
```

#### SQL query patterns

```sql
-- Graph traversal with MATCH (preferred for complex patterns)
MATCH {type: Person, as: p, where: (name = 'Alice')}
  .out('WorksAt'){as: c}
RETURN p.name, c.name

-- Simple traversal with SELECT
SELECT out('WorksAt').name FROM Person WHERE name = 'Alice'

-- Creating edges between existing vertices
CREATE EDGE WorksAt FROM (SELECT FROM Person WHERE name = 'Alice')
  TO (SELECT FROM Company WHERE name = 'Acme') SET since = '2024-01-01'
```

#### Cypher query patterns

```cypher
-- Use openCypher (not legacy cypher) — it's faster and more complete
MATCH (p:Person {name: 'Alice'})-[:WorksAt]->(c:Company)
RETURN p.name, c.name

-- Create with MERGE to avoid duplicates
MERGE (p:Person {name: 'Alice'})
ON CREATE SET p.age = 30
ON MATCH SET p.lastSeen = datetime()
RETURN p
```

#### HTTP API patterns

```bash
# Query via HTTP POST
curl -X POST "http://localhost:2480/query/graph/mydb" \
  -H "Content-Type: application/json" \
  -d '{"language": "sql", "command": "SELECT FROM Person WHERE age > 30"}'

# Execute a command (INSERT/UPDATE/DELETE)
curl -X POST "http://localhost:2480/command/mydb" \
  -H "Content-Type: application/json" \
  -d '{"language": "sql", "command": "INSERT INTO Person SET name = '\''Bob'\'', age = 25"}'

# Use Cypher via HTTP
curl -X POST "http://localhost:2480/command/mydb" \
  -H "Content-Type: application/json" \
  -d '{"language": "openCypher", "command": "MATCH (n:Person) RETURN n LIMIT 10"}'
```

#### Docker quick start

```bash
docker run --rm -p 2424:2424 -p 2480:2480 -p 5432:5432 \
  --name arcadedb arcadedata/arcadedb:latest
```

### Step 4: Common pitfalls to warn about

- **Don't use `cypher` language identifier** — use `openCypher` (the legacy `cypher` engine translates to Gremlin and is deprecated)
- **Identifiers are case-insensitive** in ArcadeDB (unlike Neo4j) — `Person` and `person` are the same type
- **RIDs are cluster-specific** — don't hardcode them; use indexes or properties for lookups
- **Transactions are optimistic** — design for retry on `ConcurrentModificationException`
- **Edge types need explicit creation** — unlike Neo4j, you must `CREATE EDGE TYPE` before using a relationship type
- **MATCH syntax differs from Cypher** — ArcadeDB SQL MATCH uses `{type: X, as: alias}` pattern with `.out()/.in()` notation, while Cypher uses `(n:Label)-[:REL]->(m)` arrow notation
- **Vector indexes need dimension match** — the `dimensions` parameter must match your embedding model output exactly

### Step 5: Verify your output

After generating queries, schemas, or code:
1. Check that all types referenced actually exist or are created
2. Verify property names and types are consistent
3. Ensure indexes are created for frequently-queried properties
4. Confirm the query language identifier matches what's being used (`sql`, `openCypher`, `gremlin`, `graphql`, `mongo`, `redis`)
5. For production code, include error handling and transaction management

## Reference file index

| File | Contents | When to read |
|------|----------|-------------|
| `references/core-concepts.md` | Records, Types, Buckets, Relationships, Transactions, Indexes, Schema, Materialized Views, Time Series, Graph Database concepts | Schema design, data modeling, understanding ArcadeDB architecture |
| `references/sql-reference.md` | Full SQL syntax — DDL, DML, MATCH, TRAVERSE, functions, filtering | Writing SQL queries, creating schemas, data manipulation |
| `references/cypher-reference.md` | OpenCypher support, compatibility matrix, clauses, functions, limitations | Writing Cypher queries, migrating from Neo4j |
| `references/api-reference.md` | HTTP/JSON API, Java API (Local/Remote/gRPC), Python, Node.js, JDBC, PostgreSQL Protocol, BOLT Protocol, Redis | Application integration, driver setup, API calls |
| `references/graph-algorithms.md` | Path finding, centrality, community detection, structural analysis, similarity, network flow, node embedding | Graph analytics, algorithm selection, network analysis |
| `references/admin-reference.md` | Installation, Docker, Kubernetes, HA, backup/restore, security, settings, server configuration | Deployment, operations, infrastructure, security |
| `references/query-languages-reference.md` | Gremlin (Server, Java, HTTP, multi-database), GraphQL (types, directives), MongoDB QL, Redis commands | Using Gremlin, GraphQL, MongoDB, or Redis with ArcadeDB |
| `references/tools-developer-reference.md` | Studio (web UI), Console (CLI), Importer (ETL), Embedded Server, User Functions, Triggers | Tools usage, embedded mode, custom functions, database triggers |

Load the smallest set of references needed. For simple questions, the information in this SKILL.md may be sufficient without loading any reference file.
