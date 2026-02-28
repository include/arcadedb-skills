# ArcadeDB API and Driver Reference Guide

## Table of Contents
1. [HTTP/JSON API](#httpjson-api)
2. [Java API (Local)](#java-api-local)
3. [Java API (Remote)](#java-api-remote)
4. [Java API (gRPC)](#java-api-grpc)
5. [Python Connectivity](#python-connectivity)
6. [NodeJS/JavaScript Connectivity](#nodejsjavascript-connectivity)
7. [PostgreSQL Protocol Plugin](#postgresql-protocol-plugin)
8. [Neo4j BOLT Protocol Plugin](#neo4j-bolt-protocol-plugin)
9. [JDBC Driver](#jdbc-driver)
10. [Redis Plugin](#redis-plugin)

---

## HTTP/JSON API

### Overview
ArcadeDB provides an HTTP/JSON API to interface with the server from any remote application. Uses standard HTTP calls with JSON payloads.

### Base Endpoints

#### Query Execution
- **GET/POST** `/query/graph/<database>`
- Executes SELECT and MATCH queries
- Supports multiple query languages via the `language` parameter

#### Command Execution
- **POST** `/command/<database>`
- Executes commands that can modify the database
- Supports INSERT, UPDATE, DELETE, CREATE statements

#### Server Management
- **GET** `/server` - Get server information
- **POST** `/server/database/<database>/create` - Create database
- **POST** `/server/database/<database>/drop` - Drop database
- **GET** `/server/ready` - Health check

### Query Examples

#### HTTP GET Query
```bash
curl "http://localhost:2480/query/graph/MyDatabase?language=sql&command=SELECT%20FROM%20User"
```

#### HTTP POST Query
```bash
curl -X POST "http://localhost:2480/query/graph/MyDatabase" \
  -H "Content-Type: application/json" \
  -d '{"language": "sql", "command": "SELECT FROM User WHERE age > 30"}'
```

#### Mongo Query via HTTP
```bash
curl "http://localhost:2480/query/graph/mongo/{collection:'Heroes', query: {$and: [{name: {$eq: 'Jay'}}, {lastName: {$exists: true}}], {lastName: {$eq: 'Miner'}}}}"
```

### Response Format
Responses are returned as JSON with result sets:
```json
{
  "result": [
    {
      "RID": "#1:0",
      "@type": "d",
      "name": "John",
      "age": 30
    }
  ]
}
```

### Query Languages Supported
- **SQL** - Standard query language
- **Mongo** - MongoDB query syntax via Mongo protocol plugin
- **Redis** - Redis commands via Redis plugin
- **Gremlin** - Graph traversal language

### Request/Response Headers
- `Content-Type: application/json`
- `Authorization: Basic` (if authentication enabled)
- User and password via query parameters or HTTP Basic Auth

### Parameters

#### For Queries
- `language` - Query language (sql, mongo, gremlin, redis)
- `command` - The query/command to execute
- `limit` - Result limit
- `skip` - Result offset
- `timeout` - Query timeout in milliseconds

#### For Commands
- `language` - Command language
- `command` - Command to execute
- Positional and named parameters supported

---

## Java API (Local)

### Setup

#### Maven Dependency
```xml
<dependency>
  <groupId>com.arcadedb</groupId>
  <artifactId>arcadedb-engine</artifactId>
  <version>26.2.1</version>
</dependency>
```

### Key Classes

#### DatabaseFactory
Entry point for creating and opening databases locally.

**Methods:**
- `create()` - Creates new database
- `open()` - Opens existing database
- `open(MODE mode)` - Opens with specific mode (READ_WRITE, READ_ONLY)
- `exists()` - Checks if database exists
- `close()` - Closes factory

**Example:**
```java
DatabaseFactory factory = new DatabaseFactory("/databases/mydb");
Database db = factory.open();
try {
  // Use database
} finally {
  db.close();
}
```

#### Database Interface
Main class to operate with ArcadeDB. Provides synchronous and asynchronous APIs.

**Key Methods:**

**Lifecycle:**
- `begin()` - Start transaction
- `commit()` - Commit transaction
- `rollback()` - Rollback transaction
- `close()` - Close database
- `drop()` - Drop database completely
- `isOpen()` - Check if database open
- `async()` - Get async executor

**Schema:**
- `getSchema()` - Get Schema instance
- `getSchema().createDocumentType(String typeName)`
- `getSchema().createVertexType(String typeName)`
- `getSchema().createEdgeType(String typeName)`

**Queries:**
- `query(String language, String command, Object... positionalParameters)` - Execute query with positional params
- `query(String language, String command, Map<String,Object> parameterMap)` - Execute query with named params
- `command(String language, String command, Object... positionalParameters)` - Execute command (can modify DB)
- `command(String language, String command, Map<String,Object> parameterMap)` - Execute command with named params

**Records:**
- `newDocument(String typeName)` - Create new document
- `newVertex(String typeName)` - Create new vertex
- `newEdgeByKeys(...)` - Create edge by vertex keys
- `lookupByKey(String type, String[] properties, Object[] keys)` - Lookup by indexed key
- `lookupByRID(RID rid, boolean loadContent)` - Lookup by RID
- `deleteRecord(Record record)` - Delete record

**Iteration:**
- `iterateBucket(String bucketName)` - Iterate bucket records
- `iterateType(String className, boolean polymorphic)` - Iterate type records
- `scanBucket(String bucketName, RecordCallback callback)` - Scan with callback
- `scanType(String className, boolean polymorphic, DocumentCallback callback)` - Scan type with callback

**Size:**
- `getSize()` - Get database size in bytes

### Schema Management

#### Creating Types
```java
Schema schema = db.getSchema();

// Create document type
DocumentType employee = schema.createDocumentType("Employee");

// Create vertex type
VertexType person = schema.createVertexType("Person");

// Create edge type
EdgeType manages = schema.createEdgeType("Manages");
```

#### Creating Properties
```java
schema.createProperty("Employee", "name", Type.STRING);
schema.createProperty("Employee", "salary", Type.LONG);
```

#### Creating Indexes
```java
TypeIndex index = schema.createTypeIndex(
  SchemaImpl.INDEX_TYPE.LSM_TREE,
  true,  // unique
  "Employee",
  new String[]{"name"}
);
```

### Transactions

#### Synchronous Transactions
```java
db.begin();
try {
  // Create/modify records
  MutableDocument doc = db.newDocument("User");
  doc.set("name", "John");
  doc.set("age", 30);
  doc.save();

  db.commit(); // All or nothing
} catch (Exception e) {
  db.rollback();
}
```

#### Asynchronous Transactions
```java
db.transaction(() -> {
  // Create/modify records
  MutableDocument doc = db.newDocument("User");
  doc.set("name", "John");
  doc.save();
  // Auto commits if no exception, auto rollback on exception
});
```

### Query Execution

#### Simple Query
```java
ResultSet resultSet = db.query("sql", "SELECT FROM User WHERE age > 30");
while (resultSet.hasNext()) {
  Result result = resultSet.next();
  System.out.println("User: " + result.getProperty("name"));
}
```

#### Parameterized Query (Positional)
```java
ResultSet resultSet = db.query("sql",
  "SELECT FROM User WHERE age > ? AND city = ?",
  30, "NewYork");
```

#### Parameterized Query (Named)
```java
Map<String, Object> params = new HashMap<>();
params.put("age", 30);
params.put("city", "NewYork");

ResultSet resultSet = db.query("sql",
  "SELECT FROM User WHERE age > :age AND city = :city",
  params);
```

### Events

#### Record Listeners
```java
// Listen before create
database.getEvents().registerListener((BeforeRecordCreateListener)
  record -> record instanceof Vertex
    && record.asVertex().getBoolean("validated") == false
);

// Listen after create
database.getEvents().registerListener((AfterRecordCreateListener)
  record -> System.out.println("Created: " + record)
);

// Listen before update
database.getEvents().registerListener((BeforeRecordUpdateListener)
  record -> record.asVertex().getBoolean("validated")
);
```

#### Type-Level Listeners
```java
database.getSchema().getType("Client").getEvents()
  .registerListener((BeforeRecordCreateListener)
    record -> record.asVertex().getBoolean("validated")
  );
```

---

## Java API (Remote)

### Setup

#### Maven Dependencies
```xml
<dependency>
  <groupId>com.arcadedb</groupId>
  <artifactId>arcadedb-engine</artifactId>
  <version>26.2.1</version>
</dependency>
<dependency>
  <groupId>com.arcadedb</groupId>
  <artifactId>arcadedb-network</artifactId>
  <version>26.2.1</version>
</dependency>
```

### Key Classes

#### RemoteServer
Manages remote server connections and database lifecycle operations.

**Constructor:**
```java
RemoteServer server = new RemoteServer(
  "localhost",  // hostname/IP
  2480,         // HTTP port
  "root",       // username
  "playwithdata" // password
);
```

**Methods:**
- `exists(String database)` - Check if database exists
- `create(String database)` - Create new database
- `databases()` - List all databases
- `close()` - Close server connection

**Example:**
```java
RemoteServer server = new RemoteServer("localhost", 2480, "root", "playwithdata");
if (!server.exists("mydb")) {
  server.create("mydb");
}
```

#### RemoteDatabase
Works with a specific remote database instance.

**Constructor:**
```java
RemoteDatabase database = new RemoteDatabase(
  "localhost",      // hostname
  2480,             // HTTP port
  "mydb",           // database name
  "root",           // username
  "playwithdata"    // password
);
```

**Important:** RemoteDatabase is NOT thread-safe. Create a new instance per thread.

**Methods:**
- `query(String language, String command, Map<String,Object> params)` - Query with named params
- `query(String language, String command, Object... positionalParams)` - Query with positional params
- `command(String language, String command, Map<String,Object> params)` - Execute command
- `command(String language, String command, Object... positionalParams)` - Execute command
- `begin()` - Start transaction
- `commit()` - Commit transaction
- `rollback()` - Rollback transaction

### Remote Connection Example

```java
RemoteServer server = new RemoteServer("localhost", 2480, "root", "playwithdata");

try (RemoteDatabase database = new RemoteDatabase(
    "localhost", 2480, "mydb", "root", "playwithdata")) {

  // Create schema
  String schema = """
    CREATE VERTEX TYPE Customer IF NOT EXISTS;
    CREATE PROPERTY Customer.name IF NOT EXISTS STRING;
    CREATE PROPERTY Customer.surname IF NOT EXISTS STRING;
    CREATE INDEX IF NOT EXISTS ON Customer (name, surname) UNIQUE;
    """;

  database.command("sqlscript", schema);

  // Insert data
  database.command("sql",
    "INSERT INTO Customer(name, surname) VALUES(?, ?)",
    "Jay", "Miner");

  // Query data
  ResultSet resultSet = database.query("sql",
    "SELECT FROM Customer WHERE name = ?", "Jay");

  while (resultSet.hasNext()) {
    Result result = resultSet.next();
    System.out.println("Customer: " + result.toMap());
  }
}
```

---

## Java API (gRPC)

### Overview
High-performance gRPC-based Java library for remote database communication. Uses HTTP/2 protocol for better latency and throughput.

### Advantages over HTTP
- **High Performance:** HTTP/2 transport with lower latency
- **Efficient Binary Protocol:** Uses Protocol Buffers for smaller message sizes
- **Streaming Support:** Bidirectional streaming for real-time applications
- **Language Agnostic:** Consistent APIs across multiple languages
- **Low Latency:** Optimized for scenarios requiring minimal latency
- **Connection Multiplexing:** Multiple requests over single TCP connection

### Setup

#### Maven Dependency
```xml
<dependency>
  <groupId>com.arcadedb</groupId>
  <artifactId>arcadedb-grpc-client</artifactId>
  <version>26.2.1</version>
</dependency>
```

### Key Classes

#### RemoteGrpcServer
Manages gRPC server connections and server-level operations.

**Constructor:**
```java
RemoteGrpcServer server = new RemoteGrpcServer(
  "localhost",     // hostname
  50051,           // gRPC port (default: 50051)
  "root",          // username
  "password",      // password
  false,           // useTls (false for development)
  List.of()        // channelCredentials (optional)
);
```

**Methods:**
- `exists(String database)` - Check if database exists
- `create(String database)` - Create new database
- `databases()` - List all databases

#### RemoteGrpcDatabase
Works with a specific remote database via gRPC.

**Constructor:**
```java
RemoteGrpcDatabase database = new RemoteGrpcDatabase(
  server,          // RemoteGrpcServer instance
  "localhost",     // hostname
  50051,           // gRPC port
  2480,            // HTTP port (for fallback operations)
  "mydb",          // database name
  "root",          // username
  "password"       // password
);
```

**Important:** RemoteGrpcDatabase is NOT thread-safe. Create new instances per thread.

**Methods:**
- `query(String language, String command, Map<String,Object> parameterMap)` - Query with named params
- `query(String language, String command, Object... positionalParameters)` - Query with positional params
- `command(String language, String command, Map<String,Object> parameterMap)` - Execute command
- `command(String language, String command, Object... positionalParameters)` - Execute command
- `begin()` - Start transaction
- `commit()` - Commit transaction
- `rollback()` - Rollback transaction
- `close()` - Close connection

### Query Execution

#### Simple Query
```java
ResultSet resultSet = database.query("sql", "SELECT * FROM Article");

while (resultSet.hasNext()) {
  Result result = resultSet.next();
  System.out.println("Article: " + result.toMap());
}
```

#### Parameterized Query (Named Parameters)
```java
ResultSet resultSet = database.query("sql",
  "SELECT * FROM Article WHERE author = :author AND published > :date",
  Map.of(
    "author", "John Doe",
    "date", LocalDateTime.now().minusDays(30)
  )
);
```

#### Parameterized Query (Positional Parameters)
```java
ResultSet resultSet = database.query("sql",
  "SELECT * FROM Article WHERE id = ? AND published = ?",
  1,
  LocalDateTime.now()
);
```

### Data Manipulation

#### Insert Data
```java
// Traditional INSERT with values
database.command("sql",
  "INSERT INTO Article (id, title, author, published) VALUES (?, ?, ?, ?)",
  1,
  "Getting Started with ArcadeDB",
  "Jane Doe",
  LocalDateTime.now()
);

// INSERT with named parameters
database.command("sql",
  "INSERT INTO Article (id, title, author, published) VALUES (:id, :title, :author, :published)",
  Map.of(
    "id", 2,
    "title", "Advanced Query Techniques",
    "author", "John Doe",
    "published", LocalDateTime.now()
  )
);
```

#### Update Data
```java
database.command("sql",
  "UPDATE Article SET title = ? WHERE id = ?",
  "Updated Title",
  3
);

database.command("sql",
  "UPDATE Article SET title = :title, views = views + 1 WHERE id = :id",
  Map.of("title", "Updated Title", "id", 3)
);
```

#### Insert with RETURN Clause
```java
ResultSet result = database.command("sql",
  "CREATE VERTEX Article SET id = :id, title = :title, author = :author RETURN @this",
  Map.of("id", 100, "title", "Test Article", "author", "Test Author")
);

if (result.hasNext()) {
  Result vertex = result.next();
  String rid = vertex.getProperty("@rid").toString();
  System.out.println("Created vertex with RID: " + rid);
}
```

#### Delete Data
```java
database.command("sql",
  "DELETE FROM Article WHERE id = ?",
  1
);
```

### Vertex and Edge Management

#### Create Vertices
```java
ResultSet author = database.command("sql",
  "CREATE VERTEX Author SET name = ? RETURN @rid",
  "Emily Davis"
);

if (author.hasNext()) {
  String authorRid = author.next().getProperty("@rid").toString();

  ResultSet article = database.command("sql",
    "CREATE VERTEX Article SET title = ? RETURN @rid",
    "Mastering Graph Databases"
  );

  if (article.hasNext()) {
    String articleRid = article.next().getProperty("@rid").toString();

    // Create edge connecting them
    database.command("sql",
      "CREATE EDGE Authored FROM ? TO ?",
      authorRid,
      articleRid
    );
  }
}
```

### Transaction Management

#### Explicit Transactions
```java
database.begin();
try {
  database.command("sql", "CREATE VERTEX Article SET id = 200");
  database.command("sql", "CREATE VERTEX Article SET id = 201");
  database.commit();
  System.out.println("Transaction committed successfully");
} catch (Exception e) {
  database.rollback();
  System.err.println("Transaction rolled back: " + e.getMessage());
}
```

#### Transaction Lambda (Auto-commit/rollback)
```java
database.transaction(() -> {
  database.command("sql", "CREATE VERTEX Article SET id = 300");
  database.command("sql", "CREATE VERTEX Article SET id = 301");
  // Automatically commits if no exception, rolls back on exception
});
```

### Error Handling

```java
try (RemoteGrpcDatabase database = new RemoteGrpcDatabase(...)) {
  ResultSet results = database.query("sql", "SELECT * FROM Article");

} catch (SecurityException e) {
  System.err.println("Authentication failed: " + e.getMessage());
} catch (DatabaseOperationException e) {
  System.err.println("Database operation failed: " + e.getMessage());
} catch (Exception e) {
  System.err.println("Unexpected error: " + e.getMessage());
  e.printStackTrace();
}
```

### Performance Tips

1. **Connection Pooling:** Create a single RemoteGrpcServer and reuse across application
2. **Batch Operations:** Group multiple operations in transactions to reduce round-trips
3. **Parameterized Queries:** Always use parameters for better performance and security
4. **Use sqlscript:** For multiple statements, use sqlscript language to execute in single transaction
5. **Stream Results:** For large result sets, process results as you iterate rather than loading all into memory
6. **Thread Safety:** Create new RemoteGrpcDatabase instance per thread
7. **TLS in Production:** Enable TLS/SSL with useT ls=true for production deployments

---

## Python Connectivity

### Available Drivers

| Driver | URL | License | Status |
|--------|-----|---------|--------|
| arcadedb-python-driver | https://github.com/adaros92/arcadedb-python-driver | Apache 2 | Active |
| pyarcade | https://github.com/ExtREMLapin/pyarcade | Apache 2 | Active |
| arcadedb-python | https://github.com/stevereiner/arcadedb-python | Apache 2 | Active |
| arcadedb-embedded-python | https://github.com/humemai/arcadedb-embedded-python | Apache 2 | Embedded mode |

### Installation

```bash
pip install arcadedb-python-driver
```

### Basic Usage

```python
from arcadedb import ArcadeDBClient

# Connect to ArcadeDB server
client = ArcadeDBClient(
    host="localhost",
    port=2480,
    username="root",
    password="playwithdata"
)

# Open database
db = client.open("mydb")

# Execute query
result = db.query("SELECT FROM User WHERE age > 30")
for record in result:
    print(f"User: {record.get('name')}")

# Insert data
db.execute("INSERT INTO User(name, age) VALUES(?, ?)", "John", 30)

# Close connection
db.close()
```

### Query Execution

```python
# Simple query
result = db.query("SELECT * FROM User")

# Parameterized query
result = db.query("SELECT FROM User WHERE age > ?", [30])

# Named parameters
result = db.query(
    "SELECT FROM User WHERE age > :age AND city = :city",
    {"age": 30, "city": "New York"}
)

# Command execution
db.execute("INSERT INTO User(name, age) VALUES(?, ?)", ["Alice", 25])
```

### Transaction Management

```python
# Begin transaction
db.begin()
try:
    db.execute("INSERT INTO User(name) VALUES(?)", ["Bob"])
    db.commit()
except Exception as e:
    db.rollback()
    print(f"Transaction failed: {e}")
```

---

## NodeJS/JavaScript Connectivity

### Available Drivers

| Driver | URL | License |
|--------|-----|---------|
| sprite.tragedy.dev | https://sprite.tragedy.dev/ | MIT |

### Installation

```bash
npm install arcadedb
```

### Basic Usage

```javascript
const ArcadeDB = require('arcadedb');

// Connect to ArcadeDB
const db = new ArcadeDB({
  host: 'localhost',
  port: 2480,
  username: 'root',
  password: 'playwithdata',
  database: 'mydb'
});

// Execute query
db.query('SELECT FROM User WHERE age > ?', [30])
  .then(result => {
    result.forEach(record => {
      console.log(`User: ${record.name}`);
    });
  })
  .catch(error => console.error(error));

// Insert data
db.execute('INSERT INTO User(name, age) VALUES(?, ?)', ['John', 30])
  .then(result => console.log('Record created'))
  .catch(error => console.error(error));
```

### Async/Await Pattern

```javascript
async function example() {
  try {
    // Query
    const users = await db.query('SELECT FROM User WHERE age > 30');
    console.log(users);

    // Insert
    await db.execute('INSERT INTO User(name, age) VALUES(?, ?)',
      ['Alice', 28]);

    // Transaction
    await db.begin();
    await db.execute('INSERT INTO User(name) VALUES(?)', ['Bob']);
    await db.execute('INSERT INTO User(name) VALUES(?)', ['Charlie']);
    await db.commit();
  } catch (error) {
    console.error(error);
    await db.rollback();
  }
}
```

---

## PostgreSQL Protocol Plugin

### Overview
Allows running ArcadeDB with PostgreSQL protocol support. Compatible with PostgreSQL drivers and tools.

### Installation

**Via Command Line:**
```bash
./bin/server.sh -Darcadedb.server.plugins=PostgreSQL:com.arcadedb.postgres.PostgreSQLProtocolPlugin
```

**Via Docker:**
```bash
docker run --rm -p 2480:2480 -p 5432:5432 \
  -e JAVA_OPTS="-Darcadedb.server.plugins=PostgreSQL:com.arcadedb.postgres.PostgreSQLProtocolPlugin" \
  arcadedata/arcadedb:latest
```

### Configuration

Default port: 5432

### Usage

Any PostgreSQL client can connect:

```bash
psql -h localhost -p 5432 -U root -d mydb
```

**Java JDBC:**
```java
Connection conn = DriverManager.getConnection(
  "jdbc:postgresql://localhost:5432/mydb",
  "root",
  "password"
);
```

**Python psycopg2:**
```python
import psycopg2

conn = psycopg2.connect(
  host="localhost",
  port=5432,
  database="mydb",
  user="root",
  password="password"
)
```

### Limitations
- Supports subset of PostgreSQL protocol
- See GitHub issues for feature requests

---

## Neo4j BOLT Protocol Plugin

### Overview
Allows running ArcadeDB with Neo4j BOLT protocol support. Compatible with Neo4j drivers.

### Installation

**Via Command Line:**
```bash
./bin/server.sh -Darcadedb.server.plugins=Neo4jBOLT:com.arcadedb.neo4j.BoltProtocolPlugin
```

**Via Docker:**
```bash
docker run --rm -p 2480:2480 -p 7687:7687 \
  -e JAVA_OPTS="-Darcadedb.server.plugins=Neo4jBOLT:com.arcadedb.neo4j.BoltProtocolPlugin" \
  arcadedata/arcadedb:latest
```

### Configuration

Default port: 7687

### Usage

Any Neo4j driver can connect:

**Java:**
```java
Driver driver = GraphDatabase.driver(
  "bolt://localhost:7687",
  AuthTokens.basic("root", "password")
);
Session session = driver.session();
Result result = session.run("MATCH (n) RETURN n LIMIT 10");
```

**Python neo4j:**
```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver(
  "bolt://localhost:7687",
  auth=("root", "password")
)
with driver.session() as session:
    result = session.run("MATCH (n) RETURN n LIMIT 10")
    for record in result:
        print(record)
```

**JavaScript neo4j:**
```javascript
const neo4j = require('neo4j-driver');

const driver = neo4j.driver(
  'bolt://localhost:7687',
  neo4j.auth.basic('root', 'password')
);

const session = driver.session();
const result = await session.run('MATCH (n) RETURN n LIMIT 10');
session.close();
driver.close();
```

### Limitations
- Supports subset of Neo4j BOLT protocol
- See GitHub discussions to request more commands support

---

## Redis Plugin

### Overview
ArcadeDB supports a subset of Redis protocol. Works in two modes:
- **Transient Commands:** Key/value pairs saved in RAM (not persistent)
- **Persistent Commands:** Documents, vertices, and edges stored in database

### Installation

**Setup:**
```bash
./bin/server.sh -Darcadedb.server.plugins=Redis:com.arcadedb.redis.RedisProtocolPlugin
```

**Docker:**
```bash
docker run --rm -p 2480:2480 -p 6379:6379 \
  -e JAVA_OPTS="-Darcadedb.server.rootPassword=playwithdata -Darcadedb.server.plugins=Redis:com.arcadedb.redis.RedisProtocolPlugin" \
  arcadedata/arcadedb:latest
```

Default Redis port: 6379

### Transient Commands (RAM)

Available commands (do NOT persist on server restart):
- **DECR** - Decrement value by 1
- **DECRBY** - Decrement by specific amount (64-bit precision)
- **EXISTS** - Check if key exists
- **GET** - Return value for key
- **GETDEL** - Remove and return value for key
- **INCR** - Increment value by 1
- **INCRBY** - Increment by specific amount (64-bit precision)
- **INCRBY FLOAT** - Increment by float amount (64-bit precision)
- **SET** - Set value for key

### Persistent Commands (Database)

Available commands (stored in ArcadeDB documents):
- **HDEL** - Delete one or more records by key, composite key, or RID
- **HEXISTS** - Check if key exists
- **HGET** - Retrieve record by key, composite key, or RID
- **HMGET** - Retrieve multiple records by key, composite key, or RID

### Usage Example

```bash
redis-cli -h localhost -p 6379
```

**Create document type and indexes:**
```sql
CREATE DOCUMENT TYPE Account;
CREATE PROPERTY Account.id LONG;
CREATE INDEX ON Account (id) UNIQUE;
CREATE PROPERTY Account.email STRING;
CREATE INDEX ON Account (email) UNIQUE;
CREATE PROPERTY Account.firstName STRING;
CREATE PROPERTY Account.lastName STRING;
CREATE INDEX ON Account (firstName, lastName) UNIQUE;
```

**Transient commands:**
```bash
# SET key-value
SET mykey myvalue

# GET value
GET mykey

# INCR counter
INCR counter

# DECR counter
DECR counter
```

**Persistent commands:**
```bash
# HSET - Create/update document
HSET MyDatabase.Account {"id":123,"email":"jay.miner@commodore.com","firstName":"Jay","lastName":"Miner"}

# HGET - Retrieve by ID
HGET MyDatabase.Account[id] 123
# Returns: {'@rid':'#1:0','@type':'Account','id':123,'email':'jay.miner@commodore.com','firstName':'Jay','lastName':'Miner'}

# HGET - Retrieve by email
HGET MyDatabase.Account[email] "jay.miner@commodore.com"

# HGET - Retrieve by composite key
HGET MyDatabase.Account[firstName,lastName] ["Jay","Miner"]

# HGET - Retrieve by RID
HGET MyDatabase "#1:0"
# Returns: {'@rid':'#1:0','@type':'Account','id':123,'email':'jay.miner@commodore.com','firstName':'Jay','lastName':'Miner'}

# HMGET - Retrieve multiple records by RID
HMGET MyDatabase "#1:0" "#1:1" "#1:2"

# HMGET - Retrieve multiple by key
HMGET MyDatabase.Account[id] 123 232 12

# HDEL - Delete by email
HDEL MyDatabase.Account[email] "jay.miner@commodore.com"

# Redis Transactions
MULTI
SET key1 value1
SET key2 value2
INCR counter
EXEC
```

### Batch Commands with Comments

```bash
# This is a comment
SET key1 value1
// This is also a comment
SET key2 value2
GET key1
GET key2
```

### Settings

- `arcadedb.redis.host` - Host Redis protocol listens on (default: 0.0.0.0)
- `arcadedb.redis.port` - Port Redis protocol listens on (default: 6379)

---

## JDBC Driver

### Overview
Java Database Connectivity (JDBC) driver for ArcadeDB. Bundled with ArcadeDB project.

### Setup

**Maven:**
```xml
<dependency>
  <groupId>com.arcadedb</groupId>
  <artifactId>arcadedb-engine</artifactId>
  <version>26.2.1</version>
</dependency>
```

### Connection String

```java
Connection conn = DriverManager.getConnection(
  "jdbc:arcadedb:http://localhost:2480/mydb",
  "root",
  "password"
);
```

### Basic Usage

```java
// Create connection
Connection conn = DriverManager.getConnection(
  "jdbc:arcadedb:http://localhost:2480/mydb",
  "root",
  "password"
);

// Execute query
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM User");

while (rs.next()) {
  String name = rs.getString("name");
  int age = rs.getInt("age");
  System.out.println(name + ", " + age);
}

rs.close();
stmt.close();
conn.close();
```

### Prepared Statements

```java
PreparedStatement pstmt = conn.prepareStatement(
  "SELECT * FROM User WHERE age > ? AND city = ?"
);
pstmt.setInt(1, 30);
pstmt.setString(2, "New York");

ResultSet rs = pstmt.executeQuery();
while (rs.next()) {
  // Process results
}
```

### Execute Updates

```java
Statement stmt = conn.createStatement();

// Insert
stmt.executeUpdate("INSERT INTO User(name, age) VALUES('John', 30)");

// Update
stmt.executeUpdate("UPDATE User SET age = 31 WHERE name = 'John'");

// Delete
stmt.executeUpdate("DELETE FROM User WHERE name = 'John'");

stmt.close();
```

### Metadata

```java
DatabaseMetaData metaData = conn.getMetaData();
ResultSet tables = metaData.getTables(null, null, "%", null);
while (tables.next()) {
  String tableName = tables.getString("TABLE_NAME");
  System.out.println("Table: " + tableName);
}
```

---

## Quick Reference Table

| API | Local | Remote | Protocol | Best For |
|-----|-------|--------|----------|----------|
| **Java API (Local)** | ✓ | ✗ | N/A | Embedded applications, single JVM |
| **Java API (Remote)** | ✗ | ✓ | HTTP | Remote applications, simple integration |
| **Java API (gRPC)** | ✗ | ✓ | HTTP/2 | High-performance, low-latency applications |
| **Python** | ✗ | ✓ | HTTP | Python applications and data science |
| **Node.js/JS** | ✗ | ✓ | HTTP | JavaScript/Node.js applications |
| **PostgreSQL Plugin** | ✗ | ✓ | PostgreSQL | PostgreSQL-compatible tools and drivers |
| **Neo4j BOLT** | ✗ | ✓ | BOLT | Neo4j-compatible drivers and tools |
| **Redis Plugin** | ✗ | ✓ | Redis | Redis-compatible clients and tools |
| **JDBC** | ~ | ✓ | HTTP | Traditional Java applications |

---

## Best Practices

### Security
- Always use parameterized queries to prevent injection
- Use HTTPS/TLS in production for remote connections
- Never hardcode credentials; use environment variables or secure vaults
- Change default credentials in production environments

### Performance
- Use batch operations when inserting/updating multiple records
- Leverage indexes on frequently queried properties
- Use appropriate transaction boundaries
- For large result sets, stream results instead of loading all into memory
- Use connection pooling for remote connections

### Error Handling
- Always catch and handle database exceptions appropriately
- Use try-with-resources for automatic resource cleanup
- Log errors with context for debugging
- Implement retry logic for transient failures

### Data Integrity
- Use transactions for operations requiring ACID properties
- Define proper indexes for query optimization
- Validate input data before insertion
- Use constraints to maintain data consistency

---

## Document Version

**Last Updated:** From ArcadeDB Manual Pages 434-540 (Chapter 10: API and Driver Reference)
**ArcadeDB Version:** 26.2.1+
**Document Format:** Markdown Reference Guide
