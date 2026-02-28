# ArcadeDB SQL Reference Guide

## Table of Contents
1. [SQL Dialect Specifics](#sql-dialect-specifics)
2. [DDL Commands](#ddl-commands)
3. [DML Commands](#dml-commands)
4. [Graph-Specific SQL](#graph-specific-sql)
5. [Key Functions](#key-functions)
6. [Filtering Syntax](#filtering-syntax)

---

## SQL Dialect Specifics

### ArcadeDB SQL Overview
ArcadeDB provides a SQL dialect that extends standard SQL to support graph database operations. The dialect combines traditional relational SQL operations with graph-specific features like pattern matching and traversals.

### Key Differences from Standard SQL

1. **Graph Extensions**: ArcadeDB SQL includes MATCH statements for pattern matching and graph traversals, which are not part of standard SQL.

2. **Property-Based Access**: Records in ArcadeDB are structured with properties that are accessed directly in queries (e.g., `SELECT name, age FROM Person WHERE age > 30`).

3. **Vertex and Edge Types**: CREATE VERTEX and CREATE EDGE statements create type-specific records in the graph structure.

4. **Traversal Operators**: Arrow notation (`-->`, `<--`, etc.) is used to define graph traversal patterns in MATCH statements.

5. **Vector Operations**: ArcadeDB includes native vector index support and similarity functions for AI/ML use cases.

### Command Categories

| Category | Type | Commands |
|----------|------|----------|
| CRUD & Graph | DML | SELECT, INSERT, UPDATE, DELETE, MATCH, TRAVERSE |
| Schema & Buckets | DDL | CREATE TYPE, CREATE VERTEX, CREATE EDGE, CREATE PROPERTY, CREATE INDEX, CREATE BUCKET, ALTER TYPE, ALTER PROPERTY, DROP TYPE, DROP PROPERTY, DROP INDEX, DROP BUCKET |
| Database & Indexes | DDL/Admin | CREATE MATERIALIZED VIEW, ALTER MATERIALIZED VIEW, DROP MATERIALIZED VIEW, ALTER DATABASE, BACKUP DATABASE, RESTORE DATABASE, CHECK DATABASE, EXPORT DATABASE, IMPORT DATABASE |
| Planning & System | Admin | EXPLAIN, CONSOLE, ALIGN DATABASE |

---

## DDL Commands

### CREATE TYPE
Creates a new record type in the database.

**Syntax:**
```
CREATE TYPE <type-name>
[EXTENDS <parent-type>]
[ABSTRACT]
[BUCKET <bucket-name>]
```

**Example:**
```
CREATE TYPE Person EXTENDS V
CREATE TYPE Friendship EXTENDS E
CREATE TYPE Vehicle ABSTRACT
```

**Notes:**
- Use EXTENDS V for vertices or EXTENDS E for edges
- ABSTRACT types cannot have records created directly from them
- BUCKET option specifies where to store records of this type

### CREATE VERTEX
Creates a vertex type (specialized CREATE TYPE for vertices).

**Syntax:**
```
CREATE VERTEX <type-name>
[BUCKET <bucket-name>]
```

**Example:**
```
CREATE VERTEX Person
CREATE VERTEX Location BUCKET locations
```

### CREATE EDGE
Creates an edge type for relationships between vertices.

**Syntax:**
```
CREATE EDGE <type-name>
[BUCKET <bucket-name>]
```

**Example:**
```
CREATE EDGE Friendship
CREATE EDGE Works_For BUCKET relationships
```

### CREATE PROPERTY
Adds a property to an existing type.

**Syntax:**
```
CREATE PROPERTY <type-name>.<property-name> <data-type>
[DEFAULT <value>]
[NOT NULL]
[READONLY]
[MANDATORY]
[COLLATE <collation>]
[CUSTOM <key>=<value>]
```

**Data Types:**
- Primitive: BOOLEAN, BYTE, SHORT, INT, LONG, FLOAT, DOUBLE, DECIMAL, STRING, BINARY, DATE, DATETIME, TIME
- Collections: LIST, SET, MAP
- Special: EMBEDDED, LINK, LINKSET, LINKLIST, LINKMAP

**Example:**
```
CREATE PROPERTY Person.name STRING NOT NULL
CREATE PROPERTY Person.age INT DEFAULT 0
CREATE PROPERTY Person.email STRING UNIQUE
```

### ALTER PROPERTY
Modifies property constraints and settings.

**Syntax:**
```
ALTER PROPERTY <type-name>.<property-name>
[DEFAULT <value>]
[NOT NULL | NULLABLE]
[MANDATORY | OPTIONAL]
[READONLY | READWRITE]
```

**Example:**
```
ALTER PROPERTY Person.name MANDATORY
ALTER PROPERTY Person.age NOT NULL DEFAULT 0
```

### CREATE INDEX
Creates an index on property values for faster lookups.

**Syntax:**
```
CREATE INDEX <index-name>
ON <type-name> (<property-name> [, ...])
[TYPE <index-type>]
[UNIQUE]
[METADATA <metadata-key>=<metadata-value> [, ...]]
```

**Index Types:**
- UNIQUE: Prevents duplicate values
- NOTUNIQUE: Allows duplicate values
- UNIQUE_HASH: Hash-based unique index (faster for exact matches)
- FULL_TEXT: Full-text search index (for text properties)
- GEOSPATIAL: Geographic spatial indexing
- HNSW: Hierarchical Navigable Small World (for vector similarity search)
- LSM_VECTOR_INDEX: Log-Structured Merge vector index

**Example:**
```
CREATE INDEX idx_person_name ON Person (name)
CREATE INDEX idx_person_email ON Person (email) TYPE UNIQUE
CREATE INDEX idx_location_geom ON Location (geometry) TYPE GEOSPATIAL
CREATE INDEX idx_embedding ON Vector (embedding) TYPE HNSW METADATA m=16, ef=200
```

### CREATE TRIGGER
Creates a trigger to execute actions on events.

**Syntax:**
```
CREATE TRIGGER <trigger-name>
[BEFORE | AFTER] <event-type>
ON <type-name>
[WHERE <condition>]
[CONTENT <json>]
```

**Event Types:** CREATE, READ, UPDATE, DELETE

**Context Variables:**
- $event: The event object
- $this: The record being modified
- $old: Previous state of the record (UPDATE/DELETE)
- $new: New state of the record (UPDATE/CREATE)

**Example:**
```
CREATE TRIGGER audit_person_updates
AFTER UPDATE ON Person
CONTENT { "action": "log_update", "timestamp": "NOW" }

CREATE TRIGGER validate_age
BEFORE CREATE ON Person
WHERE $this.age < 0
CONTENT { "action": "prevent", "message": "Age cannot be negative" }
```

### CREATE MATERIALIZED VIEW
Creates a pre-computed view of query results.

**Syntax:**
```
CREATE MATERIALIZED VIEW <view-name>
FROM <query>
[INTERVAL <interval>]
[UPDATE STRATEGY <strategy>]
```

**Example:**
```
CREATE MATERIALIZED VIEW active_users
FROM SELECT FROM Person WHERE active = true

CREATE MATERIALIZED VIEW user_stats
FROM SELECT name, COUNT(*) as friends FROM Person
       LET $friends = (SELECT FROM Friendship WHERE FROM = $current)
INTERVAL 3600 seconds
```

### ALTER TYPE
Modifies type properties and settings.

**Syntax:**
```
ALTER TYPE <type-name>
[BUCKET <bucket-name>]
[SUPERTYPE <new-supertype>]
[CUSTOM <key>=<value>]
```

**Example:**
```
ALTER TYPE Person SUPERTYPE V
ALTER TYPE Vehicle BUCKET vehicles_bucket
```

### ALTER DATABASE
Modifies database-level settings.

**Syntax:**
```
ALTER DATABASE
SETTING <setting-name> = <value>
```

**Common Settings:**
- isolation: Transaction isolation level
- validation: Enable/disable validation
- localCacheSize: Size of local cache
- readCacheSize: Size of read cache

**Example:**
```
ALTER DATABASE SETTING isolation = SERIALIZABLE
ALTER DATABASE SETTING validation = true
```

### ALTER MATERIALIZED VIEW
Modifies a materialized view.

**Syntax:**
```
ALTER MATERIALIZED VIEW <view-name>
[INTERVAL <interval>]
[UPDATE STRATEGY <strategy>]
```

### DROP TYPE
Removes a type definition from the database.

**Syntax:**
```
DROP TYPE <type-name> [UNSAFE]
```

**Example:**
```
DROP TYPE Person
DROP TYPE Friendship UNSAFE
```

### DROP PROPERTY
Removes a property from a type.

**Syntax:**
```
DROP PROPERTY <type-name>.<property-name>
```

**Example:**
```
DROP PROPERTY Person.deprecated_field
```

### DROP INDEX
Removes an index.

**Syntax:**
```
DROP INDEX <index-name>
```

**Example:**
```
DROP INDEX idx_person_name
```

### DROP MATERIALIZED VIEW
Removes a materialized view.

**Syntax:**
```
DROP MATERIALIZED VIEW <view-name>
```

### CREATE BUCKET
Creates a new bucket to store records.

**Syntax:**
```
CREATE BUCKET <bucket-name>
[TYPE <bucket-type>]
[SELECTION STRATEGY <strategy>]
```

**Bucket Types:**
- PHYSICAL: Default bucket type
- MEMORY: In-memory bucket

**Selection Strategies:**
- ROUND_ROBIN: Distribute records evenly
- THREAD: Assign to current thread
- PARTITIONED: Hash-based partitioning

**Example:**
```
CREATE BUCKET users_bucket TYPE PHYSICAL
CREATE BUCKET temp_data TYPE MEMORY SELECTION STRATEGY ROUND_ROBIN
```

### DROP BUCKET
Removes a bucket.

**Syntax:**
```
DROP BUCKET <bucket-name>
```

---

## DML Commands

### SELECT
Retrieves records from the database.

**Syntax:**
```
SELECT [DISTINCT]
  <projection> [, ...]
FROM <target>
[LET <variable> = <expression> [, ...]]
[WHERE <condition>]
[GROUP BY <expression> [, ...]]
[HAVING <condition>]
[ORDER BY <expression> [ASC | DESC] [, ...]]
[SKIP <number> | OFFSET <number>]
[LIMIT <number>]
[RETURN <return-type>]
[LOCK <lock-type>]
[FETCHPLAN <fetchplan>]
[TIMEOUT <timeout>]
```

**Target:**
- Type name: `FROM Person`
- Bucket: `FROM BUCKET person_bucket`
- Index: `FROM INDEX idx_person_name`
- Subquery: `FROM (SELECT * FROM Person)`
- Graph pattern: Combined with MATCH

**Projections:**
- Simple: `SELECT name, age FROM Person`
- Aliased: `SELECT name as n, age as a FROM Person`
- Expression: `SELECT name, age * 2 as double_age FROM Person`
- Expansion: `SELECT *, name.toUpperCase() as upper_name`
- Nested: `SELECT name, friends: (SELECT FROM --> WHERE type = 'Friendship')`

**Example:**
```
SELECT name, age FROM Person WHERE age > 30 ORDER BY age DESC LIMIT 10

SELECT DISTINCT country FROM Person WHERE active = true

SELECT name, count(*) as friend_count
FROM Person
LET $friends = (SELECT FROM --> WHERE type = 'Friendship')
GROUP BY name
ORDER BY friend_count DESC

SELECT @rid, @type, name, (SELECT FROM --> WHERE type = 'Friendship') as friends
FROM Person
FETCHPLAN *:-1
```

### INSERT
Adds new records to the database.

**Syntax (SET):**
```
INSERT INTO <type>
(<property> [, ...])
VALUES (<value> [, ...])
[RETURN <return-type>]
```

**Syntax (CONTENT):**
```
INSERT INTO <type>
CONTENT <json-content>
[RETURN <return-type>]
```

**Syntax (FROM SELECT):**
```
INSERT INTO <type>
FROM <select-query>
[RETURN <return-type>]
```

**Return Types:**
- BEFORE: Original record before insert
- AFTER: Final record after insert
- COUNT: Number of inserted records

**Example:**
```
INSERT INTO Person (name, age, email)
VALUES ('John Doe', 30, 'john@example.com')

INSERT INTO Person CONTENT {
  "name": "Jane Doe",
  "age": 28,
  "email": "jane@example.com",
  "active": true
}

INSERT INTO Person
FROM SELECT name, age FROM ImportedData

INSERT INTO Friendship (from, to, type, since)
VALUES (#5:0, #6:1, 'friend', '2024-01-01')
RETURN AFTER
```

### UPDATE
Modifies existing records.

**Syntax:**
```
UPDATE <target>
SET <property> = <value> [, ...]
[INCR <property> [BY <amount>] [, ...]]
[ADD <property>, <value> [, ...]]
[PUT <map-property>, <key>, <value> [, ...]]
[REMOVE <property> [, ...]]
[WHERE <condition>]
[LIMIT <number>]
[RETURN <return-type>]
```

**Operations:**
- SET: Replace property value
- INCR: Increment numeric value
- ADD: Add value to collection
- PUT: Add key-value pair to map
- REMOVE: Delete property

**Example:**
```
UPDATE Person SET age = 31 WHERE name = 'John Doe'

UPDATE Person
SET active = false, last_updated = NOW()
WHERE age < 18

UPDATE Person
INCR friend_count BY 1
WHERE name = 'John Doe'

UPDATE Person
ADD interests, 'photography'
WHERE name = 'John Doe'

UPDATE Person
SET metadata = {'key': 'value', 'status': 'updated'}
WHERE @rid = #5:0
RETURN AFTER
```

### DELETE
Removes records from the database.

**Syntax:**
```
DELETE FROM <target>
[WHERE <condition>]
[LIMIT <number>]
[RETURN <return-type>]
```

**Example:**
```
DELETE FROM Person WHERE age > 100

DELETE FROM Friendship WHERE created < '2020-01-01'

DELETE FROM Person WHERE name = 'John Doe'
LIMIT 1
RETURN BEFORE

DELETE FROM Person
WHERE @rid IN (SELECT FROM Person WHERE inactive = true)
LIMIT 1000
```

### TRAVERSE
Traverses relationships in the graph following a path.

**Syntax:**
```
TRAVERSE <traversal-expression>
FROM <target>
[WHERE <condition>]
[LIMIT <number>]
[MAXDEPTH <depth>]
[AMBIGUOUS {BREADTH_FIRST | DEPTH_FIRST}]
[STRATEGY {BREADTH_FIRST | DEPTH_FIRST}]
```

**Context Variables:**
- $depth: Current traversal depth
- $path: Path taken to reach current record
- $traversed: Set of traversed records

**Example:**
```
TRAVERSE --> FROM Person WHERE name = 'John Doe'

TRAVERSE -->[type='Friendship'] FROM Person
MAXDEPTH 3
WHERE $depth > 0

TRAVERSE --> FROM #5:0
STRATEGY BREADTH_FIRST
WHERE @type = 'Person'
```

---

## Graph-Specific SQL

### MATCH Statement
Advanced pattern matching for graph traversals and relationship queries.

**Simplified Syntax:**
```
MATCH
<pattern1> [, <pattern2> ...]
[WHERE <condition>]
[RETURN <projection>]
[ORDER BY <expression>]
[LIMIT <number>]
```

**Pattern Syntax:**

Basic pattern:
```
(variable:type {property: value})
```

Relationship pattern:
```
(a) --> (b)              # Outgoing edge
(a) <-- (b)              # Incoming edge
(a) --> (b) --> (c)      # Path of length 2
(a) --> {type:'Friend'} --> (b)  # Typed edge
```

**Arrow Notation:**
- `-->` : Outgoing edge (any type)
- `<--` : Incoming edge (any type)
- `-[edge:Type]->` : Specific edge type
- `-[:Type]->` : Named edge type
- `-->*` : Multiple hops
- `-->?` : Optional (0 or 1)

**Context Variables:**
- `$current`: Current vertex in traversal
- `$matched`: Complete matched pattern
- `$currentMatch`: Current match iteration
- `$depth`: Depth in traversal

**Examples:**

Simple pattern:
```
MATCH (person:Person {name: 'John'})
RETURN person.name, person.age
```

Two-vertex pattern:
```
MATCH (a:Person), (b:Person)
WHERE a.age > b.age
RETURN a.name as older, b.name as younger
```

Relationship pattern:
```
MATCH (a:Person) --> (b:Person)
RETURN a.name, b.name

MATCH (a:Person) -[friendship:Friendship]-> (b:Person)
RETURN a.name, friendship.since, b.name
```

Path pattern with depth:
```
MATCH (start:Person {name: 'John'}) --> {1,3} (friend:Person)
WHERE $depth > 0
RETURN start.name, friend.name, $depth as distance
```

Negative patterns:
```
MATCH (a:Person)
WHERE NOT ((a) --> (b:Person WHERE b.blocked = true))
RETURN a.name
```

Complex multi-step traversal:
```
MATCH
  (person:Person {name: 'John'})
  --> (friend:Person)
  --> (friend_of_friend:Person)
WHERE person != friend_of_friend
RETURN person.name, friend.name, friend_of_friend.name
```

Graph expansion in projections:
```
MATCH (person:Person)
RETURN
  person.name,
  person.age,
  friends: (SELECT FROM --> WHERE @type = 'Friendship') as relationships
```

**Use Cases:**
1. **Recommendation Systems**: Find friends-of-friends
2. **Social Networks**: Path finding and influence analysis
3. **Knowledge Graphs**: Entity relationship discovery
4. **Fraud Detection**: Suspicious pattern identification
5. **Network Analysis**: Connectivity and centrality metrics

---

## Key Functions

### Graph Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `traverseV()` | Get vertices connected to current record | `traverseV(direction, filter)` |
| `traverseE()` | Get edges connected to current record | `traverseE(direction, filter)` |
| `in()` | Get incoming edges | `in(edgeType)` |
| `out()` | Get outgoing edges | `out(edgeType)` |
| `inV()` | Get vertices from incoming edges | `inV(edgeType)` |
| `outV()` | Get vertices from outgoing edges | `outV(edgeType)` |
| `both()` | Get bidirectional edges | `both(edgeType)` |
| `bothV()` | Get vertices from bidirectional edges | `bothV(edgeType)` |
| `connected()` | Check if vertex is connected | `connected(vertex, edgeType)` |
| `distance()` | Calculate shortest path distance | `distance(vertex1, vertex2)` |
| `path()` | Find path between vertices | `path(vertex1, vertex2)` |

### Vector Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `similarity()` | Calculate vector similarity (cosine) | `similarity(vector1, vector2)` |
| `euclidean_distance()` | Calculate Euclidean distance | `euclidean_distance(vector1, vector2)` |
| `manhattan_distance()` | Calculate Manhattan distance | `manhattan_distance(vector1, vector2)` |
| `vector_search()` | Search similar vectors in HNSW index | `vector_search(indexName, queryVector, limit)` |

### Collection Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `size()` | Get collection size | `size(collection)` |
| `length()` | Get string/collection length | `length(value)` |
| `contains()` | Check if collection contains value | `contains(collection, value)` |
| `containsAll()` | Check if contains all values | `containsAll(collection, values)` |
| `containsAny()` | Check if contains any value | `containsAny(collection, values)` |
| `join()` | Join collection into string | `join(collection, separator)` |
| `split()` | Split string into collection | `split(string, separator)` |
| `append()` | Append value to collection | `append(collection, value)` |
| `remove()` | Remove value from collection | `remove(collection, value)` |
| `reverse()` | Reverse collection | `reverse(collection)` |
| `first()` | Get first element | `first(collection)` |
| `last()` | Get last element | `last(collection)` |
| `filter()` | Filter collection with condition | `filter(collection, condition)` |
| `map()` | Map collection values | `map(collection, expression)` |
| `reduce()` | Reduce collection to single value | `reduce(collection, expression)` |

### Math Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `abs()` | Absolute value | `abs(number)` |
| `round()` | Round number | `round(number, decimals)` |
| `floor()` | Round down | `floor(number)` |
| `ceil()` | Round up | `ceil(number)` |
| `min()` | Minimum of collection or values | `min(collection)` or `min(a, b, ...)` |
| `max()` | Maximum of collection or values | `max(collection)` or `max(a, b, ...)` |
| `sum()` | Sum of collection | `sum(collection)` |
| `avg()` | Average of collection | `avg(collection)` |
| `sqrt()` | Square root | `sqrt(number)` |
| `power()` | Raise to power | `power(base, exponent)` |
| `mod()` | Modulo operation | `mod(number, divisor)` |
| `random()` | Random number | `random()` |
| `sin()`, `cos()`, `tan()` | Trigonometric functions | `sin(radians)` |
| `log()`, `log10()`, `ln()` | Logarithmic functions | `log(number)` |

### String Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `toUpperCase()` | Convert to uppercase | `string.toUpperCase()` |
| `toLowerCase()` | Convert to lowercase | `string.toLowerCase()` |
| `trim()` | Trim whitespace | `string.trim()` |
| `substring()` | Extract substring | `substring(string, start, end)` |
| `indexOf()` | Find substring position | `indexOf(string, substring)` |
| `replace()` | Replace substring | `replace(string, search, replacement)` |
| `startsWith()` | Check string start | `startsWith(string, prefix)` |
| `endsWith()` | Check string end | `endsWith(string, suffix)` |
| `matches()` | Regex match | `matches(string, regex)` |
| `format()` | Format string | `format(pattern, values)` |
| `concat()` | Concatenate strings | `concat(string1, string2, ...)` |

### Date/Time Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `now()` | Current timestamp | `now()` |
| `sysdate()` | Current date | `sysdate()` |
| `date()` | Create date from components | `date(year, month, day)` |
| `dateTime()` | Create datetime | `dateTime(year, month, day, hour, minute, second)` |
| `format()` | Format date/time | `format(datetime, pattern)` |
| `parse()` | Parse string to date | `parse(string, pattern)` |

### Type/Conversion Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `asString()` | Convert to string | `asString(value)` |
| `asInteger()` | Convert to integer | `asInteger(value)` |
| `asFloat()` | Convert to float | `asFloat(value)` |
| `asBoolean()` | Convert to boolean | `asBoolean(value)` |
| `asDate()` | Convert to date | `asDate(value)` |
| `type()` | Get type name | `type(value)` |
| `className()` | Get class name | `className(record)` |

### Aggregate Functions

| Function | Description | Usage |
|----------|-------------|-------|
| `count()` | Count records | `COUNT(*)`, `COUNT(field)`, `COUNT(DISTINCT field)` |
| `sum()` | Sum values | `SUM(field)` |
| `avg()` | Average values | `AVG(field)` |
| `min()` | Minimum value | `MIN(field)` |
| `max()` | Maximum value | `MAX(field)` |
| `group_concat()` | Concatenate values | `GROUP_CONCAT(field, separator)` |
| `percentile()` | Percentile value | `PERCENTILE(field, percentage)` |

### Example Function Usage

```
-- String functions
SELECT name.toUpperCase() as upper_name FROM Person

-- Collection functions
SELECT name, interests.size() as interest_count
FROM Person
WHERE interests.contains('photography')

-- Math functions
SELECT name, age, round(salary / 1000, 2) as salary_k
FROM Employee
WHERE salary > avg(salary)

-- Graph functions
SELECT name, inV('Friendship').size() as friend_count
FROM Person

-- Vector functions
SELECT name, vector_search('embedding_index', @embedding, 10) as similar
FROM Documents

-- Date functions
SELECT name, format(created, 'yyyy-MM-dd') as created_date
FROM Person
WHERE created > NOW() - INTERVAL 30 days

-- Type conversion
SELECT asString(@rid) as rid_string, asInteger(age) as age_int
FROM Person
```

---

## Filtering Syntax

### WHERE Clause Overview
The WHERE clause filters records based on conditions. Multiple conditions can be combined with AND/OR operators.

**Syntax:**
```
WHERE <condition> [AND|OR <condition> ...]
```

### Comparison Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Equals | `WHERE age = 30` |
| `!=` or `<>` | Not equals | `WHERE status != 'inactive'` |
| `<` | Less than | `WHERE age < 30` |
| `<=` | Less than or equal | `WHERE age <= 30` |
| `>` | Greater than | `WHERE age > 30` |
| `>=` | Greater than or equal | `WHERE age >= 30` |
| `<=>` | Null-safe equals | `WHERE age <=> NULL` |

### Pattern Matching Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `LIKE` | Case-sensitive pattern match (uses `%` and `_`) | `WHERE name LIKE 'John%'` |
| `ILIKE` | Case-insensitive pattern match | `WHERE name ILIKE 'john%'` |
| `NOT LIKE` | Negative case-sensitive pattern match | `WHERE name NOT LIKE 'Admin%'` |
| `NOT ILIKE` | Negative case-insensitive pattern match | `WHERE name NOT ILIKE 'test%'` |
| `MATCHES` | Regular expression match | `WHERE email MATCHES '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'` |

**Pattern Examples:**
```
WHERE email LIKE '%@example.com'        -- Ends with @example.com
WHERE name LIKE 'J%n'                  -- Starts with J, ends with n
WHERE phone LIKE '___-____'             -- Format: XXX-XXXX
WHERE email ILIKE 'john%'               -- Case-insensitive prefix
WHERE status MATCHES '(active|pending)' -- Regex match
```

### Logical Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `AND` | All conditions must be true | `WHERE age > 18 AND status = 'active'` |
| `OR` | At least one condition must be true | `WHERE status = 'admin' OR role = 'moderator'` |
| `NOT` | Negate condition | `WHERE NOT (status = 'deleted')` |

### Null Checking

| Operator | Description | Example |
|----------|-------------|---------|
| `IS NULL` | Property is null | `WHERE email IS NULL` |
| `IS NOT NULL` | Property is not null | `WHERE email IS NOT NULL` |

### Collection Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `IN` | Value in collection | `WHERE status IN ('active', 'pending')` |
| `NOT IN` | Value not in collection | `WHERE id NOT IN (1, 2, 3)` |
| `CONTAINS` | Collection contains value | `WHERE tags CONTAINS 'important'` |
| `CONTAINSALL` | Collection contains all values | `WHERE tags CONTAINSALL ['python', 'javascript']` |
| `CONTAINSANY` | Collection contains any value | `WHERE tags CONTAINSANY ['python', 'java']` |
| `CONTAINSKEY` | Map contains key | `WHERE metadata CONTAINSKEY 'version'` |
| `CONTAINSVALUE` | Map contains value | `WHERE metadata CONTAINSVALUE 'v1.0'` |
| `CONTAINSTEXT` | Full-text search | `WHERE content CONTAINSTEXT 'search term'` |

### Type Checking

| Operator | Description | Example |
|----------|-------------|---------|
| `INSTANCEOF` | Check if record is instance of type | `WHERE @type INSTANCEOF 'Person'` |

### Range and Arithmetic

| Operator | Description | Example |
|----------|-------------|---------|
| `BETWEEN` | Value within range (inclusive) | `WHERE age BETWEEN 18 AND 65` |
| Arithmetic: `+`, `-`, `*`, `/`, `%` | Math operations in conditions | `WHERE salary * 1.1 > 50000` |

### Complex Filtering Examples

**Multiple conditions:**
```
SELECT * FROM Person
WHERE age > 25
  AND status = 'active'
  AND (role = 'admin' OR role = 'moderator')
```

**Nested conditions:**
```
SELECT * FROM Person
WHERE (age > 25 AND status = 'active')
  OR (age > 65 AND status = 'retired')
```

**Null-safe comparisons:**
```
SELECT * FROM Person
WHERE department <=> NULL OR department = 'Sales'
```

**Pattern matching with wildcards:**
```
SELECT * FROM Person
WHERE email LIKE '%@%'
  AND name LIKE '[A-Z]%'
  AND phone NOT LIKE '%x%'
```

**Collection filtering:**
```
SELECT * FROM Person
WHERE tags CONTAINSALL ['verified', 'active']
  AND interests CONTAINSANY ['coding', 'design', 'music']
```

**Full-text search:**
```
SELECT * FROM Article
WHERE content CONTAINSTEXT 'machine learning'
  AND CONTAINSTEXT 'neural networks'
```

**Graph-based filtering:**
```
SELECT * FROM Person
WHERE (-->).size() > 5
  AND NOT ((-->) CONTAINS (type = 'blocked'))
```

**Type-specific filtering:**
```
SELECT * FROM V
WHERE @type INSTANCEOF 'Person'
  AND @version >= 2
```

**Recursive conditions:**
```
SELECT * FROM Person
WHERE age > 30
  AND (SELECT COUNT(*) FROM --> WHERE @type = 'Friendship') > 10
```

### Filtering Performance Tips

1. **Use indexes**: Filter on indexed properties for faster lookups
```
SELECT * FROM Person WHERE email = 'john@example.com'  -- Indexed
```

2. **Combine with LIMIT**: Stop searching after finding enough results
```
SELECT * FROM Person WHERE status = 'active' LIMIT 100
```

3. **Use type constraints**: Filter by type first
```
SELECT * FROM Person WHERE @type = 'Person' AND age > 30
```

4. **Avoid full table scans**: Use indexed columns in WHERE
```
-- Bad: Forces full scan
WHERE UPPER(name) = 'JOHN'

-- Good: Uses index
WHERE name = 'John'
```

5. **Use INSTANCEOF for polymorphic queries**:
```
SELECT * FROM V WHERE @type INSTANCEOF 'Animal'
```

---

## Additional Reference

### Projection Techniques

**Simple projection:**
```
SELECT name, age FROM Person
```

**Aliased projection:**
```
SELECT name as full_name, age as current_age FROM Person
```

**Expression projection:**
```
SELECT name, age * 2 as double_age, salary / 12 as monthly FROM Employee
```

**Expand projection:**
```
SELECT *, name.toUpperCase() as upper_name FROM Person
```

**Nested projection:**
```
SELECT
  name,
  age,
  friends: (SELECT FROM --> WHERE @type = 'Friendship')
FROM Person
```

**Conditional projection (CASE):**
```
SELECT
  name,
  CASE
    WHEN age < 18 THEN 'minor'
    WHEN age < 65 THEN 'adult'
    ELSE 'senior'
  END as age_group
FROM Person
```

### Pagination Techniques

**SKIP-LIMIT approach:**
```
SELECT * FROM Person
ORDER BY name
SKIP 20
LIMIT 10
```

**RID-LIMIT approach (more efficient):**
```
SELECT * FROM Person
WHERE @rid > #5:100
ORDER BY @rid
LIMIT 10
```

### Transaction Control

**Explicit transaction:**
```
BEGIN
INSERT INTO Person CONTENT {"name": "John"}
UPDATE Person SET status = 'active'
COMMIT
```

**Rollback on error:**
```
BEGIN
DELETE FROM Account WHERE balance < 0
ROLLBACK
```

### System Information

**View database statistics:**
```
SELECT @type, COUNT(*) as count FROM V GROUP BY @type
```

**Check record metadata:**
```
SELECT @rid, @type, @version, @class FROM Person LIMIT 1
```

**List all types:**
```
SELECT FROM METADATA:SCHEMA WHERE @type = 'Type'
```

---

## Notes and Best Practices

1. **Always use parameterized queries** in application code to prevent injection attacks
2. **Index frequently filtered properties** for better query performance
3. **Use type constraints** (@type = 'TypeName') for polymorphic hierarchies
4. **Leverage materialized views** for complex, repeated queries
5. **Use LIMIT** when exploring large datasets to avoid timeouts
6. **Test with EXPLAIN** to understand query execution plans
7. **Use appropriate lock types** (LOCK READ, LOCK WRITE) in transactions
8. **Normalize data** using BUCKET strategies for distributed performance
9. **Keep vector dimensions reasonable** for similarity search performance
10. **Monitor query performance** with EXPLAIN and adjust indexes accordingly
