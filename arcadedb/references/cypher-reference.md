# ArcadeDB OpenCypher Reference

## Overview

ArcadeDB provides native support for Cypher, a declarative graph query language originally developed by Neo4j. The native OpenCypher implementation in ArcadeDB uses an ANTLR4-based parser with the official Cypher 2.5 grammar.

### Key Features

- **High Performance**: Direct execution without Gremlin translation (comparable to native SQL)
- **Full Read/Write Support**: MATCH, CREATE, MERGE, SET, DELETE with automatic transaction handling
- **Advanced Pattern Matching**: Variable-length paths, OPTIONAL MATCH, multiple MATCH clauses
- **Query Composition**: UNION/UNION ALL for combining results, WITH for query chaining
- **Procedures and Profiling**: CALL for invoking built-in procedures and custom functions, PROFILE for performance analysis
- **Rich Function Library**: 23+ native Cypher functions plus bridge to 100+ ArcadeDB SQL functions
- **Cost-Based Optimizer**: Intelligent query planning with index selection and join ordering
- **100% Test Coverage**: Comprehensive test suite with 295+ passing tests

---

## Cypher Compatibility Matrix

### OpenCypher TCK Compliance

ArcadeDB's OpenCypher implementation has been validated against the official OpenCypher Technology Compatibility Kit (TCK) version 9, developed by Neo4j and released under the Apache License 2.0.

**Compliance Metrics:**
- **TCK Pass Rate**: 97.8% (Production Ready)
- **Total Test Scenarios**: 3,897 (Official OpenCypher v9 test suite)
- **Passed Scenarios**: 3,812 (Full compatibility with core features)
- **Failed Scenarios**: 0 (All executable tests pass)
- **Skipped Scenarios**: 85 (2.2% - documented limitations)

### Known Limitations

The 85 skipped scenarios (2.2%) represent either architectural design choices or rare edge cases that rarely occur in production use.

#### 1. Temporal Range Limitation (Java Platform Constraint)

**TCK Impact**: 4 scenarios (temporal sorting with extreme date ranges)

**Reason**: ArcadeDB uses Java's `LocalDateTime` with nanosecond precision stored as a 64-bit long, limiting the date range to approximately 1677-2262.

**Examples that don't work**:
```cypher
// Year 0001 with nanoseconds - exceeds range
CREATE (:Event {datetime: localdatetime(year: 1, month: 1, day: 1,
                                        hour: 1, minute: 1, second: 1,
                                        nanosecond: 1})})

// Year 9999 with nanoseconds - exceeds range
CREATE (:FutureEvent {datetime: localdatetime(year: 9999, month: 9, day: 9,
                                              hour: 9, minute: 59, second: 59,
                                              nanosecond: 999999999})})
```

**Workaround**: Use dates within the 1677-2262 range (covers virtually all real-world data), or store historical dates as strings.

#### 2. Case-Insensitive Identifiers (By Design)

**TCK Impact**: ~40 scenarios (case-sensitive type/property names)

**Reason**: ArcadeDB follows SQL conventions where identifiers are case-insensitive, providing consistency across query languages.

**Examples that don't work**:
```cypher
// These create a collision - T2 and t2 are considered the same type
CREATE ()-[:T2]->()
CREATE ()-[:t2]->()  // Error: Conflicts with :T2
```

**Workaround**: Use distinct type names (e.g., `:Type2` and `:Type2Alt`).

#### 3. Variable-Length Path with USING Clause (Advanced Feature)

**TCK Impact**: ~20 scenarios (VLP with relationship list constraints)

**Reason**: The USING clause for constraining VLP traversal to specific relationship lists is an advanced feature rarely used in production.

**Examples that don't work**:
```cypher
// Collect relationships, then traverse using only those relationships
MATCH ()-[rels]->()
MATCH (a)-[?2 USING rels]->(b)  // USING clause not implemented
RETURN a, b
```

**Workaround**: Use standard VLP with relationship type filters:
```cypher
MATCH (a)-[:TYPE*2]->(b)  // Works - filters by type
RETURN a, b
```

#### 4. Cross-Type Comparison Strictness (Semantic Edge Case)

**TCK Impact**: ~20 scenarios (incompatible type comparisons)

**Reason**: Comparing fundamentally different types (nodes vs strings, paths vs primitives) in production code typically indicates a query error. ArcadeDB enforces strict null-return semantics for cross-type comparisons.

**Affected**: Strict null-return semantics for cross-type comparisons.

#### 5. User Functions Support

**TCK Impact**: 0 scenarios (extension feature)

**ArcadeDB Enhancement**: ArcadeDB provides user-defined functions in 4 languages (SQL, JavaScript, OpenCypher, Java) vs Neo4j's Java-only compiled plugins.

---

## Execution Models

### OpenCypher Implementation

- **NEW (Recommended)**: `openCypher` - Native implementation with direct execution
  - Comparable performance to native SQL
  - Full read/write support
  - Latest features and development

- **DEPRECATED**: `cypher` - Legacy Cipher-for-Gremlin implementation
  - Up to 20x slower for complex queries
  - Limited write operation support
  - No active development (will be removed in future release)

**Migration**: Simply change query language from `cypher` to `openCypher`. The syntax is identical for supported features.

---

## Comparison with Legacy Cypher (Neo4j-based)

| Feature | openCypher (NEW) | cypher (DEPRECATED) |
|---------|------------------|-------------------|
| Execution | Native, direct | Gremlin translation |
| Performance | Fast (comparable to SQL) | Slow (up to 20x slower) |
| Write Operations | Full support | Limited support |
| OPTIONAL MATCH | Supported | Partial support |
| Variable-length Paths | Supported | Limited |
| UNION / UNION ALL | Supported | Not supported |
| CALL Procedures | Supported | Not supported |
| PROFILE | Supported | Not supported |
| Aggregations | Full support | Limited |
| Functions | 100+ functions | Limited |
| Transaction Handling | Automatic | Manual |
| Active Development | Yes | No (abandoned upstream) |

---

## Using OpenCypher

### Query Language Identifier

To execute OpenCypher queries, use `openCypher` as the query language identifier.

#### From Java API

```java
ResultSet result = database.query("openCypher",
    "MATCH (p:Person) WHERE p.age >= $minAge RETURN p.name, p.age ORDER BY p.age",
    "minAge", 25);
```

#### Through Neo4j BOLT Protocol (v26.2.1+)

```java
Driver driver = GraphDatabase.driver("bolt://localhost:7687", AuthTokens.basic("root", "password"));
Session session = driver.session(SessionConfig.forDatabase("mydatabase"));
Result result = session.run("MATCH (p:Person) WHERE p.age >= 25 RETURN p.name, p.age ORDER BY p.age");
```

#### Through PostgreSQL Driver

Prefix your query with `{openCypher}`:
```sql
{openCypher} MATCH (p:Person) WHERE p.age >= 25 RETURN p.name, p.age ORDER BY p.age
```

#### Through HTTP/JSON API (GET - idempotent queries)

```bash
curl "http://localhost:2480/query/graph/openCypher/MATCH%20(p:Person)%20RETURN%20p"
```

#### Through HTTP/JSON API (POST - updates)

```bash
curl -X POST "http://localhost:2480/command/graph" \
  -d '{"language": "openCypher", "command": "CREATE (p:Person {name: \"Alice\", age: 30})"}'
```

### Using Record IDs (RIDs)

You can use ArcadeDB's RecordIDs (RID) in Cypher queries to start from a specific vertex. RIDs in Cypher are always strings.

```cypher
MATCH (m:Movie)-[a:ACTED_IN]->(p:Person) WHERE id(m) = '#1:0' RETURN *
```

### Parameters

OpenCypher supports parameterized queries using the `$paramName` syntax.

```cypher
MATCH (p:Person) WHERE p.age >= $minAge AND p.city = $city RETURN p
```

Pass parameters as key-value pairs:
```java
ResultSet result = database.query("openCypher",
    "MATCH (p:Person) WHERE p.age >= $minAge RETURN p",
    "minAge", 25);
```

### Automatic Transaction Handling

All write operations (CREATE, SET, DELETE, MERGE) automatically handle transactions:

- **If no transaction is active**: The operation creates, executes, and commits its own transaction
- **If a transaction is already active**: The operation reuses it without committing
- **Proper rollback on errors**: Failed self-managed transactions are automatically rolled back

```java
// Automatic transaction - commits automatically
database.command("openCypher", "CREATE (n:Person {name: 'Alice'})");

// Manual transaction - you control commit/rollback
database.transaction(() -> {
    database.command("openCypher", "CREATE (a:Person {name: 'Alice'})");
    database.command("openCypher", "CREATE (b:Person {name: 'Bob'})");
    database.command("openCypher", "MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'}) CREATE (a)-[:KNOWS]->(b)");
    // Commits when lambda completes successfully
});
```

### Performance Optimization

The native OpenCypher implementation includes a cost-based query optimizer that:

- **Automatically selects optimal indexes** for node lookups
- **Uses range index scans** for inequality predicates (<, >, <=, >=, <>)
- **Optimizes join order** based on cardinality estimation
- **Detects bounded patterns** for efficient relationship traversal

**Best Practices for Performance**:

1. **Create indexes on frequently queried properties**:
   ```cypher
   CREATE INDEX ON Person (name)
   CREATE INDEX ON Movie (title)
   ```

2. **Use labeled nodes in MATCH patterns** for faster type filtering:
   ```cypher
   MATCH (p:Person {name: 'Alice'})  // Good - uses index on Person type
   RETURN p

   MATCH (n {name: 'Alice'})  // Avoid - scans all nodes
   RETURN n
   ```

3. **Prefer specific property constraints over full scans**:
   ```cypher
   MATCH (p:Person) WHERE p.age > 30  // Good - might use range index
   RETURN p

   MATCH (p:Person) WHERE p.salary / 1000 > 50  // Avoid - computed predicate
   RETURN p
   ```

4. **Use parameters for queries executed multiple times** (enables query plan caching):
   ```cypher
   MATCH (p:Person) WHERE p.age >= $minAge RETURN p
   ```

---

## Clause Reference

### MATCH

The MATCH clause searches for patterns in the graph. It is the primary read operation in Cypher.

#### Syntax

```cypher
MATCH pattern [, pattern ...]
[WHERE condition]
```

#### Pattern Syntax

- `(variable)` - Match any node
- `(variable:Label)` - Match nodes with specific label
- `(variable:Label {prop: value})` - Match nodes with properties
- `(a)-[r]->(b)` - Match directed relationship
- `(a)-[r]-(b)` - Match relationship in either direction
- `(a)-[r:TYPE]->(b)` - Match relationship with specific type
- `(a)-[r:TYPE1|TYPE2]->(b)` - Match relationship with multiple types
- `(a)-[*min..max]->(b)` - Match variable-length path
- `path = (a)-[r:TYPE]->(b)` - Named path

#### Examples

```cypher
// Match all Person nodes
MATCH (p:Person)
RETURN p

// Match with property filter
MATCH (p:Person {name: 'Alice', age: 30})
RETURN p

// Match relationships
MATCH (p:Person)-[r:KNOWS]->(friend:Person)
RETURN p.name, friend.name

// Match with relationship properties
MATCH (p:Person)-[r:WORKS_AT {since: 2020}]->(c:Company)
RETURN p.name, c.name

// Variable-length paths
MATCH (a:Person {name: 'Alice'})-[:KNOWS]*1..3->(b:Person)
WHERE a.name = 'Alice'
RETURN b.name

// Named paths
MATCH path = (a:Person)-[:KNOWS]->(b:Person)
WHERE a.name = 'Alice'
RETURN path

// Comma-separated patterns (Cartesian product)
MATCH (a:Person), (b:Company)
RETURN a.name, b.name

// Multiple MATCH clauses
MATCH (a:Person {name: 'Alice'})
MATCH (b:Person {name: 'Bob'})
RETURN a, b
```

---

### OPTIONAL MATCH

OPTIONAL MATCH works like MATCH, but returns NULL values for missing parts of the pattern instead of filtering out rows. It behaves like a LEFT OUTER JOIN in SQL.

#### Syntax

```cypher
MATCH pattern
OPTIONAL MATCH optional_pattern
[WHERE condition]
RETURN ...
```

**Note**: The WHERE clause after OPTIONAL MATCH filters the optional match results, but preserves rows where the match failed (with NULL values).

#### Examples

```cypher
// Find all people and their optional movies
MATCH (p:Person)
OPTIONAL MATCH (p)-[r:ACTED_IN]->(m:Movie)
RETURN p.name, m.title

// With WHERE clause scoped to OPTIONAL MATCH
MATCH (p:Person)
OPTIONAL MATCH (p)-[:KNOWS]->(friend:Person)
WHERE friend.age > 30
RETURN p.name, friend.name
```

---

### WHERE

The WHERE clause filters results based on conditions.

#### Syntax

```cypher
WHERE condition
```

#### Operators

| Type | Operators | Example |
|------|-----------|---------|
| Comparison | =, <>, <, >, <=, >= | WHERE n.age >= 25 |
| Logical | AND, OR, NOT | WHERE n.age > 25 AND n.city = 'NYC' |
| Null check | IS NULL, IS NOT NULL | WHERE n.email IS NOT NULL |
| List | IN | WHERE n.name IN ['Alice', 'Bob', 'Charlie'] |
| String | STARTS WITH, ENDS WITH, CONTAINS | WHERE n.name STARTS WITH 'A' |
| Regex | =~ | WHERE n.email =~ '.*@gmail\.com' |

#### Examples

```cypher
// Simple comparison
MATCH (p:Person)
WHERE p.age > 30
RETURN p.name

// Multiple conditions
MATCH (p:Person)
WHERE p.age >= 25 AND p.age <= 40 AND p.city = 'NYC'
RETURN p

// NULL checks
MATCH (p:Person)
WHERE p.email IS NOT NULL
RETURN p.name, p.email

// IN operator
MATCH (p:Person)
WHERE p.name IN ['Alice', 'Bob', 'Charlie']
RETURN p

// String matching
MATCH (p:Person)
WHERE p.name STARTS WITH 'A'
RETURN p

// Regular expression
MATCH (p:Person)
WHERE p.email =~ '.*@example\.com'
RETURN p

// Parenthesized expressions
MATCH (p:Person)
WHERE (p.age < 25 OR p.age > 60) AND p.status = 'active'
RETURN p

// Pattern predicates
MATCH (p:Person)
WHERE (p)-[:KNOWS]->()
RETURN p.name AS hasConnections
```

---

### RETURN

The RETURN clause specifies what to include in the query result.

#### Syntax

```cypher
RETURN expression [AS alias] [, expression [AS alias] ...]
```

#### Examples

```cypher
// Return nodes
MATCH (p:Person)
RETURN p

// Return properties
MATCH (p:Person)
RETURN p.name, p.age

// Return with aliases
MATCH (p:Person)
RETURN p.name AS personName, p.age AS personAge

// Return all: *
MATCH (a:Person)-[r:KNOWS]->(b:Person)
RETURN *

// Return expressions
MATCH (p:Person)
RETURN p.name, toUpper(p.city) AS upperCity

// Aggregations
MATCH (p:Person)
RETURN count(p) AS total, avg(p.age) AS avgAge

// RETURN DISTINCT - removes duplicate rows
MATCH (p:Person)
RETURN DISTINCT p.city AS city

// DISTINCT with multiple columns
UNWIND [1, 2, 2, 3, 3, 3] AS n
RETURN DISTINCT n

// Aggregations with DISTINCT
UNWIND [1, 1, 2, 3] AS n
RETURN count(DISTINCT n) AS uniqueCount, sum(DISTINCT n) AS uniqueSum

// Standalone expressions (no MATCH)
RETURN abs(-42), sqrt(16), range(1, 5)
```

---

### CREATE

The CREATE clause creates new nodes and relationships.

#### Syntax

```cypher
CREATE pattern [, pattern ...]
```

#### Examples

```cypher
// Create a node
CREATE (p:Person {name: 'Alice', age: 30})

// Create multiple nodes
CREATE (a:Person {name: 'Bob'}), (b:Person {name: 'Charlie'})

// Create a relationship between new nodes
CREATE (a:Person {name: 'Alice'})-[:KNOWS]->(b:Person {name: 'Bob'})

// Create a relationship with properties
CREATE (a:Person {name: 'Alice'})-[:WORKS_AT {since: 2020}]->(c:Company {name: 'ArcadeDB'})

// Create a relationship between existing nodes
MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'})
CREATE (a)-[:KNOWS {since: 2024}]->(b)

// Create chained paths
CREATE (a:Person {name: 'A'})-[:KNOWS]->(b:Person {name: 'B'})-[:KNOWS]->(c:Person {name: 'C'})
```

---

### MERGE

The MERGE clause creates nodes or relationships if they don't exist, or matches them if they do (upsert operation).

#### Syntax

```cypher
MERGE pattern
[ON CREATE SET property = value [, property = value ...]]
[ON MATCH SET property = value [, property = value ...]]
```

#### Examples

```cypher
// Find or create a node
MERGE (p:Person {name: 'Alice'})

// With ON CREATE action
MERGE (p:Person {name: 'Bob'})
ON CREATE SET p.created = true, p.timestamp = timestamp()

// With ON MATCH action
MERGE (p:Person {name: 'Alice'})
ON MATCH SET p.lastSeen = timestamp(), p.visits = p.visits + 1

// Both ON CREATE and ON MATCH
MERGE (p:Person {name: 'Charlie'})
ON CREATE SET p.created = true, p.count = 1
ON MATCH SET p.count = p.count + 1, p.updated = true

// Merge relationship between existing nodes
MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'})
MERGE (a)-[r:KNOWS]->(b)
ON CREATE SET r.since = 2024
ON MATCH SET r.lastContact = timestamp()
```

---

### SET

The SET clause updates properties on nodes and relationships.

#### Syntax

```cypher
SET property = value [, property = value ...]
```

#### Examples

```cypher
// Update a single property
MATCH (p:Person {name: 'Alice'})
SET p.age = 31

// Update multiple properties
MATCH (p:Person {name: 'Alice'})
SET p.age = 31, p.city = 'Boston'

// Update relationship property
MATCH (a:Person)-[r:KNOWS]->(b:Person)
WHERE a.name = 'Alice' AND b.name = 'Bob'
SET r.strength = 10
```

---

### DELETE

The DELETE clause removes nodes and relationships from the graph.

#### Syntax

```cypher
DELETE variable [, variable ...]
DETACH DELETE variable [, variable ...]
```

**Note**: A node cannot be deleted if it has relationships. Use DETACH DELETE to delete a node and all its relationships.

#### Examples

```cypher
// Delete a relationship
MATCH (a:Person)-[r:KNOWS]->(b:Person)
WHERE a.name = 'Alice' AND b.name = 'Bob'
DELETE r

// Delete a node (must have no relationships)
MATCH (p:Person {name: 'TestNode'})
DELETE p

// Delete node with all its relationships
MATCH (p:Person {name: 'Alice'})
DETACH DELETE p

// Delete multiple elements
MATCH (a:Person)-[r:TEMP]->(b:Person)
DELETE a, r, b
```

---

### WITH

The WITH clause allows query chaining by passing results from one part of a query to another. It can project, filter, aggregate, and paginate intermediate results.

#### Syntax

```cypher
WITH expression [AS alias] [, expression [AS alias] ...]
[WHERE condition]
[ORDER BY expression [ASC|DESC]]
[SKIP n]
[LIMIT n]
```

#### Examples

```cypher
// Basic projection
MATCH (p:Person)
WITH p.name AS name, p.age AS age
WHERE age > 30
RETURN name ORDER BY name

// Filter after projection
MATCH (p:Person)
WITH p.name AS name, p.age AS age
WHERE age > 30
RETURN name

// Aggregation with WITH
MATCH (p:Person)-[r:ACTED_IN]->(m:Movie)
WITH m.title AS movie, count(p) AS actorCount
WHERE actorCount > 2
RETURN movie, actorCount

// Aggregation with WHERE
MATCH (p:Person)-[r:ACTED_IN]->(m:Movie)
WITH m.title AS movie, count(p) AS actorCount
WHERE actorCount > 2
RETURN movie, actorCount

// DISTINCT in WITH
MATCH (p:Person)
WITH DISTINCT p.city AS city
RETURN city ORDER BY city

// Pagination with WITH
MATCH (p:Person)
WITH p.name AS name
ORDER BY name
SKIP 5
LIMIT 10
RETURN name

// Multiple WITH clauses
MATCH (p:Person)
WITH p.name AS name, p.age AS age
WHERE age > 25
WITH name, age
WHERE age < 50
RETURN name ORDER BY name

// WITH * (pass all variables)
MATCH (p:Person)
WITH *
WHERE p.age > 30
RETURN p.name
```

---

### UNWIND

The UNWIND clause expands a list into individual rows.

#### Syntax

```cypher
UNWIND list AS variable
```

#### Examples

```cypher
// Unwind a literal list
UNWIND [1, 2, 3] AS x
RETURN x

// Unwind with range
UNWIND range(1, 10) AS num
RETURN num

// Unwind property array
MATCH (p:Person {name: 'Alice'})
UNWIND p.hobbies AS hobby
RETURN p.name, hobby

// Create multiple nodes from list
UNWIND ['Alice', 'Bob', 'Charlie'] AS name
CREATE (:Person {name: name})

// Nested unwind
UNWIND [[1, 2], [3, 4]] AS inner
UNWIND inner AS num
RETURN num

// Combine with MATCH context
MATCH (p:Person)
UNWIND p.skills AS skill
RETURN p.name, skill
```

---

### ORDER BY

The ORDER BY clause sorts the results.

#### Syntax

```cypher
ORDER BY expression [ASC|DESC] [, expression [ASC|DESC] ...]
```

#### Examples

```cypher
// Ascending order (default)
MATCH (p:Person)
RETURN p.name
ORDER BY p.name

// Descending order
MATCH (p:Person)
RETURN p.name, p.age
ORDER BY p.age DESC

// Multiple sort keys
MATCH (p:Person)
RETURN p.name, p.age, p.city
ORDER BY p.city ASC, p.age DESC, p.name ASC
```

---

### SKIP and LIMIT

SKIP skips a number of results, and LIMIT restricts the maximum number of results returned. Used together for pagination.

#### Syntax

```cypher
SKIP number
LIMIT number
```

#### Examples

```cypher
// Limit results
MATCH (p:Person)
RETURN p.name
LIMIT 10

// Skip results
MATCH (p:Person)
RETURN p.name
ORDER BY p.name
SKIP 20

// Pagination (page 3, 10 items per page)
MATCH (p:Person)
RETURN p.name
ORDER BY p.name
SKIP 20
LIMIT 10
```

---

### EXPLAIN

The EXPLAIN clause shows the query execution plan without actually executing the query. It's useful for understanding how the query optimizer processes your query and for performance tuning.

#### Syntax

```cypher
EXPLAIN query
```

#### Output

The EXPLAIN output includes:

- Whether the cost-based optimizer is being used
- Physical operators in the execution plan (for optimized queries)
- Estimated costs and row counts
- Index usage information

#### Examples

```cypher
// Show execution plan for a simple query
EXPLAIN MATCH (p:Person) RETURN p

// Show plan for a relationship traversal
EXPLAIN MATCH (a:Person)-[:KNOWS]->(b:Person) WHERE a.name = 'Alice' RETURN b

// Show plan for a query with aggregation
EXPLAIN MATCH (p:Person) RETURN p.city, count(p)
```

**Example Output:**

```
OpenCypher Native Execution Plan
==================================

Using Cost-Based Query Optimizer

Physical Plan:
NodeIndexSeek (Person.name = 'Alice')
  └─ Estimated Cost: 10.00
  └─ Estimated Rows: 1

Total Estimated Cost: 10.00
Total Estimated Rows: 1
```

---

### PROFILE

The PROFILE clause executes the query and returns performance metrics along with the actual results. Unlike EXPLAIN which only shows the plan, PROFILE runs the query and provides real execution statistics.

#### Syntax

```cypher
PROFILE query
```

#### Output

The PROFILE output includes:

- Execution time in milliseconds
- Number of rows returned
- Query execution plan details (when using optimizer)

#### Examples

```cypher
// Profile a simple query
PROFILE MATCH (p:Person) RETURN p.name

// Profile a relationship traversal
PROFILE MATCH (a:Person)-[:KNOWS]->(b:Person) WHERE a.name = 'Alice' RETURN b.name

// Profile a query with aggregation
PROFILE MATCH (p:Person) RETURN p.city, count(p) AS count
```

**Example Output:**

```
OpenCypher Query Profile
========================

Execution Time: 15 ms
Rows Returned: 42

Physical Plan:
NodeTypeScan (Person)
  └─ Estimated Rows: 50

Actual Results Follow...
```

**Note**: The profile information is returned as the first row in the result set, followed by the actual query results.

---

### UNION and UNION ALL

The UNION clause combines the results of two or more queries into a single result set. UNION removes duplicate rows, while UNION ALL preserves all rows including duplicates.

#### Syntax

```cypher
query1
UNION
query2
[UNION [ALL]
query3 ...]
```

**Important**: All queries in a UNION must return the same column names.

#### Examples

```cypher
// UNION - removes duplicate results
MATCH (p:Person) RETURN p.name AS name
UNION
MATCH (c:Company) RETURN c.name AS name

// UNION ALL - keeps all results including duplicates
MATCH (p:Person) RETURN p.name AS name
UNION ALL
MATCH (c:Company) RETURN c.name AS name

// Multiple UNIONs
MATCH (p:Person) RETURN p.name AS name
UNION
MATCH (c:Company) RETURN c.name AS name
UNION
MATCH (l:Location) RETURN l.name AS name

// UNION with WHERE clauses
MATCH (p:Person) WHERE p.age > 30 RETURN p.name AS name, 'person' AS type
UNION ALL
MATCH (c:Company) WHERE c.employees > 100 RETURN c.name AS name, 'company' AS type
```

**Behavior**:

- UNION removes duplicate rows from the combined result (like SELECT DISTINCT in SQL)
- UNION ALL preserves all rows, including duplicates (more efficient when duplicates are acceptable)
- Results are returned in the order they are produced by each query

---

### CALL

The CALL clause invokes procedures and functions. This includes built-in database procedures and custom functions defined via SQL's DEFINE FUNCTION statement.

#### Syntax

```cypher
CALL procedureName([arguments])
[YIELD field [AS alias] [, field [AS alias] ...]]
[WHERE condition]
```

#### Built-in Procedures

| Procedure | Description |
|-----------|-------------|
| `db.labels()` | Returns all vertex type names (labels) |
| `db.relationshipTypes()` | Returns all edge type names |
| `db.propertyKeys()` | Returns all property keys defined in the schema |
| `db.schema()` | Returns schema visualization data |
| `db.schema.visualization()` | Returns schema visualization data (alias) |

#### Examples

```cypher
// List all vertex types (labels)
CALL db.labels()

// List all relationship types
CALL db.relationshipTypes()

// List all property keys
CALL db.propertyKeys()

// Get schema information
CALL db.schema()
```

#### Calling User Functions

User functions defined via SQL's DEFINE FUNCTION statement can be called using the CALL clause with their namespace:

```cypher
// Define user functions in different languages
DEFINE FUNCTION math.add "SELECT :a + :b" PARAMETERS [a,b] LANGUAGE sql
DEFINE FUNCTION js.multiply "return x * y" PARAMETERS [x,y] LANGUAGE js
DEFINE FUNCTION cypher.double "RETURN $n * 2" PARAMETERS [n] LANGUAGE openCypher

// Call user functions from Cypher using CALL clause
CALL math.add(3, 5)
// Returns: {result: 8}

CALL js.multiply(4, 2)
// Returns: {result: 8}

CALL cypher.double(5)
// Returns: {result: 10}

// Or call directly in expressions
RETURN math.add(3, 5) AS sum,
       js.multiply(4, 2) AS product,
       cypher.double(5) AS doubled
// Returns: {sum: 8, product: 8, doubled: 10}
```

#### Using YIELD

The YIELD clause filters and renames the columns returned by a procedure:

```cypher
// Get only the label column
CALL db.labels() YIELD label

// Rename the output column
CALL db.labels() YIELD label AS vertexType

// Filter results with WHERE
CALL db.labels() YIELD label
WHERE label STARTS WITH 'P'
RETURN label

// Yield all columns
CALL db.propertyKeys() YIELD *
```

#### Subqueries (CALL { ... })

Cypher supports subqueries using the CALL { ... } syntax. Subqueries are enclosed in curly braces and can import variables from the outer scope using WITH.

```cypher
// Basic subquery with imported variable
UNWIND [1, 2, 3] AS x
CALL {
  WITH x
  RETURN x * 10 AS y
}
RETURN x, y

// Subquery with MATCH
MATCH (n:Item)
CALL {
  WITH n
  RETURN n.value * 2 AS doubled
}
RETURN n.name, doubled

// Multiple imported variables
UNWIND [1, 2] AS a
UNWIND [10, 20] AS b
CALL {
  WITH a, b
  RETURN a + b AS sum
}
RETURN a, b, sum
```

#### Subqueries IN TRANSACTIONS

When importing large datasets (e.g., from LOAD CSV), you can use CALL { ... } IN TRANSACTIONS to commit changes in batches rather than accumulating all changes in a single transaction. This prevents running out of memory and allows progress to be committed incrementally.

```cypher
// Commit every 1000 rows (default batch size)
LOAD CSV WITH HEADERS FROM 'file:///data.csv' AS row
CALL {
  WITH row
  CREATE (n:Person {name: row.name, age: toInteger(row.age)})
} IN TRANSACTIONS

// Custom batch size of 500 rows
LOAD CSV WITH HEADERS FROM 'file:///large-dataset.csv' AS row
CALL {
  WITH row
  CREATE (n:Person {name: row.name, email: row.email})
} IN TRANSACTIONS OF 500 ROWS
```

**Important**: IN TRANSACTIONS should not be used inside an already open explicit transaction. The subquery manages its own transaction boundaries.

---

### FOREACH

The FOREACH clause iterates over a list and executes write operations (CREATE, SET, DELETE, MERGE) for each element. Unlike other clauses, FOREACH does not modify the result set - it passes through input rows unchanged.

#### Syntax

```cypher
FOREACH (variable IN list |
  update_clause [update_clause ...]
)
```

#### Examples

```cypher
// Create multiple nodes from a list
FOREACH (name IN ['Alice', 'Bob', 'Charlie'] |
  CREATE (:Person {name: name})
)

// Create relationships in a loop
CREATE (root:Root {name: 'Main'})
FOREACH (i IN [1, 2, 3] |
  CREATE (root)-[:HAS_ITEM]->(item:Item {id: i})
)

// Use with MATCH context
MATCH (p:Person)
FOREACH (tag IN ['developer', 'tester'] |
  CREATE (p)-[:HAS_TAG]->(t:Tag {value: tag})
)

// Update properties with SET
MATCH (item:Item)
WITH collect(item) AS items
FOREACH (item IN items |
  SET item.status = 'processed', item.timestamp = timestamp()
)

// FOREACH passes through input rows
CREATE (root:Root {name: 'test'})
FOREACH (i IN [1, 2] |
  CREATE (:Child {id: i})
)
RETURN root.name AS name
// Returns: {name: 'test'}
```

**Important Notes**:

- FOREACH can only contain write operations (CREATE, SET, DELETE, MERGE)
- FOREACH does not modify the result set - it passes input rows through unchanged
- Variables defined inside FOREACH are not visible outside the clause
- FOREACH is useful for batch operations and creating multiple elements from lists

---

### LOAD CSV

The LOAD CSV clause reads data from a CSV file (local or remote) and makes each row available as a variable for further query processing. This is the primary mechanism for bulk-importing data into ArcadeDB from CSV files.

#### Syntax

```cypher
LOAD CSV FROM <url> AS <variable>
LOAD CSV WITH HEADERS FROM <url> AS <variable>
LOAD CSV WITH HEADERS FROM <url> AS <variable> FIELDTERMINATOR <delimiter>
```

**Options**:

- **WITHOUT HEADERS**: Each row is bound as a list of strings. Access fields by index: `row[0]`, `row[1]`, etc.
- **WITH HEADERS**: The first row is treated as column names. Each subsequent row is bound as a map. Access fields by name: `row.name`, `row['age']`, etc.
- **FIELDTERMINATOR**: Specify a custom field delimiter (default is `,`). Example: `FIELDTERMINATOR ';'` for semicolon-separated files.

**Supported Sources**:

- **Local files**: `file:///path/to/file.csv` (subject to security configuration)
- **Remote URLs**: `https://example.com/data.csv` (always allowed)
- **Compressed files**: Transparently decompresses `.gz` (gzip) and `.zip` files

#### Basic Examples

```cypher
// Load CSV without headers (list access)
LOAD CSV FROM 'file:///people.csv' AS row
CREATE (:Person {name: row[0], age: toInteger(row[1])})

// Load CSV with headers (map access)
LOAD CSV WITH HEADERS FROM 'file:///people.csv' AS row
CREATE (:Person {name: row.name, age: toInteger(row.age)})

// Custom field delimiter
LOAD CSV WITH HEADERS FROM 'file:///data.tsv' AS row FIELDTERMINATOR '\t'
RETURN row.name, row.value

// Load from HTTP URL
LOAD CSV WITH HEADERS FROM 'https://example.com/data.csv' AS row
CREATE (:Item {id: row.id, label: row.label})
```

#### Bulk Import with CALL IN TRANSACTIONS

For large files, combine LOAD CSV with CALL { ... } IN TRANSACTIONS to commit in batches and prevent memory issues:

```cypher
LOAD CSV WITH HEADERS FROM 'file:///large-dataset.csv' AS row
CALL {
  WITH row
  CREATE (p:Person {name: row.name, email: row.email})
} IN TRANSACTIONS OF 1000 ROWS
```

#### CSV Quoting Rules

CSV fields follow RFC 4180 and support backslash escaping:

- Fields containing the delimiter, newlines, or quotes must be enclosed in double quotes
- Embedded quotes can be escaped as `""` (RFC 4180) or `\"` (backslash escaping)
- Quoted fields can span multiple lines

**Example CSV with special characters**:
```
name,email,bio
"Smith, John","john@example.com","Loves ""coding"" and coffee"
Alice,"alice@example.com","Has a newline
in bio"
```

#### Security Configuration

LOAD CSV file access is controlled by two configuration settings:

| Setting | Default | Description |
|---------|---------|-------------|
| `arcadedb.openCypher.loadCsv.allowFileUrls` | true | Allow access to local files via `file://` URLs and bare file paths. Set to false in multi-tenant server deployments. |
| `arcadedb.openCypher.loadCsv.importDirectory` | (empty) | Root directory for file imports. When set, file paths are resolved relative to this directory and path traversal (`../`) is blocked. Empty means no restriction. |

**Security Example** - Restrict imports to a specific directory:

```properties
arcadedb.openCypher.loadCsv.allowFileUrls=true
arcadedb.openCypher.loadCsv.importDirectory=/var/arcadedb/import
```

With this configuration:
```cypher
LOAD CSV WITH HEADERS FROM 'file:///data.csv' AS row RETURN row
// Resolves to /var/arcadedb/import/data.csv - OK

LOAD CSV WITH HEADERS FROM 'file:///../../../etc/passwd' AS row RETURN row
// Blocked - path traversal not allowed
```

#### Built-in Helper Functions

Two special functions are available within LOAD CSV queries:

- **`file()`** - Returns the URL of the currently loaded CSV file
- **`linenumber()`** - Returns the current line number being processed (1-based)

```cypher
// Track which file and line caused an error
LOAD CSV WITH HEADERS FROM 'file:///data.csv' AS row
RETURN file(), linenumber(), row
```

---

### CREATE CONSTRAINT

The CREATE CONSTRAINT clause defines schema constraints on node or relationship properties. Constraints enforce data integrity rules such as uniqueness, mandatory properties, and node keys.

#### Syntax

```cypher
CREATE CONSTRAINT [name] [IF NOT EXISTS]
FOR (variable:Label)
REQUIRE variable.property IS UNIQUE | IS NOT NULL | IS NODE KEY | IS KEY
```

**Constraint Types**:

- **IS UNIQUE** - Creates a unique LSM-Tree index on the specified property. Multiple properties can be combined for composite unique constraints.
- **IS NOT NULL** - Sets the property as mandatory in the schema.
- **IS NODE KEY** - Combines UNIQUE + NOT NULL. Creates a unique index and marks the property as mandatory.
- **IS KEY** - Alias for IS NODE KEY (ArcadeDB-specific).

#### Examples

```cypher
// Unique constraint on a single property
CREATE CONSTRAINT FOR (p:Person) REQUIRE p.email IS UNIQUE

// Named constraint with IF NOT EXISTS (idempotent)
CREATE CONSTRAINT unique_person_email IF NOT EXISTS
FOR (p:Person) REQUIRE p.email IS UNIQUE

// Composite unique constraint (multiple properties)
CREATE CONSTRAINT FOR (p:Person) REQUIRE (p.firstName, p.lastName) IS UNIQUE

// NOT NULL constraint (mandatory property)
CREATE CONSTRAINT FOR (p:Person) REQUIRE p.name IS NOT NULL

// Node key constraint (unique + mandatory)
CREATE CONSTRAINT FOR (p:Person) REQUIRE p.id IS NODE KEY

// Relationship constraint
CREATE CONSTRAINT FOR ()-[r:KNOWS]-() REQUIRE r.since IS NOT NULL
```

#### Important Notes

- **Property Auto-Creation**: If the property doesn't already exist in the schema, ArcadeDB automatically creates it as a STRING type. For other types, define the property first using SQL:
  ```sql
  ALTER TYPE Person CREATE PROPERTY email STRING
  ```

- **Idempotent Constraint Creation**: Use `IF NOT EXISTS` to make constraint creation idempotent. This prevents errors when running the same constraint creation query multiple times.

- **Constraint Names = Index Names**: In ArcadeDB, constraint names correspond to index names. Use backtick escaping for names containing special characters like brackets:
  ```cypher
  DROP CONSTRAINT `Person[email]`
  ```

---

### DROP CONSTRAINT

The DROP CONSTRAINT clause removes a previously created constraint by its index name.

#### Syntax

```cypher
DROP CONSTRAINT constraintName [IF EXISTS]
```

#### Examples

```cypher
// Drop a constraint by name
DROP CONSTRAINT 'Person[email]'

// Drop only if it exists (no error if missing)
DROP CONSTRAINT 'Person[email]' IF EXISTS
```

#### Important Notes

- **Constraint Names = Index Names**: In ArcadeDB, constraint names correspond to index names.
- **Backtick Escaping**: Use backtick escaping for constraint names containing special characters like brackets:
  ```cypher
  DROP CONSTRAINT `Person[email]`
  ```

- **IF EXISTS**: Makes the operation safe to run even when the constraint doesn't exist.

---

### User Management Clauses

These clauses manage database users in server mode.

#### SHOW USERS

Lists all configured users on the server and their database access.

**Note**: Requires ArcadeDB to be running in server mode. Does not work in embedded mode.

```cypher
SHOW USERS
```

**Result Columns**:
- `user` - The user name
- `databases` - The set of databases the user has access to (* means all databases)

#### SHOW CURRENT USER

Returns information about the currently authenticated user executing the query.

```cypher
SHOW CURRENT USER
```

**Result Columns**:
- `user` - The current user name
- `databases` - The set of databases the current user has access to

#### CREATE USER

Creates a new database user with the specified password. The new user is assigned to the `admin` group on all databases (*) by default.

**Note**: Requires ArcadeDB to be running in server mode.

```cypher
CREATE USER username SET PASSWORD 'password'
CREATE USER username IF NOT EXISTS SET PASSWORD 'password'
```

**Password Storage**: Passwords are hashed using PBKDF2 before storage - they are never stored in plaintext.

**Examples**:

```cypher
// Create a new user
CREATE USER bob SET PASSWORD 'SecurePass123!'

// Create user only if it doesn't already exist (idempotent)
CREATE USER bob IF NOT EXISTS SET PASSWORD 'SecurePass123!'
```

#### DROP USER

Removes a user from the server.

```cypher
DROP USER username
DROP USER username IF EXISTS
```

**Examples**:

```cypher
// Drop a user
DROP USER bob

// Drop only if it exists (no error if missing)
DROP USER bob IF EXISTS
```

#### ALTER USER

Changes the password of an existing user.

```cypher
ALTER USER username SET PASSWORD 'newPassword'
```

**Examples**:

```cypher
// Change a user's password
ALTER USER bob SET PASSWORD 'NewSecurePass456!'
```

---

## Expressions and Functions

### Literals

| Type | Syntax | Example |
|------|--------|---------|
| Integer | `123`, `-456` | `RETURN 42` |
| Float | `3.14`, `-2.5` | `RETURN 3.14159` |
| String | `'text'`, `"text"` | `RETURN 'Hello'` |
| Boolean | `true`, `false` | `RETURN true` |
| Null | `null` | `RETURN null` |
| List | `[1, 2, 3]` | `RETURN [1, 'a', true]` |
| Map | `{key: value}` | `RETURN {name: 'Alice', age: 30}` |

### Property Access

Access properties using dot notation:

```cypher
MATCH (p:Person)
RETURN p.name, p.age, p.address.city
```

### CASE Expression

Conditional logic in expressions.

#### Simple CASE

```cypher
MATCH (p:Person)
RETURN p.name,
  CASE
    WHEN p.age < 18 THEN 'minor'
    WHEN p.age < 65 THEN 'adult'
    ELSE 'senior'
  END AS ageGroup
```

#### Extended CASE

```cypher
MATCH (p:Person)
RETURN p.name,
  CASE p.status
    WHEN 'A' THEN 'Active'
    WHEN 'I' THEN 'Inactive'
    ELSE 'Unknown'
  END AS statusName
```

---

### Arithmetic Expressions

Perform arithmetic operations on numeric values.

| Operator | Description | Example |
|----------|-------------|---------|
| `+` | Addition (or string concatenation) | `RETURN 5 + 3`, `RETURN 'Hello' + ' ' + 'World'` |
| `-` | Subtraction | `RETURN 10 - 4` |
| `*` | Multiplication | `RETURN 6 * 7` |
| `/` | Division | `RETURN 15 / 3` |
| `%` | Modulo | `RETURN 17 % 5` |
| `^` | Power | `RETURN 2 ^ 8` |

**Examples with properties**:

```cypher
MATCH (p:Person)
RETURN p.name, p.age * 2 AS doubleAge

MATCH (p:Person)
RETURN p.name, p.salary / 12 AS monthlySalary

MATCH (p:Person)
RETURN p.name, (p.age * 2) + 10 AS result
```

---

### Map Literals

Create inline map (object) structures.

#### Syntax

```cypher
{key1: value1, key2: value2, ...}
```

**Simple map literal**:

```cypher
RETURN {name: 'Alice', age: 30}
```

**Empty map**:

```cypher
RETURN {}
```

**Map with expressions**:

```cypher
MATCH (p:Person)
WHERE p.name = 'Alice'
RETURN {
  personName: p.name,
  doubled: p.age * 2,
  nextYear: p.age + 1
} AS info
```

---

### List Comprehensions

Transform and filter lists using comprehension syntax.

#### Syntax

```cypher
[variable IN list WHERE condition | expression]
```

**Components**:

- `variable` - Iteration variable
- `list` - Source list or range
- `WHERE condition` - Optional filter (elements that don't match are excluded)
- `| expression` - Optional mapping expression (transforms each element)

#### Basic transformation

```cypher
// Double each number
RETURN [x IN [1, 2, 3] | x * 2]
// Returns: [2, 4, 6]

// Square numbers using range
RETURN [x IN range(1, 5) | x * x]
// Returns: [1, 4, 9, 16, 25]
```

#### With filter

```cypher
// Filter and transform
RETURN [x IN [1, 2, 3, 4, 5] WHERE x > 2 | x * 10]
// Returns: [30, 40, 50]

// Filter only (no transformation)
RETURN [x IN [1, 2, 3, 4, 5] WHERE x > 3]
// Returns: [4, 5]
```

---

### Map Projections

Extract specific properties from nodes or maps into a new map structure.

#### Syntax

```cypher
variable{.property1, .property2, key: expression, .*}
```

**Components**:

- `.property` - Include this property from the source
- `key: expression` - Add a computed value with the given key
- `.*` - Include all properties from the source

#### Property selection

```cypher
MATCH (p:Person)
WHERE p.name = 'Alice'
RETURN p{.name, .age} AS person
// Returns: {name: 'Alice', age: 30}
```

#### With computed values

```cypher
MATCH (p:Person)
WHERE p.name = 'Alice'
RETURN p{.name, doubleAge: p.age * 2} AS person
// Returns: {name: 'Alice', doubleAge: 60}
```

#### All properties

```cypher
MATCH (p:Person)
WHERE p.name = 'Alice'
RETURN p{.*} AS person
// Returns: {name: 'Alice', age: 30, salary: 50000}
```

---

### Aggregation Functions

Aggregation functions process multiple values and return a single result.

#### count()

Returns the number of values or rows.

```cypher
MATCH (p:Person)
RETURN count(p)  // Count all rows

MATCH (p:Person)
RETURN count(*)  // Count all rows (alternative syntax)

MATCH (p:Person)
RETURN count(p.email)  // Count non-null email values
```

#### sum()

Returns the sum of numeric values.

```cypher
MATCH (p:Person)
RETURN sum(p.salary)
```

#### avg()

Returns the average of numeric values.

```cypher
MATCH (p:Person)
RETURN avg(p.age)
```

#### min()

Returns the minimum value.

```cypher
MATCH (p:Person)
RETURN min(p.age) AS youngest
```

#### max()

Returns the maximum value.

```cypher
MATCH (p:Person)
RETURN max(p.age) AS oldest
```

#### collect()

Collects values into a list.

```cypher
MATCH (p:Person)
RETURN collect(p.name) AS allNames

MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
WITH p.name AS actor, collect(m.title) AS movies
RETURN actor, movies
```

---

### String Functions

#### toUpper()

Converts string to uppercase.

```cypher
RETURN toUpper('hello')  // Returns: 'HELLO'
```

#### toLower()

Converts string to lowercase.

```cypher
RETURN toLower('HELLO')  // Returns: 'hello'
```

#### trim()

Removes leading and trailing whitespace.

```cypher
RETURN trim('  hello  ')  // Returns: 'hello'
```

#### substring()

Extracts a substring.

**Syntax**: `substring(string, start [, length])`

```cypher
RETURN substring('hello world', 0, 5)  // Returns: 'hello'
RETURN substring('hello world', 6)  // Returns: 'world'
```

#### replace()

Replaces occurrences of a substring.

**Syntax**: `replace(string, search, replacement)`

```cypher
RETURN replace('hello', 'l', 'L')  // Returns: 'heLLo'
```

#### split()

Splits a string into a list.

**Syntax**: `split(string, delimiter)`

```cypher
RETURN split('a,b,c', ',')  // Returns: ['a', 'b', 'c']
```

#### left()

Returns the leftmost characters.

**Syntax**: `left(string, length)`

```cypher
RETURN left('hello', 3)  // Returns: 'hel'
```

#### right()

Returns the rightmost characters.

**Syntax**: `right(string, length)`

```cypher
RETURN right('hello', 3)  // Returns: 'llo'
```

---

### Math Functions

#### abs()

Returns the absolute value.

```cypher
RETURN abs(-42)  // Returns: 42
```

#### sqrt()

Returns the square root.

```cypher
RETURN sqrt(16)  // Returns: 4.0
```

#### ceil()

Rounds up to the nearest integer.

```cypher
RETURN ceil(3.2)  // Returns: 4
```

#### floor()

Rounds down to the nearest integer.

```cypher
RETURN floor(3.8)  // Returns: 3
```

#### round()

Rounds to the nearest integer.

```cypher
RETURN round(3.5)  // Returns: 4.0
```

#### rand()

Returns a random float between 0 and 1.

```cypher
RETURN rand()  // Returns: 0.7234...
```

---

### List Functions

#### size()

Returns the length of a list.

```cypher
RETURN size([1, 2, 3])  // Returns: 3
```

#### head()

Returns the first element of a list.

```cypher
RETURN head([1, 2, 3])  // Returns: 1
```

#### tail()

Returns all elements except the first.

```cypher
RETURN tail([1, 2, 3])  // Returns: [2, 3]
```

#### last()

Returns the last element of a list.

```cypher
RETURN last([1, 2, 3])  // Returns: 3
```

#### range()

Generates a list of integers.

**Syntax**: `range(start, end [, step])`

```cypher
RETURN range(1, 5)  // Returns: [1, 2, 3, 4, 5]
RETURN range(0, 10, 2)  // Returns: [0, 2, 4, 6, 8, 10]
```

#### reverse()

Reverses a list.

```cypher
RETURN reverse([1, 2, 3])  // Returns: [3, 2, 1]
```

---

### Node/Relationship Functions

#### id()

Returns the unique identifier of a node or relationship.

```cypher
MATCH (p:Person)
RETURN id(p) AS personId

MATCH (a)-[r:KNOWS]->(b)
RETURN id(r) AS relationshipId
```

#### labels()

Returns the labels (types) of a node as a list.

```cypher
MATCH (p:Person)
RETURN labels(p)  // Returns: ['Person']
```

#### type()

Returns the type of a relationship.

```cypher
MATCH (a)-[r:KNOWS]->(b)
RETURN type(r)  // Returns: 'KNOWS'
```

#### keys()

Returns the property keys of a node or relationship.

```cypher
MATCH (p:Person)
RETURN keys(p)  // Returns: ['name', 'age', 'email']
```

#### properties()

Returns all properties of a node or relationship as a map.

```cypher
MATCH (p:Person)
RETURN properties(p)  // Returns: {name: 'Alice', age: 30, email: 'alice@...'}
```

#### startNode()

Returns the start node of a relationship.

```cypher
MATCH (a)-[r:KNOWS]->(b)
RETURN startNode(r) AS startPerson
```

---

## Best Practices

### 1. Use Labels

Always specify labels in MATCH patterns for better performance:

```cypher
// Good - uses index on Person type
MATCH (p:Person {name: 'Alice'})
RETURN p

// Avoid - scans all nodes
MATCH (n {name: 'Alice'})
RETURN n
```

### 2. Create Indexes

Create indexes on frequently queried properties:

```cypher
CREATE INDEX ON Person (name)
CREATE INDEX ON Movie (title)
```

### 3. Use Parameters

For queries executed multiple times, use parameters to enable query plan caching:

```cypher
MATCH (p:Person)
WHERE p.name = $name AND p.age >= $minAge
RETURN p
```

### 4. Limit Results

Use LIMIT for exploratory queries:

```cypher
MATCH (p:Person)-[:KNOWS]*1..5->(other)
RETURN p.name, other.name
LIMIT 100
```

### 5. Use WITH for Complex Queries

Break complex queries into steps with WITH:

```cypher
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
WITH m.title AS movie, count(p) AS actorCount
WHERE actorCount > 2
RETURN movie, actorCount
```

### 6. Batch Operations

For bulk imports, combine LOAD CSV with CALL { ... } IN TRANSACTIONS:

```cypher
LOAD CSV WITH HEADERS FROM 'file:///large-dataset.csv' AS row
CALL {
  WITH row
  CREATE (p:Person {name: row.name, email: row.email})
} IN TRANSACTIONS OF 1000 ROWS
```

### 7. Handle Null Values

Always check for null values when needed:

```cypher
MATCH (p:Person)
WHERE p.email IS NOT NULL
RETURN p.name, p.email
```

---

## Extended Functions (APOC Compatible)

ArcadeDB provides 100+ additional functions organized by namespace. These are compatible with Neo4j's APOC library functions.

### Available Namespaces

| Namespace | Description | Example |
|-----------|-------------|---------|
| `agg.*` | Aggregation functions | `agg.median([1,2,3,4,5])` |
| `algo.*` | Graph algorithms | `CALL algo.dijkstra(...)` |
| `coll.*` | Collection operations | `coll.flatten([[1,2],[3,4]])` |
| `convert.*` | Type conversion | `convert.toJson({name: "John"})` |
| `date.*` | Date/time operations | `date.currentTimestamp()` |
| `map.*` | Map operations | `map.merge({a:1}, {b:2})` |
| `meta.*` | Schema introspection | `CALL meta.schema()` |
| `text.*` | String manipulation | `text.join(["a","b"], ",")` |
| `util.*` | Hashing, compression | `util.sha256("hello")` |
| `vector.*` | Vector/embeddings | `vector.cosinesimilarity(v1, v2)` |

See Extended Functions Reference for the complete reference.

---

## Key Differences from Neo4j's Cypher

While ArcadeDB's OpenCypher implementation is highly compatible with Neo4j, there are some intentional architectural differences:

### 1. Case-Insensitive Identifiers

**Neo4j**: Type and property names are case-sensitive
**ArcadeDB**: Follows SQL conventions - identifiers are case-insensitive

This ensures consistency across SQL, Cypher, Gremlin, and GraphQL query languages on the same data.

### 2. Multi-Model Database

**Neo4j**: Graph-only database
**ArcadeDB**: Unified graph, document, and key-value model

You can query the same data with SQL (document/relational), Cypher (graph), or Gremlin (traversal) in a single database.

### 3. Native Indexing

**Neo4j**: Pluggable index backends
**ArcadeDB**: Uses LSM-Tree indexes optimized for write-heavy workloads

LSM-Tree indexes provide superior performance for high-velocity data ingestion.

### 4. User-Defined Functions

**Neo4j**: Java-only compiled plugins (requires deployment, no hot-reload)
**ArcadeDB**: Functions in 4 languages (SQL, JavaScript, OpenCypher, Java) with hot-reload

Define functions on-the-fly without redeploying:
```cypher
DEFINE FUNCTION math.multiply "SELECT :a * :b" PARAMETERS [a,b] LANGUAGE sql
```

### 5. Query Language Flexibility

**Neo4j**: Cypher only
**ArcadeDB**: SQL, Cypher, Gremlin, GraphQL, MongoDB Query Language

Query the same graph data with the language you prefer.

---

## Summary

ArcadeDB's OpenCypher implementation provides a modern, high-performance graph query language with:

- **97.8% OpenCypher TCK compliance** (3,812 of 3,897 tests)
- **Native execution** comparable to SQL performance
- **Full CRUD support** with automatic transaction handling
- **Advanced pattern matching** including variable-length paths and optional matches
- **100+ built-in functions** plus extensibility via user-defined functions
- **Cost-based query optimization** with index selection and join reordering
- **Integration with SQL, Gremlin, GraphQL** on the same data

For production use, always use `openCypher` instead of the deprecated `cypher` implementation.
