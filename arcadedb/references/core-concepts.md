# ArcadeDB Core Concepts Reference

Comprehensive guide to ArcadeDB's core concepts and features from the official manual.

## Table of Contents

1. [Record](#record)
2. [Types](#types)
3. [Buckets](#buckets)
4. [Relationships](#relationships)
5. [Database](#database)
6. [Transactions](#transactions)
7. [Inheritance](#inheritance)
8. [Schema](#schema)
9. [Indexes](#indexes)
10. [Full-Text Index](#full-text-index)
11. [Geospatial Index](#geospatial-index)
12. [Materialized Views](#materialized-views)
13. [Graph Database](#graph-database)
14. [Time Series](#time-series)

---

## Record

A record is the smallest unit you can load from and store in the database. Records come in three types:

### Document
Documents are softly typed and defined by schema types, but can also be used in a schema-less mode. They handle fields flexibly and can be imported/exported in JSON format.

Example:
```json
{
  "name":"Jay",
  "surname":"Miner",
  "job":"Developer",
  "creations":[
    {
      "name":"Amiga 1000",
      "company":"Commodore Inc."
    },
    {
      "name":"Amiga 500",
      "company":"Commodore Inc."
    }
  ]
}
```

### Vertex
In graph databases, vertices (also called nodes) represent the main entity holding information. A vertex can be a patient, company, or product. Vertices are documents with additional features: they can contain embedded records and arbitrary properties, and are connected with other vertices through edges.

### Edge
An edge (or arc) is the connection between two vertices. Edges can be unidirectional and bidirectional, connecting only two vertices. Like vertices, edges are documents with additional features.

### Record ID (RID)

When ArcadeDB generates a record, it auto-assigns a unique identifier called a Record ID (RID). The syntax is:

```
#<bucket-identifier>:<record-position>
```

**Components:**
- **bucket-identifier**: Number indicating the bucket ID to which the record belongs. Positive numbers indicate persistent records. You can have up to 2,147,483,648 buckets in a database.
- **record-position**: Number defining the absolute position of the record in the bucket.

Special case: `#-1:-1` symbolizes the null RID.

**Key points:**
- The prefix character `#` is mandatory
- Each Record ID is immutable, universal, and only reused when configured
- Records can be accessed directly through their RIDs at O(1) complexity
- Query speed is constant and unaffected by database size
- No need to create a field to serve as primary key (unlike relational databases)

### Record Retrieval Complexity

Retrieving a record by RID is O(1) complexity. This is possible because the RID encodes both the file a record is stored in and the position inside it.

**Storage organization:**
- Bucket files are organized in pages (default size: 64KB)
- Default maximum records per page: 2048
- Byte position calculation in pseudo-code:

```
int pageId = floor(rid.getPosition() / maxRecordsInPage);
int positionInPage = floor(rid.getPosition() % maxRecordsInPage);
```

---

## Types

The concept of type is taken from Object Oriented Programming (OOP). In ArcadeDB, types define records. They are closest to the concept of a "Table" in relational databases and a "Class" in an object database.

**Type characteristics:**
- Can be schema-less, schema-full, or a mix
- Can inherit from other types, creating a tree of types (Inheritance)
- Each type has its own buckets (data files)
- A type can support multiple buckets
- When executing a query against a type, it automatically fetches from all associated buckets
- When creating a new record, ArcadeDB selects the bucket using a configurable strategy

### Type Configuration

By default, ArcadeDB creates one bucket per type, but can be configured to create multiple buckets. When you execute a query against a type with subtypes, ArcadeDB searches the buckets of the target type and all subtypes.

**Default behavior:**
- ArcadeDB creates one bucket per type
- Bucket naming: `<type-name>_<sequential-number>` (starting from 0)
- Example: `Beer` type → `Beer_0` bucket file; `Beer_31` bucket file in file system

**Performance optimization:**
For massive inserts, create additional buckets (one per CPU core) to achieve parallelism with zero contention between CPUs/cores.

**Query types:**
```sql
SELECT FROM schema:types
```

---

## Buckets

Buckets provide physical or in-memory space where ArcadeDB stores data. Each bucket is one file at the file system level. Buckets are comparable to:
- "Collection" in Document databases
- "Table" in Relational databases
- "Cluster" in OrientDB

**Bucket characteristics:**
- Each bucket can only be part of one type
- A type can support multiple buckets
- Two types cannot share the same bucket
- Sub-types have their separate buckets from their super-types
- Up to 2,147,483,648 buckets possible in a database

### Types vs. Buckets in Queries

The combination of types and buckets is powerful with many use cases. In most cases, you work with types and you'll be fine. But if you can split your database into multiple buckets, you can address a specific bucket instead of the entire type.

**Performance considerations:**
- Querying by bucket is faster than querying by type
- Indexes slow down insertion and take disk space/RAM
- In some use cases, you can totally or partially avoid using indexes by wisely dividing your database

### One Bucket Per Period

Example: Create an `Invoice` type with one bucket per year: `Invoice_2015` and `Invoice_2016`.

Query all invoices:
```sql
SELECT FROM Invoice
```

Filter by year:
```sql
SELECT FROM Invoice WHERE year = 2016
```

Query specific bucket:
```sql
SELECT FROM BUCKET:Invoice_2016
```

By using explicit bucket notation, queries run significantly faster because ArcadeDB narrows the search to the targeted bucket. No index is needed on the year because all invoices for year 2016 are stored in the `Invoice_2016` bucket.

### One Bucket Per Location

Split records by location creating separate buckets:
```sql
CREATE BUCKET Customer_Europe
CREATE BUCKET Customer_Americas
CREATE BUCKET Customer_Asia
CREATE BUCKET Customer_Other

CREATE VERTEX TYPE Customer BUCKET Customer_Europe,Customer_Americas,Customer_Asia,Customer_Other
```

Store vertices in the right bucket:
```sql
INSERT INTO BUCKET:Customer_Europe CONTENT { firstName: 'Enzo', lastName: 'Ferrari' }
```

Query by location:
```sql
SELECT FROM BUCKET:Customer_Europe
```

Query multiple buckets:
```sql
SELECT FROM BUCKET:[Customer_Europe_Italy,Customer_Europe_Spain]
```

---

## Relationships

ArcadeDB supports three kinds of relationships: **connections**, **referenced**, and **embedded**. It can manage relationships in schema-full or schema-less scenarios.

### Graph Connections

As a graph database, ArcadeDB expresses connections between records using edges spanning between vertices. This is the graph model's natural way of relationships and is traversable by SQL, Gremlin, and Cypher query languages.

Internally, ArcadeDB deposits a direct (referenced) relationship for edge-wise connected vertices, ensuring fast graph traversals.

**Example:**
```
Vertex A (TYPE Customer, RID #5:23)
    ↓ (Edge X, TYPE isLinked, RID #16:9)
Vertex B (TYPE Invoice, RID #10:2)
```

Create edges via SQL:
```sql
CREATE EDGE
```

### Referenced Relationships

In Relational databases, tables are linked through JOIN commands, which prove costly on computing resources. ArcadeDB manages relationships natively without computing a JOIN, but by storing a direct LINK to the target object of the relationship. This boosts load speed for the entire graph of connected objects.

**Key difference from edges:** References are properties connecting any record, while edges are types connecting vertices. Graph traversal applies only to edges.

**Example:**
```
Record A (TYPE Customer, RID #5:23)
    → [Link to Record B] (TYPE Invoice, RID #10:2)
```

### Embedded Relationships

When using Embedded relationships, ArcadeDB stores the relationship within the record that embeds it. These relationships are stronger than Reference relationships, similar to UML Composition relationship.

**Key characteristics:**
- Embedded records do not have their own RID
- Only accessible through the container record
- Stored inside the embedding record
- If the container record is deleted, the embedded record is also deleted
- Represents UML Composition relationship

**Example:**
```
Record A (TYPE Account, RID #5:23)
    ├─ address (embedded Record B, TYPE Address, NO RID)
```

Query embedded records:
```sql
SELECT FROM Account WHERE address.city = 'Rome'
```

### 1:1 and n:1 Embedded Relationships

Use the EMBEDDED type.

### 1:n and n:n Embedded Relationships

Use a list or map of links:
- **LIST**: An ordered list of records
- **MAP**: An ordered map of records as value and a string as key (no duplicate keys)

### Inverse Relationships

In ArcadeDB, all edges in the graph model are bidirectional. This differs from the document model where relationships are always unidirectional. ArcadeDB automatically maintains the consistency of all bidirectional relationships.

### Edge Constraints

ArcadeDB supports edge constraints, limiting the admissible vertex types connectable by an edge type. Define @in and @out metadata properties explicitly:

```sql
CREATE EDGE TYPE HasParts;
CREATE PROPERTY HasParts.'@out' link OF Product;
CREATE PROPERTY HasParts.'@in' link OF Component;
```

This edge type `HasParts` connects only from vertices of type `Product` to vertices of type `Component`.

### Relationship Traversal Complexity

As a native graph database, ArcadeDB supports index-free adjacency. This means constant graph traversal complexity of O(1), independent of graph expansion (database size).

To traverse a graph structure, follow references stored by the current record. References are always RIDs and are not only pointers to incoming and outgoing edges, but also to connected vertices. References are managed by a stack (also known as LIFO), allowing you to get the latest insertion first. This is useful if edges are used purely to connect vertices and do not carry properties themselves.

---

## Database

Each server or Java VM can handle multiple database instances, but the database name must be unique.

### Database URL

ArcadeDB uses its own URL format:

```
<engine>:<db-name>
```

The embedded engine is the default and can be omitted. To open a database on the local file system directly:

```
<path-as-url>
```

### Database Usage

You must always close the database once you finish working on it.

**Important note:**
ArcadeDB automatically closes all opened databases when the process dies gracefully (not by killing it by force). This is assured if the operating system allows a graceful shutdown. For example, on Unix/Linux systems using SIGTERM, or in Docker exit code 143 instead of SIGKILL, or in Docker exit code 137.

---

## Transactions

A transaction comprises a unit of work performed within a database management system (or similar system) against a database, and treated in a coherent and reliable way independent of other transactions.

### Transaction Purposes

Transactions serve two main purposes:
1. Provide reliable units of work that allow correct recovery from failures and keep a database consistent even in cases of system failure, when execution stops (completely or partially) and many operations upon a database remain uncompleted
2. Provide isolation between programs accessing a database concurrently

### ACID Properties

A database transaction, by definition, must be **atomic**, **consistent**, **isolated**, and **durable**. Database practitioners often refer to these properties of database transactions using the acronym ACID.

**ArcadeDB is ACID compliant.**

**Key point:** ArcadeDB keeps the transaction in the host's RAM, so the transaction size is affected by the available RAM (Heap memory) on JVM. For transactions involving many records, consider splitting it into multiple transactions.

### ACID Properties Explained

#### Atomicity
"Atomicity requires that each transaction is 'all or nothing': if one part of the transaction fails, the entire transaction fails, and the database state is left unchanged. An atomic system must guarantee atomicity in each and every situation, including power failures, errors, and crashes. To the outside world, a committed transaction appears (by its effects on the database) to be indivisible ('atomic')."

#### Consistency
"The consistency property ensures that any transaction will bring the database from one valid state to another. Any data written to the database must be valid according to all defined rules, including but not limited to constraints, cascades, triggers, and any combination thereof. This does not guarantee correctness of the transaction in all ways the application programmer might have wanted (that is the responsibility of application-level code) but merely that any programming errors do not violate any defined rules."

ArcadeDB uses MVCC (Multi-Version Concurrency Control) to assure consistency by versioning the page where the record is stored.

#### Isolation
"The isolation property ensures that the concurrent execution of transactions results in a system state that would be obtained if transactions were executed serially, i.e. one after the other. Providing isolation is the main goal of concurrency control. Depending on concurrency control method, the effects of an incomplete transaction might not even be visible to another transaction."

The SQL standard defines the following phenomena which are prohibited at various levels:
- **Dirty Read**: A transaction reads data written by a concurrent uncommitted transaction (never possible with ArcadeDB)
- **Non Repeatable Read**: A transaction re-reads data it has previously read and finds that data has been modified by another transaction (that committed since the initial read)
- **Phantom Read**: A transaction re-executes a query returning a set of rows that satisfy a search condition and finds that the set of rows satisfying the condition has changed

**SQL Standard Isolation Levels:**

| Isolation Level | Dirty Read | Non repeatable Read | Phantom Read |
|---|---|---|---|
| READ_COMMITTED (default) | Not possible | Possible | Possible |
| REPEATABLE_READ | Not possible | Not possible | Possible |

The SQL SERIALIZABLE level is not supported by ArcadeDB.

#### Durability
"Durability means that once a transaction has been committed, it will remain so, even in the event of power loss, crashes, or errors. In a relational database, for instance, once a group of SQL statements execute successfully, the results need to be stored permanently (even if there is a power loss right after). To defend against power loss, transactions (or their effects) must be recorded in a non-volatile memory."

### Fail-over

An ArcadeDB instance can fail for several reasons:
- Hardware problems, such as loss of power or disk error
- Software problems, such as an operating system crash
- Application problems, such as a bug that crashes your application

With embedded engine in the same process: If your application crashes (for any reason), the ArcadeDB engine also crashes.

With remote server connection: If your application crashes but the engine continues to work, any pending transaction owned by the client will be rolled back.

### Auto-recovery

At startup the ArcadeDB engine checks if it is restarting from a crash. In this case, the auto-recovery phase starts which rolls back all pending transactions.

ArcadeDB has different levels of durability based on storage type, configuration and settings.

### Optimistic Transaction

This mode uses the well-known Multi Version Control System MVCC by allowing multiple reads and writes on the same records. The integrity check is made on commit. If the record has been saved by another transaction in the interim, then a ConcurrentModificationException will be thrown. The application can choose either to repeat the transaction or to abort it.

**Key point:** ArcadeDB keeps the whole transaction in the host's RAM, so the transaction size is affected by the available RAM (Heap memory) on JVM. For transactions involving many records, consider splitting it into multiple transactions.

### Nested Transactions and Propagation

ArcadeDB does support nested transactions. If a `begin()` is called after a transaction is already begun, then the new transaction is the current one until commit or rollback. When this nested transaction is completed, the previous transaction becomes the current transaction again.

---

## Inheritance

Unlike many object-relational mapping tools, ArcadeDB does not split documents between different types. Each document resides in one or a number of buckets associated with its specific type. When you execute a query against a type that has subtypes, ArcadeDB searches the buckets of the target type and all subtypes.

### Declaring Inheritance in Schema

In developing your application, bear in mind that ArcadeDB needs to know the type inheritance relationship.

Example:
```java
DocumentType account = database.getSchema().createDocumentType("Account");
DocumentType company = database.getSchema().createDocumentType("Company");
account.addSuperType(account);
```

### Using Polymorphic Queries

By default, ArcadeDB treats all queries as polymorphic. Using the example above, you can run the following query:

```sql
SELECT FROM Account WHERE name.toUpperCase() = 'GOOGLE'
```

This query returns all instances of the types `Account` and `Company` that have a property name that matches Google.

### How Inheritance Works

**Hierarchy example:** Three types with bucket identifiers:
- ArcadeData (27)
- Company (13) — inherits from ArcadeData
- Account (10) — inherits from Company

By default, ArcadeDB creates a separate bucket for each type. The defaultBucketId property in a type indicates the bucket used by default when not specified. However, a type has a property bucketIds (as int[]), that contains all the buckets able to contain the records of that type. The properties bucketIds and defaultBucketId are the same by default.

When you execute a query against a type, ArcadeDB limits the result-sets to only the records of the buckets contained in the bucketIds property.

Example query:
```sql
SELECT FROM Account WHERE name.toUpperCase() = 'GOOGLE'
```

This query returns all records with the name property set to GOOGLE from all three types, given that the base type `Account` was specified. For the type `Account`, ArcadeDB searches inside the buckets 10, 13 and 27, following the inheritance specified in the schema.

---

## Schema

ArcadeDB supports schema-less, schema-full and hybrid operation. This means for all types that have a declared schema it is applied. Minimally, a type (document, vertex or edge) needs to be declared in a database to be able to insert to, i.e. CREATE TYPE (SQL) and ALTER TYPE (SQL).

Beyond the type, declaration properties with or without constraints, can be declared, i.e. CREATE PROPERTY (SQL) and ALTER PROPERTY (SQL). Inserting into a declared type, all datasares accepted (even with additional undeclared properties) as long as no property constraints are violated.

---

## Indexes

ArcadeDB supports multiple index algorithms, each optimized for different access patterns:

### Supported Index Types

1. **LSM Tree** (default)
   - Optimized for range scans, ordered iteration, and write-heavy workloads
   - Best for general-purpose use

2. **Hash Index**
   - O(1) equality lookups using extendible hashing
   - Best for primary key access, JOINs, and edge traversal where ordering is not needed
   - No compaction needed
   - Local splits: When a bucket is full, only that bucket splits
   - Supports unique and non-unique indexes
   - Null strategy: Same as LSM Tree (SKIP, ERROR, INDEX)

### LSM Tree Algorithm

LSM tree is a data structure used to store and retrieve data efficiently. It works by organizing data in a tree-like structure, where each node represents a certain range of data.

**How it works:**

1. When you want to store data in the LSM tree, it first goes into a special part of the tree called a "write buffer." The write buffer is like a temporary storage area where new data is kept.

2. When the write buffer gets full, the LSM tree will "flush" the data from the write buffer into the main part of the tree. This is done by creating a new node in the tree and adding the data from the write buffer to it.

3. As more data is added to the tree, it will eventually become too large to be stored in memory. When this happens, the LSM tree will start to "compact" the data by moving some of it to disk storage. This allows the tree to continue growing without running out of memory.

4. When you want to retrieve a piece of data from the LSM tree, the algorithm will search for it in the write buffer, the main part of the tree, and any data that has been compacted to disk storage. If the data is found, it will be returned.

### LSM Tree vs B+Tree

B+Tree is the most common algorithm used by relational DBMSs. What are the differences?

1. LSM tree and B+ tree are both data structures commonly used to store and retrieve data efficiently. LSM tree has several advantages over B+ tree:

2. **Write efficiency**: LSM tree uses a write buffer to temporarily store new data, which allows it to batch writes and reduce the number of disk accesses required. B+ tree, on the other hand, requires more complex rebalancing operations when compacting data.

3. **Compaction efficiency**: LSM tree stores data in a sorted fashion, it can compact data more efficiently by simply merging sorted data sets. B+ tree, on the other hand, requires more complex rebalancing operations when compacting data.

4. **Space efficiency**: LSM tree stores data in a compact, sorted format, which can make it more space-efficient than B+ tree. This can be especially useful when storing large amounts of data on disk.

5. **Disadvantages**: There are also some potential disadvantages of LSM tree compared to B+ tree. For example, B+ tree may be faster for queries that require range scans or random access, and it may be easier to implement in some cases.

If you're interested in ArcadeDB's LSM-Tree index implementation detail, look at LSM-Tree

### Hash Index Algorithm

ArcadeDB's Hash Index uses extendible hashing, a disk-oriented algorithm that provides O(1) equality lookups with typically 1-2 page reads.

**How it works:**

1. Each key is hashed to produce a binary hash code. The index maintains a global depth — the number of leading bits used from each hash.

2. A directory maps each possible bit prefix (2^globalDepth entries) to a bucket page. Multiple directory entries can point to the same bucket if the bucket's local depth is less than the global depth.

3. To look up a key, the index hashes the key, reads the directory entry for the corresponding bit prefix, and reads the target bucket page. The key is found via binary search within the bucket.

4. When a bucket overflows, it splits: the bucket's local depth increases by one, and entries are redistributed based on the additional bit. If the local depth exceeds the global depth, the directory doubles in size.

**Key properties:**
- No compaction needed: Unlike LSM Tree, there are no background merge operations and no write amplification
- Local splits: When a bucket is full, only that bucket splits — no global reorganization
- Supports unique and non-unique: Both UNIQUE_HASH and NOTUNIQU_HASH modes are available
- Null strategy: Supports the same NULL_STRATEGY options as LSM Tree (SKIP, ERROR, INDEX)

### When to Use Hash Index vs LSM Tree

| Use Case | Hash Index | LSM Tree |
|---|---|---|
| Point lookup (WHERE id = ?) | Best — O(1), 1-2 page reads | O(log N), multiple page reads |
| JOINs and edge traversal | Best — constant-time resolution | Good |
| Range scans (WHERE age > 30) | Not supported | Best — ordered iteration |
| ORDER BY on indexed field | Not supported | Best — natural ordering |
| Bulk insertion throughput | Consistent — no compaction | May degrade at scale due to compaction |

### Index Property Types

ArcadeDB indexes can index property fields in various ways:

1. All properties (scalar or collections) can be indexed in a unique or non-unique index (as a whole)
2. An index can be for a single property, or multiple properties in a compound index
3. String properties can be indexed tokenized in a full-text index
4. Object properties can be indexed by keys or by values
5. List properties can be indexed by values
6. Vectors of floats can be indexed using specialized vector indexes (HNSW or LSMVectorIndex)
7. WKT geometry strings can be indexed using a geospatial index

### LSMVectorIndex

LSMVectorIndex is a vector index built on ArcadeDB's LSM Tree architecture, providing efficient persistent storage and retrieval of vector embeddings. Key features:

- **Persistent Storage**: Vector indexes are stored on disk with automatic page management and compaction
- **Configurable Similarity Functions**: Supports COSINE (default), DOT_PRODUCT, and EUCLIDEAN distance metrics
- **Configurable ID Property**: The property used to identify vertices can be customized (default: "id")
- **SQL Query Support**: Query vectors using the vectorNeighbors() SQL function
- **Automatic Compaction**: Efficiently reclaims disk space through automatic compaction of immutable pages
- **High Performance**: LSM Tree benefits for write efficiency and space optimization at scale

---

## Full-Text Index

ArcadeDB's full-text index is built on Apache Lucene and supports configurable analyzers, advanced Lucene query syntax, multi-property indexing, relevance scoring, and "More Like This" similarity search.

### Creating a Full-Text Index

```sql
CREATE INDEX ON Article (content) FULL_TEXT
```

Multiple properties can be indexed together:
```sql
CREATE INDEX ON Article (title, body) FULL_TEXT
```

### Configuring the Analyzer

Pass a METADATA block to choose a Lucene analyzer:

```sql
CREATE INDEX ON Article (content) FULL_TEXT
  METADATA {
    "analyzer": "org.apache.lucene.analysis.en.EnglishAnalyzer",
    "allowLeadingWildcard": false,
    "defaultOperator": "OR"
  }
```

### Metadata Options

| Option | Type | Default | Description |
|---|---|---|---|
| analyzer | string | org.apache.lucene.analysis.standard.StandardAnalyzer | Lucene analyzer class used for both indexing and querying |
| index_analyzer | string | — | Override analyzer used only at index time |
| query_analyzer | string | — | Override analyzer used only at query time |
| allowLeadingWildcard | boolean | false | Allow *term wildcard queries |
| defaultOperator | "OR" \| "AND" | "OR" | Default operator between terms when none is specified |
| <field>_analyzer | string | — | Per-field analyzer override, e.g., "title_analyzer" |

### Common Analyzers

| Analyzer Class | Description |
|---|---|
| org.apache.lucene.analysis.standard.StandardAnalyzer | General-purpose tokenizer (default) |
| org.apache.lucene.analysis.en.EnglishAnalyzer | English stemming and stop words |
| org.apache.lucene.analysis.core.SimpleAnalyzer | Lowercase only, no stop words |
| org.apache.lucene.analysis.core.WhitespaceAnalyzer | Split on whitespace only |

### Per-Field Analyzers

For multi-property indexes, each field can use a different analyzer:

```sql
CREATE INDEX ON Article (title, body) FULL_TEXT
  METADATA {
    "analyzer": "org.apache.lucene.analysis.standard.StandardAnalyzer",
    "title_analyzer": "org.apache.lucene.analysis.en.EnglishAnalyzer"
  }
```

Here `title` uses the English analyzer while `body` falls back to the standard analyzer.

### Searching

**SEARCH_INDEX()** - Searches a named full-text index using Lucene query syntax:

```sql
SELECT * FROM Article
WHERE SEARCH_INDEX('Article[content]', 'java programming')
```

**SEARCH_FIELDS()** - Finds the full-text index automatically from field names, without needing to know the index name:

```sql
SELECT * FROM Article
WHERE SEARCH_FIELDS(['title', 'body'], 'database tutorial')
```

Signature: `SEARCH_FIELDS(fieldNames, query)`

| Parameter | Type | Description |
|---|---|---|
| fieldNames | array of strings | Fields to search; a full-text index covering these fields must exist |
| query | string | A Lucene query string |

### Lucene Query Syntax

Both SEARCH_INDEX and SEARCH_FIELDS accept standard Lucene query syntax.

**Boolean Operators:**

| Syntax | Meaning |
|---|---|
| java programming | Either term (OR, default) |
| +java +programming | Both terms required (AND) |
| java -python | java required, python excluded |
| java AND programming | Explicit AND |
| java OR python | Explicit OR |

**Phrase Queries:**

Wrap a phrase in double quotes to require terms to appear in order:

```sql
SELECT * FROM Article WHERE SEARCH_INDEX('Article[content]', '"machine learning"')
```

**Wildcard Queries:**

| Syntax | Matches |
|---|---|
| data* | database, datastore, dataset... |
| dat?base | database, datXbase... |
| *base | Requires allowLeadingWildcard: true |

**Fuzzy Queries:**

Append ~ to match terms within an edit distance:

```sql
SELECT * FROM Article WHERE SEARCH_INDEX('Article[content]', 'database~')
-- also matches terms within edit distance 2 (Lucene default), e.g., "databaSe", "databsee"
```

**Field-Qualified Queries (Multi-Property Indexes):**

For indexes over multiple fields, restrict a term to a specific field:

```sql
-- Only match "database" in the title field
SELECT * FROM Article WHERE SEARCH_INDEX('Article[title,body]', 'title:database')

-- Combine field-specific and general terms
SELECT * FROM Article WHERE SEARCH_INDEX('Article[title,body]', 'title:"multi model" -nosql')
```

### Relevance Score ($score)

Every match carries a relevance score. Use `$score` in projections or ORDER BY clauses:

```sql
SELECT title, $score
FROM Article
WHERE SEARCH_INDEX('Article[content]', 'java programming')
ORDER BY $score DESC
```

Documents that match more query terms receive higher scores.

### More Like This

"More Like This" finds documents similar to one or more source documents. It extracts representative terms from the sources, then searches for other documents sharing those terms.

**SEARCH_INDEX_MORE():**

```sql
SELECT title, $score, $similarity
FROM Article
WHERE SEARCH_INDEX_MORE('Article[content]', [#10:3])
ORDER BY $similarity DESC
```

Signature: `SEARCH_INDEX_MORE(indexName, sourceRIDs [, config])`

| Parameter | Type | Description |
|---|---|---|
| indexName | string | Full-text index name |
| sourceRIDs | array of RIDs | One or more source documents |
| config | JSON object | Optional MLT parameters (see More Like This Configuration) |

**SEARCH_FIELDS_MORE():**

Same as SEARCH_INDEX_MORE but resolves the full-text index automatically from field names:

```sql
SELECT title, $similarity
FROM Article
WHERE SEARCH_FIELDS_MORE(['content'], [#10:3])
ORDER BY $similarity DESC
```

Signature: `SEARCH_FIELDS_MORE(fieldNames, sourceRIDs [, config])`

| Parameter | Type | Description |
|---|---|---|
| fieldNames | array of strings | Fields to search; a full-text index covering these fields must exist |
| sourceRIDs | array of RIDs | One or more source documents |
| config | JSON object | Optional MLT parameters |

**Multiple Source Documents:**

Provide multiple RIDs to find documents similar to a combination of sources:

```sql
SELECT title, $similarity
FROM Article
WHERE SEARCH_INDEX_MORE('Article[content]', [#10:3, #10:4])
ORDER BY $similarity DESC
```

**Similarity Score ($similarity):**

`$similarity` is a normalized score from 0.0 to 1.0. The most similar document in the result set always receives 1.0.

### More Like This Configuration

Pass an optional JSON object to tune the algorithm:

```sql
SELECT title, $similarity
FROM Article
WHERE SEARCH_FIELDS_MORE(['title', 'content'], [#10:3], {
  "minTermFreq": 1,
  "minDocFreq": 3,
  "maxQueryTerms": 50,
  "excludeSource": false
})
ORDER BY $similarity DESC
LIMIT 5
```

| Option | Type | Default | Description |
|---|---|---|---|
| minTermFreq | int | 2 | Minimum times a term must appear in the source document(s) to be considered |
| minDocFreq | int | 5 | Minimum number of index documents that must contain a term for it to be used |
| maxDocFreqPercent | float | null | Exclude terms appearing in more than this fraction of all documents (e.g., 0.5 = 50%) |
| maxQueryTerms | int | 25 | Maximum number of terms to use in the similarity query |
| minWordLen | int | 0 | Ignore terms shorter than this length (0 = no minimum) |
| maxWordLen | int | 0 | Ignore terms longer than this length (0 = no maximum) |
| boostByScore | boolean | true | Weight terms by TF-IDF score rather than raw frequency |
| excludeSource | boolean | true | Exclude source documents from results |
| maxSourceDocs | int | 25 | Maximum number of source RIDs allowed |

### Practical Examples

**Blog Search with Ranking:**

```sql
SELECT title, author, $score AS relevance
FROM BlogPost
WHERE SEARCH_INDEX('BlogPost[title,body]', '+java +spring -legacy')
ORDER BY relevance DESC
LIMIT 20
```

**Autocomplete with Prefix Wildcard:**

```sql
SELECT title
FROM Product
WHERE SEARCH_INDEX('Product[name]', 'micro*')
```

**Stemming with English Analyzer:**

With EnglishAnalyzer, searching for "run" also matches "running", "runs", and "ran":

```sql
CREATE INDEX ON Article (content) FULL_TEXT
  METADATA { "analyzer": "org.apache.lucene.analysis.en.EnglishAnalyzer" }

SELECT * FROM Article WHERE SEARCH_INDEX('Article[content]', 'running')
-- also returns documents containing "run", "runs", "ran"
```

**Related Articles Recommendation:**

```sql
SELECT title, $similarity AS score
FROM Article
WHERE SEARCH_INDEX_MORE('Article[title,body]', [#10:42], {
  "minTermFreq": 1,
  "minDocFreq": 2,
  "maxQueryTerms": 30
})
ORDER BY score DESC
LIMIT 5
```

**Readers Who Liked This Also Liked:**

```sql
SELECT title, $similarity
FROM Article
WHERE SEARCH_INDEX_MORE('Article[content]', [#10:1, #10:7, #10:12])
ORDER BY $similarity DESC
LIMIT 10
```

### Notes

- Full-text indexes require all indexed properties to be of type STRING
- Full-text indexes cannot be marked UNIQUE
- Without a METADATA block, indexes use StandardAnalyzer with OR default operator and leading wildcards disabled
- Indexes created without metadata continue to work with SEARCH_INDEX and SEARCH_INDEX_MORE exactly as before
- `$score` and `$similarity` are always available as query variables for matching documents; non-matching documents receive 0

---

## Geospatial Index

ArcadeDB's geospatial index enables spatial queries directly from SQL using the `geo.*` function namespace. Geometries are stored as Well-Known Text (WKT) strings in regular STRING properties — no special column type is required.

The index is built on ArcadeDB's LSM-Tree storage engine, which means ACID guarantees, write-ahead logging, high availability replication, and automatic compaction all apply without any extra configuration. Internally, the index decomposes each geometry into GeoHash tokens using a RecursivePrefixTreeStrategy, stores each token in the underlying LSM-Tree, and applies an exact Spatial4j predicate as a post-filter.

### Creating a Geospatial Index

Create a STRING property, then add a GEOSPATIAL index to it:

```sql
CREATE VERTEX TYPE Location
CREATE PROPERTY Location.coords STRING
CREATE INDEX ON Location (coords) GEOSPATIAL
```

The index type keyword is GEOSPATIAL. Only STRING properties are supported (WKT values are stored as strings).

### Configuring Precision

The precision level controls the GeoHash grid resolution used to decompose geometries. Higher precision means more tokens per geometry, finer resolution, and a larger index.

```sql
CREATE INDEX ON Location (coords) GEOSPATIAL
  METADATA { "precision": 9 }
```

| Precision | Cell size (approx.) | Typical use |
|---|---|---|
| 5 | ~4.9 km × 4.9 km | Country / region level |
| 7 | ~153 m × 153 m | City / neighborhood level |
| 9 | ~4.8 m × 4.8 m | Street-address precision |
| 11 (default) | ~2.4 m × 2.4 m | High precision — building footprints, GPS tracks |

**Important:** Changing precision after data has been indexed requires rebuilding the index with REBUILD INDEX.

### Inserting Geometries

Store any valid WKT string in the indexed property. Pass it as a plain string literal or use the `geo.asText()` function to convert a geometry object back to WKT:

**Insert a point by WKT literal:**

```sql
INSERT INTO Location SET name = 'Eiffel Tower', coords = 'POINT(2.2945 48.8584)'
```

**Insert a polygon by WKT literal:**

```sql
INSERT INTO Location SET name = 'Paris Centre', coords = 'POLYGON((2.2945 48.8584, 2.3522 48.8566, 2.3488 48.8791, 2.2945 48.8584))'
```

**Use geo.asText() to store a constructed geometry:**

```sql
INSERT INTO Location SET name = 'Arc de Triomphe',
  coords = geo.asText(geo.point(2.2950, 48.8738))
```

Records with null or unparsable WKT values in the indexed property are silently skipped during indexing.

### Querying with Spatial Predicates

Use any `geo.*` predicate in a WHERE clause. When a geospatial index exists on the referenced field, the query planner automatically uses it — no hint or special syntax is required.

**All locations within a bounding polygon:**

```sql
SELECT name FROM Location
WHERE geo.within(coords, geo.geomFromText(
  'POLYGON((2.28 48.85, 2.40 48.85, 2.40 48.90, 2.28 48.90, 2.28 48.85))'
)) = true
```

**All locations that intersect a line:**

```sql
SELECT name FROM Location
WHERE geo.intersects(coords, geo.geomFromText(
  'LINESTRING(2.29 48.85, 2.35 48.87)'
)) = true
```

**All locations within 500 m of a point (full-scan fallback — see note):**

```sql
SELECT name FROM Location
WHERE geo.dWithin(coords, geo.geomFromText('POINT(2.2945 48.8584)'), 0.0045) = true
```

### How Index-Accelerated Queries Work

1. The SQL query planner detects a spatial predicate function implementing IndexableSQLFunction in the WHERE clause
2. It calls allowsIndexedExecution(), which returns true when the first argument is a bare field reference and a GEOSPATIAL index exists on that field
3. The index decomposes the query shape into GeoHash cells and looks up candidate record IDs in the LSM-Tree
4. The predicate function's exact Spatial4j implementation post-filters the candidates to remove false positives (GeoHash cells are rectangular approximations)

The result is correct — the index is a superset filter, and the exact predicate is always applied afterward.

### geo.* Functions Reference

Geospatial functions are organized into two groups: **constructor / accessor functions** (pure computation, no index) and **spatial predicate functions** (used in WHERE clauses, index-accelerated where possible).

All functions accept either WKT strings or geometry objects produced by other geo.* functions. All predicate functions return null when either argument is null (SQL three-valued logic).

**Constructor and Accessor Functions:**

- `geo.geomFromText(<wkt-string>)` - Parses any WKT string into a geometry object
- `geo.point(<x>, <y>)` - Creates a 2D point from longitude and latitude (or X and Y)
- `geo.lineString([<point>*])` - Creates a line string from a list of points
- `geo.polygon([<point>*])` - Creates a polygon from an ordered list of points (first and last must be identical)
- `geo.buffer(<geometry>, <distance-in-degrees>)` - Returns a new geometry that is a buffer of the given distance around the input geometry
- `geo.envelope(<geometry>)` - Returns the minimum bounding rectangle (envelope) of a geometry as a polygon WKT
- `geo.distance(<geometry1>, <geometry2> [, <unit>])` - Returns distance between two points using the Haversine formula (great-circle distance). Optional unit parameter accepts 'km' (default) or 'm'
- `geo.area(<geometry>)` - Returns the area of a geometry in square degrees
- `geo.asText(<geometry>)` - Converts a geometry object to its WKT string representation
- `geo.asGeoJson(<geometry>)` - Converts a geometry object to a GeoJSON string

**Spatial Predicate Functions:**

All predicate functions accept a field reference or WKT string as the first argument and a geometry (from any geo.* constructor) as the second. When a GEOSPATIAL index exists on the referenced field, the query planner uses it automatically.

| Function | Semantics | Index used? |
|---|---|---|
| geo.within(g, shape) | g is fully contained within shape | Yes |
| geo.intersects(g, shape) | g and shape share at least one point | Yes |
| geo.contains(g, shape) | g fully contains shape | Yes |
| geo.equals(g, shape) | g and shape are geometrically identical | Yes |
| geo.crosses(g, shape) | g crosses the boundary of shape | Yes |
| geo.overlaps(g, shape) | g and shape overlap (share interior but neither contains the other) | Yes |
| geo.touches(g, shape) | g touches the boundary of shape but interiors do not intersect | Yes |
| geo.disjoint(g, shape) | g and shape share no points | No — full scan |
| geo.dWithin(g, shape, dist) | g is within dist degrees of shape | No — full scan |

**Important notes:**
- `geo.disjoint` cannot use the index because the index stores records that intersect indexed cells; records absent from the index are exactly those that are disjoint.
- `geo.dWithin` falls back to a full scan because correct proximity indexing requires expanding the query shape into a bounding circle first — planned as a future enhancement.

### Practical Examples

**Points of Interest Near a Landmark:**

```sql
SELECT name, coords
FROM PointOfInterest
WHERE geo.within(coords, geo.buffer(geo.point(2.2945, 48.8584), 0.0045)) = true
ORDER BY geo.distance(coords, geo.geomFromText('POINT(2.2945 48.8584)'))
LIMIT 20
```

**Delivery Zones Containing an Address:**

```sql
SELECT zone_name, carrier
FROM DeliveryZone
WHERE geo.contains(boundary, geo.geomFromText('POINT(2.3522 48.8566)')) = true
```

**Fleet Vehicles Inside a Geofence:**

```sql
SELECT vehicle_id, last_seen
FROM Vehicle
WHERE geo.within(last_position,
  geo.geomFromText('POLYGON((2.28 48.84, 2.42 48.84, 2.42 48.90, 2.28 48.90, 2.28 48.84))')
) = true
AND last_seen > date('2026-01-01', 'yyyy-MM-dd')
```

**Building Envelopes Intersecting a Road Segment:**

```sql
SELECT building_id, address
FROM Building
WHERE geo.intersects(footprint, geo.geomFromText(
  'LINESTRING(2.2900 48.8600, 2.3100 48.8700)'
)) = true
```

**Combining Geospatial and Graph Queries:**

Graph traversal and spatial predicates compose naturally in the same query:

```sql
SELECT expand(outV())
FROM Has
WHERE geo.within(outV().location, geo.geomFromText(
  'POLYGON((2.28 48.84, 2.42 48.84, 2.42 48.90, 2.28 48.90, 2.28 48.84))'
)) = true
```

### Notes and Limitations

- Geometries must be stored as WKT strings in STRING properties. No dedicated geometry column type is introduced; WKT is the only supported format at storage level
- GeoJSON is not a storage format, but `geo.asGeoJson()` converts any geometry to GeoJSON for output
- 3D geometry and raster data are not supported
- The antimeridian and polar edge cases are handled correctly by the underlying GeoHash grid
- Changing the precision level of an existing geospatial index requires rebuilding the index with REBUILD INDEX
- A GEOSPATIAL index cannot be marked UNIQUE

---

## Materialized Views

A materialized view is a schema-level object that stores the result of a SQL SELECT query as a backing document type. Unlike regular views (which re-execute the query on every access), a materialized view holds a pre-computed snapshot of data that can be queried directly for fast reads.

**A materialized view:**
- Wraps a SQL SELECT query as its defining query
- Stores results in a backing DocumentType with standard buckets
- Supports three refresh modes: manual, incremental (post-commit), and periodic (scheduled)
- Persists its definition and metadata in `schema.json` alongside other schema objects
- Can be created, dropped, refreshed, and altered via SQL DDL statements or the Java Schema API

### Refresh Modes

#### MANUAL
The view data is never automatically updated. You must trigger a refresh explicitly via REFRESH MATERIALIZED VIEW or the Java API. Use this when you control refresh timing yourself or when source data changes infrequently.

#### INCREMENTAL
After every committed transaction that modifies a source type, ArcadeDB automatically refreshes the view in a post-commit callback. The refresh is:
- **Transactionally safe**: runs after the source transaction commits successfully; rolled-back transactions do not trigger a refresh
- **Batched per transaction**: multiple record changes in a single transaction result in one refresh, not one per record
- **For simple queries** (single source type, no aggregates, no GROUP BY, no subqueries, no JOINs): performs a full refresh
- **For complex queries** (aggregates, GROUP BY, etc.): also performs a full refresh (per-record incremental optimization is planned for a future release)

If the refresh fails, the view is marked STALE and a WARNING is logged. A manual refresh can recover it.

#### PERIODIC
A background scheduler thread runs a full refresh at the specified interval after each successful refresh completes. Intervals are specified in seconds, minutes, or hours:

```sql
REFRESH EVERY 30 SECOND
REFRESH EVERY 5 MINUTE
REFRESH EVERY 1 HOUR
```

The scheduler uses a single daemon thread (ArcadeDB-MV-Scheduler) shared across all periodic views. If the database is closed, all scheduled tasks are cancelled automatically.

### View Status

Each view tracks a `status` field that reflects its current state:

| Status | Meaning |
|---|---|
| VALID | Data is up to date with the last refresh |
| STALE | A refresh failed or was interrupted; data may be outdated |
| BUILDING | A refresh is currently in progress |
| ERROR | The last refresh encountered a fatal error |

If the database crashes while a view is BUILDING, the status is reset to STALE on the next startup to signal that the data may be incomplete.

### Querying a Materialized View

Query a materialized view exactly like any other document type:

```sql
SELECT * FROM ActiveUsers
SELECT name FROM ActiveUsers WHERE name LIKE 'A%'
SELECT count(*) FROM RecentOrders
```

### Java API

**Creating a view:**

```java
database.transaction(() -> {
  database.getSchema().buildMaterializedView()
    .withName("ActiveUsers")
    .withQuery("SELECT name, email FROM User WHERE active = true")
    .withRefreshMode(MaterializedViewRefreshMode.MANUAL)
    .create();
});
```

**Builder options:**

| Method | Description |
|---|---|
| withName(String) | Name for the view (required) |
| withQuery(String) | Defining SQL SELECT query (required) |
| withRefreshMode(MaterializedViewRefreshMode) | MANUAL, INCREMENTAL, or PERIODIC (default: MANUAL) |
| withTotalBuckets(int) | Number of buckets for the backing type |
| withPageSize(int) | Page size for the backing type |
| withRefreshInterval(long) | Interval in milliseconds for PERIODIC mode |
| withIgnoreIfExists(boolean) | When true, returns existing view instead of throwing |

**Querying schema:**

```java
Schema schema = database.getSchema();

// Check existence
boolean exists = schema.existsMaterializedView("ActiveUsers");

// Get a specific view
MaterializedView view = schema.getMaterializedView("ActiveUsers");

// List all views
MaterializedView[] views = schema.getMaterializedViews();
```

**Refreshing and dropping:**

```java
// Programmatic refresh
database.getSchema().getMaterializedView("ActiveUsers").refresh();

// Drop via schema
database.getSchema().dropMaterializedView("ActiveUsers");

// Drop via the view itself
database.getSchema().getMaterializedView("ActiveUsers").drop();
```

**Inspecting a view:**

```java
MaterializedView view = database.getSchema().getMaterializedView("HourlySummary");

view.getName();                // "HourlySummary"
view.getQuery();               // the defining SQL query
view.getRefreshMode();         // MaterializedViewRefreshMode.PERIODIC
view.getStatus();              // MaterializedViewStatus.VALID
view.getLastRefreshTime();     // epoch millis of last successful refresh
view.isSimpleQuery();          // true if eligible for per-record incremental optimization
view.getSourceTypeNames();     // list of source type names parsed from the query
view.getBackingType();         // the underlying DocumentType
```

### Behavior and Constraints

- **Backing type protection**: You cannot DROP TYPE on a type that backs a materialized view. Drop the materialized view first.
- **Name uniqueness**: The view name must not match any existing type or materialized view.
- **Source type validation**: All types referenced in the FROM clause must exist when the view is created.
- **Persistence**: View definitions and metadata are stored in `schema.json` under a "materializedViews" key and survive database restarts. Listener registration for INCREMENTAL views and scheduler tasks for PERIODIC views are re-established on startup.
- **Transaction safety**: The initial full refresh and all subsequent refreshes run inside their own transactions.
- **Query result columns**: Only non-internal properties (those not starting with @) are copied into the backing type during refresh.
- **No schema on backing type**: The backing document type is schema-less; property types are not enforced.

### Error Handling

- If a post-commit refresh (INCREMENTAL mode) fails, the view is marked STALE and a WARNING is logged. The source transaction is unaffected.
- If a periodic refresh (PERIODIC mode) fails, the view is marked ERROR and a SEVERE log entry is written. The scheduler continues running and will retry on the next interval.
- Callback errors in the transaction callback system are logged at WARNING level and do not affect the triggering transaction or other callbacks.

### Limitations

- ALTER MATERIALIZED VIEW is not yet implemented.
- Per-record incremental refresh (tracking _sourceRID to update individual view rows) is a planned future optimization. Currently, all refresh operations perform a full truncate-and-reload.
- No support for cross-database queries in the defining query.
- Server replication: materialized view data lives in the local backing type and is replicated like any other document type in an HA cluster, but refresh triggering is local to the node that executes the write.

### Example: Sales Dashboard

**Source type:**

```sql
CREATE DOCUMENT TYPE Sale;
```

**Periodic summary refreshed every minute:**

```sql
CREATE MATERIALIZED VIEW SalesByProduct
  AS SELECT product, sum(amount) AS total, count(*) AS count
     FROM Sale
     GROUP BY product
  REFRESH EVERY 1 MINUTE;
```

**Incremental view of recent activity (simple query):**

```sql
CREATE MATERIALIZED VIEW RecentSales
  AS SELECT product, amount, date
     FROM Sale
     WHERE date >= '2026-01-01'
  REFRESH INCREMENTAL;
```

**Query the views:**

```sql
SELECT * FROM SalesByProduct ORDER BY total DESC;
SELECT product, amount FROM RecentSales WHERE amount > 1000;
```

**Manual refresh after a bulk import:**

```sql
REFRESH MATERIALIZED VIEW SalesByProduct;
```

**Teardown:**

```sql
DROP MATERIALIZED VIEW SalesByProduct;
DROP MATERIALIZED VIEW RecentSales;
```

---

## Graph Database

ArcadeDB is a native graph database; this section explains what this means and how it relates to applications.

### Graph Components

Essentially a graph is a tuple, or pair of sets, of vertices (aka nodes) and edges (aka arcs), whereas the set of vertices contains an (indexed) set of objects, and the set of edges contains (at least) pairs specifying a respective edge's endpoints.

A particular type of graph is the directed graph, which is characterized by oriented edges, meaning each edge's pair is ordered. A simple example of a directed graph is given by:

**Vertices:** V = {v_1, v_2}
**Edges:** E = {e_(1,2)}

With:
- v_1 → e_(1,2) → v_2

### Graph Database Types

There are two prevalent data models for graph databases (both usually directed):

1. **Triple store (RDF graph)**: Data is represented as subject-predicate-object triples; all facts are uniform triples and semantics are standardized (RDF, RDFS/OWL, SPARQL).

2. **Property Graph**: Vertices and edges carry labels (types) and arbitrary key/value properties directly attached to them.

ArcadeDB is a **property graph database**: vertices and edges have labels and can store arbitrary key/value properties, supporting compact storage, fast traversals, and flexible schema evolution within its multi-model, document-oriented engine.

Practically, a vertex or edge consists of an identifier (RID), a label (type), and properties (document), plus ordered endpoint references. The latter are vertex properties of incoming and outgoing edges, instead of edge properties of ordered vertex pairs, for technical reasons.

### Why (Property) Graph Databases?

- The modeled domain is already a network.
- Fast traversal of relations instead of costly joins in relational databases.
- Naturally annotated edges instead of inconvenient reification in RDF graphs.

---

## Time Series

ArcadeDB includes a native Time Series engine designed for high-throughput ingestion and fast analytical queries over timestamped data. Unlike bolt-on solutions, the Time Series model is integrated directly into the multi-model core — the same database that stores graphs, documents, and key/value pairs can store and query billions of time-stamped samples with specialized columnar compression, SIMD-vectorized aggregation, and automatic lifecycle management.

### Key Capabilities

- **Columnar storage** with Gorilla (float), Delta-of-Delta (timestamp), Simple-8b (integer), and Dictionary (tag) compression — as low as 0.4 to 1.4 bytes per sample
- **Shard-per-core parallelism** with lock-free writes
- **Block-level aggregation statistics** for zero-decompression fast-path queries
- **InfluxDB Line Protocol** ingestion for compatibility with Telegraf, Grafana Agent, and hundreds of collection agents
- **Prometheus remote_write / remote_read** protocol for drop-in Prometheus backend usage
- **PromQL query language** with native parser and HTTP-compatible API endpoints
- **SQL analytical functions** — ts.timeBucket, ts.rate, ts.percentile, ts.interpolate, window functions, and more
- **Continuous aggregates** with watermark-based incremental refresh
- **Retention policies and downsampling tiers** for automatic data lifecycle
- **Grafana integration** via DataFrame-compatible endpoints (works with the Infinity datasource plugin)
- **Studio TimeSeries Explorer** with query, schema inspection, ingestion docs, and PromQL tabs

### Creating a TimeSeries Type

Use `CREATE TIMESERIES TYPE` to define a new time series type. Every type requires a TIMESTAMP column, zero or more TAGS (low-cardinality indexed dimensions), and one or more FIELDS (high-cardinality measurement values).

```sql
CREATE TIMESERIES TYPE SensorReading
  TIMESTAMP ts PRECISION NANOSECOND
  TAGS (sensor_id STRING, location STRING)
  FIELDS (
    temperature DOUBLE,
    humidity DOUBLE,
    pressure DOUBLE
  )
  SHARDS 8
  RETENTION 90 DAYS
  COMPACTION_INTERVAL 1 HOURS
  BLOCK_SIZE 65536
```

**Minimal syntax** (defaults for everything optional):

```sql
CREATE TIMESERIES TYPE SensorReading
  TIMESTAMP ts
  TAGS (sensor_id STRING)
  FIELDS (temperature DOUBLE)
```

### CREATE TIMESERIES TYPE Options

| Option | Default | Description |
|---|---|---|
| TIMESTAMP <name> | (required) | Name of the timestamp column |
| PRECISION | NANOSECOND | Timestamp resolution: SECOND, MILLISECOND, MICROSECOND, NANOSECOND |
| TAGS (...) | (none) | Comma-separated name TYPE pairs for low-cardinality dimensions |
| FIELDS (...) | (required) | Comma-separated name TYPE pairs for measurement values |
| SHARDS | CPU count | Number of shards for parallel writes |
| RETENTION | (none) | Automatic deletion of data older than the specified duration (e.g., 90 DAYS, 1 HOURS) |
| COMPACTION_INTERVAL | (none) | Splits sealed blocks at time-bucket boundaries for fast-path aggregation |
| BLOCK_SIZE | 65536 | Samples per sealed block |
| IF NOT EXISTS | (none) | Silently skip creation if the type already exists |

### Altering a TimeSeries Type

Add downsampling policies to automatically reduce resolution of old data:

```sql
ALTER TIMESERIES TYPE SensorReading
  ADD DOWNSAMPLING POLICY
    AFTER 7 DAYS GRANULARITY 1 MINUTES
    AFTER 30 DAYS GRANULARITY 1 HOURS
```

Remove all downsampling policies:

```sql
ALTER TIMESERIES TYPE SensorReading DROP DOWNSAMPLING POLICY
```

### Dropping a TimeSeries Type

```sql
DROP TIMESERIES TYPE SensorReading
DROP TIMESERIES TYPE IF EXISTS SensorReading
```

### Ingesting Data

There are four ways to ingest data into a TimeSeries type, listed from fastest to most convenient.

#### InfluxDB Line Protocol (Recommended for High Throughput)

The fastest remote ingestion path. ArcadeDB exposes an InfluxDB Line Protocol-compatible HTTP endpoint that skips SQL parsing entirely.

```
POST /api/v1/ts/{database}/write?precision=ns

Content-Type: text/plain

SensorReading,sensor_id=sensor-A,location=building-1 temperature=22.5,humidity=65.0,pressure=1013.25 1708430400000000000
SensorReading,sensor_id=sensor-B,location=building-2 temperature=19.1,humidity=70.0 1708430401000000000
```

**Line Protocol format:** `<measurement>[,<tag>=<value>,...] <field>=<value>[,...] [<timestamp>]`

**Precision parameter:** ns (nanoseconds, default), us (microseconds), ms (milliseconds), s (seconds).

**Example with curl:**

```bash
curl -X POST "http://localhost:2480/api/v1/ts/mydb/write?precision=ns" \
  -u root:password \
  -H "Content-Type: text/plain" \
  --data-binary 'SensorReading,sensor_id=sensor-A temperature=22.5 1708430400000000000'
```

**Example with Python:**

```python
import requests

lines = [
  "SensorReading,sensor_id=sensor-A temperature=22.5 1708430400000000000",
  "SensorReading,sensor_id=sensor-A temperature=22.6 1708430401000000000",
]

requests.post(
  "http://localhost:2480/api/v1/ts/mydb/write?precision=ns",
  auth=("root", "password"),
  headers={"Content-Type": "text/plain"},
  data="\n".join(lines),
)
```

If the type does not exist and auto-creation is enabled (arcadedb.tsAutoCreateType=true), the schema is inferred from the first line: measurement name becomes the type, tags become TAG columns, and fields become FIELD columns with inferred types.

#### Prometheus Remote Write

ArcadeDB acts as a drop-in Prometheus remote storage backend. Configure Prometheus to write to ArcadeDB:

```yaml
# prometheus.yml
remote_write:
  - url: "http://localhost:2480/ts/mydb/prom/write"
    basic_auth:
      username: root
      password: password
```

The endpoint accepts the standard Prometheus remote_write Protobuf payload (snappy-compressed). Each time series is mapped to an ArcadeDB TimeSeries type named after the __name__ label. Types are auto-created if they do not exist.

#### SQL INSERT

Standard ArcadeDB SQL syntax works for TimeSeries types:

**Single row:**

```sql
INSERT INTO SensorReading
  SET ts = '2026-02-20T10:00:00.000Z',
      sensor_id = 'sensor-A',
      location = 'building-1',
      temperature = 22.5,
      humidity = 65.0
```

**Batch insert:**

```sql
INSERT INTO SensorReading
  (ts, sensor_id, location, temperature, humidity)
  VALUES
    ('2026-02-20T10:00:00Z', 'sensor-A', 'building-1', 22.5, 65.0),
    ('2026-02-20T10:00:01Z', 'sensor-A', 'building-1', 22.6, 64.8),
    ('2026-02-20T10:00:02Z', 'sensor-B', 'building-2', 19.1, 70.0)
```

**CONTENT syntax:**

```sql
INSERT INTO SensorReading
  CONTENT { "ts": "2026-02-20T10:00:00.000Z", "sensor_id": "sensor-A", "temperature": 22.5 }
```

#### Java Embedded API

The fastest path — bypasses all protocol and SQL overhead:

```java
TimeSeriesEngine engine = database.getSchema()
  .getTimeSeriesType("SensorReading").getEngine();

long[] timestamps = { 1708430400000000000L, 1708430401000000000L };
String[] sensorIds = { "sensor-A", "sensor-A" };
double[] temperatures = { 22.5, 22.6 };

database.transaction(() -> {
  engine.appendSamples(timestamps,
    new Object[] { sensorIds, temperatures });
});
```

### Ingestion Method Comparison

| Method | Throughput | Overhead | Best For |
|---|---|---|---|
| Java Embedded API | ~0.5-1 us/sample | None (direct) | Embedded applications |
| InfluxDB Line Protocol | ~1-5 us/sample | Text parsing | Remote ingestion, Telegraf |
| Prometheus Remote Write | ~2-10 us/sample | Protobuf + Snappy | Prometheus ecosystems |
| SQL INSERT | ~50-100 us/sample | SQL parsing + planning | Ad-hoc inserts, small batches |

### Querying Time Series Data

#### SQL Queries

Time series types support standard SQL SELECT with WHERE, GROUP BY, and ORDER BY. Time range conditions (BETWEEN, >, >=, <, <=, =) on the timestamp column are pushed down to the storage engine for efficient range scans.

**Basic range query:**

```sql
SELECT ts, sensor_id, temperature, humidity
FROM SensorReading
WHERE ts BETWEEN '2026-02-19' AND '2026-02-20'
  AND sensor_id = 'sensor-A'
ORDER BY ts
```

**Aggregation with time bucketing:**

```sql
SELECT ts.timeBucket('1h', ts) AS hour,
       sensor_id,
       avg(temperature) AS avg_temp,
       max(temperature) AS max_temp,
       min(temperature) AS min_temp,
       count(*) AS sample_count
FROM SensorReading
WHERE ts BETWEEN '2026-02-19' AND '2026-02-20'
GROUP BY hour, sensor_id
ORDER BY hour
```

### TimeSeries SQL Functions

ArcadeDB provides a comprehensive set of `ts.*` SQL functions for time series analytics.

| Function | Description |
|---|---|
| ts.timeBucket(interval, timestamp) | Truncates a timestamp to the nearest interval boundary for GROUP BY aggregation. Intervals: '1s', '5m', '1h', '1d', etc. |
| ts.first(value, timestamp) | Returns the value corresponding to the earliest timestamp in the group |
| ts.last(value, timestamp) | Returns the value corresponding to the latest timestamp in the group |
| ts.rate(value, timestamp [, counterResetDetection]) | Per-second rate of change. Optional 3rd parameter (true) enables Prometheus-style counter reset detection for monotonic counters |
| ts.delta(value, timestamp) | Difference between the last and first values in the group |
| ts.movingAvg(value, window) | Moving average with a configurable window size |
| ts.interpolate(value, method [, timestamp]) | Gap filling. Methods: 'zero', 'prev', 'linear', 'none'. The 'linear' method requires the timestamp parameter |
| ts.correlate(a, b) | Pearson correlation coefficient between two series |
| ts.percentile(value, percentile) | Approximate percentile calculation (0.0-1.0). E.g., ts.percentile(latency, 0.99) for p99 |
| ts.lag(value, offset, timestamp [, default]) | Window function: returns the value from a previous row |
| ts.lead(value, offset, timestamp [, default]) | Window function: returns the value from a subsequent row |
| ts.rowNumber(timestamp) | Window function: sequential 1-based row numbering |
| ts.rank(value, timestamp) | Window function: rank with ties, gaps after ties |

**Examples:**

```sql
-- Rate of change with counter reset detection
SELECT ts.timeBucket('5m', ts) AS window,
       ts.rate(request_count, ts, true) AS requests_per_sec
FROM HttpMetrics
WHERE ts > '2026-02-20T10:00:00Z'
GROUP BY window
```

### PromQL Query Language

ArcadeDB's Time Series engine supports PromQL, the Prometheus query language, via native parser and evaluator. Query using the HTTP-compatible API:

```
GET /api/v1/query?query=rate(http_requests_total[5m])&time=1708430400
GET /api/v1/query_range?query=temperature&start=1708344000&end=1708430400&step=300
```

### Continuous Aggregates with Watermarks

Materialized incremental refreshes of time-bucketed results using watermark-based logic.

---

## Creating Databases

To create a database, use the HTTP API or embedded Java API.

**HTTP API:**

```
POST /api/v1/databases/{database-name}
```

**Java API:**

```java
Database db = new DatabaseFactory()
  .open("mydb");
```

---

## Summary

This comprehensive reference covers the core concepts essential to using ArcadeDB effectively:

- **Records** are the fundamental storage unit (Documents, Vertices, Edges)
- **Types** provide logical organization with inheritance support
- **Buckets** provide physical storage with partitioning strategy options
- **Relationships** can be expressed as graph edges, references, or embedded documents
- **Transactions** provide ACID guarantees with optimistic MVCC
- **Inheritance** enables polymorphic queries and schema hierarchies
- **Schemas** can be schema-full, schema-less, or hybrid
- **Indexes** support LSM-Tree, Hash, Full-Text, Geospatial, and Vector algorithms
- **Materialized Views** provide pre-computed snapshots with multiple refresh modes
- **Graphs** are natively supported with O(1) traversal complexity
- **Time Series** is integrated for high-throughput analytics on timestamped data

All these concepts work together within ArcadeDB's unified, multi-model engine.
