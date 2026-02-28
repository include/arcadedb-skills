# ArcadeDB Additional Query Languages Reference

## Overview

Beyond SQL and OpenCypher (covered in their own reference files), ArcadeDB supports four additional query languages and protocol interfaces: Gremlin, GraphQL, MongoDB Query Language, and Redis. Each provides a familiar interface for developers coming from those ecosystems, allowing them to work with ArcadeDB using tools and drivers they already know.

### Language Identifier Quick Reference

| Language   | Identifier   | Use Case                                    |
|------------|-------------|---------------------------------------------|
| Gremlin    | `"gremlin"` | TinkerPop graph traversals                  |
| GraphQL    | `"graphql"` | Schema-driven queries with directives       |
| MongoDB QL | `"mongo"`   | Document-oriented CRUD with BSON protocol   |
| Redis      | `"redis"`   | Key/value operations, caching, sessions     |

### Access Methods (All Languages)

Every query language can be executed through multiple interfaces:

- **Java API**: `database.query("<language>", "<query>")` or `database.command("<language>", "<query>")`
- **HTTP/JSON API**: `/query` (idempotent) and `/command` (non-idempotent) endpoints with `language` parameter
- **PostgreSQL Driver**: Prefix query with `{<language>}` (e.g., `{gremlin}`, `{graphql}`, `{mongo}`)

---

## Gremlin

### Overview

ArcadeDB supports Apache TinkerPop Gremlin 3.7.x as a query language and protocol. Gremlin is a graph traversal language that allows you to express complex graph queries as a series of chained steps. You can execute Gremlin queries from nearly every access method ArcadeDB provides.

### CRITICAL SECURITY WARNING

> **DO NOT use the Groovy Gremlin engine** (`arcadedb.gremlin.engine=groovy`) in production or with untrusted users. The Groovy engine has a **Critical Remote Code Execution (RCE) vulnerability** that allows authenticated users to execute arbitrary operating system commands. ArcadeDB uses the secure Java Gremlin engine by default (from version 25.12.1). Only the default Java engine should be used.

### Maven Dependency (Embedded)

If you are using ArcadeDB as an embedded database, add the `arcadedb-gremlin` dependency:

```xml
<dependency>
    <groupId>com.arcadedb</groupId>
    <artifactId>arcadedb-gremlin</artifactId>
    <version>26.2.1</version>
</dependency>
```

You also need `apache-tinkerpop-gremlin-server`, `gremlin-groovy`, and `opencypher-util-9.0` on your classpath.

### Gremlin from Java API

Use `"gremlin"` as the language identifier in the `query()` method:

```java
ResultSet result = database.query("gremlin",
    "g.V().has('name','Michelle').has('lastName','Besso').out('IsFriendOf')");
```

For applications primarily using Gremlin, use the `ArcadeGraph` class as an entry point:

```java
// Embedded database
try (final ArcadeGraph graph = ArcadeGraph.open("./databases/graph")) {
    Vertex michelleFriend = graph.traversal().V()
        .has("name", "Michelle")
        .has("lastName", "Besso")
        .out("IsFriendOf").next();
}
```

For remote databases, connect through `ArcadeGraph` with a `RemoteDatabase`:

```java
try (RemoteDatabase database = new RemoteDatabase("127.0.0.1", 2480, "graph", "root",
    "playwithdata")) {
    try (final ArcadeGraph graph = ArcadeGraph.open(database)) {
        Vertex michelleFriend = graph.traversal().V()
            .has("name", "Michelle")
            .has("lastName", "Besso")
            .out("IsFriendOf").next();
    }
}
```

For multi-threaded applications, use `ArcadeGraphFactory` to acquire pooled `ArcadeGraph` instances (since `RemoteDatabase` is not thread-safe):

```java
try (ArcadeGraphFactory pool = ArcadeGraphFactory.withRemote("127.0.0.1", 2480,
    "mydb", "root", "playwithdata")) {
    try (ArcadeGraph graph = pool.get()) {
        // DO SOMETHING WITH ARCADE GRAPH
    }
}
```

### Gremlin through PostgreSQL Driver

Prefix the query with `{gremlin}`:

```
"{gremlin} g.V().has('name','Michelle').has('lastName','Besso').out('IsFriendOf')"
```

ArcadeDB server will execute the query using the Gremlin query language.

### Gremlin through HTTP/JSON

**Idempotent query (HTTP GET)**:

```bash
curl "http://localhost:2480/query/graph/gremlin/g.V().has('name','Michelle').has('lastName','Besso').out('IsFriendOf')"
```

**Non-idempotent command (HTTP POST)**:

```bash
curl -X POST "http://localhost:2480/command/graph" \
  -d "{'language': 'gremlin', 'command': 'g.V().has(\"name\",\"Michelle\").has(\"lastName\",\"Besso\").out(\"IsFriendOf\")'}"
```

### Use the Gremlin Server

Apache TinkerPop Gremlin provides a standalone server to allow remote access with a Gremlin client. Enable it from ArcadeDB's server plugin system:

```bash
~/arcadedb $ bin/server.sh -Darcadedb.server.plugins\
  ="GremlinServer:com.arcadedb.server.gremlin.GremlinServerPlugin"
```

The Gremlin Server plugin looks for `config/gremlin-server.yaml` under the ArcadeDB path. If present, it uses those settings; otherwise it uses the default configuration.

You can override individual configuration settings by prefixing the configuration key with `gremlin.`. All settings with that prefix are passed to the Gremlin Server plugin.

#### Docker Example

Start the Gremlin Server with Docker (expose port `8182` and use `-e` for JVM settings):

```bash
docker run -d --name arcadeDB -p 2424:2424 -p 2480:2480 -p 8182:8182 \
  --env JAVA_OPTS="-Darcadedb.server.rootPassword=playwithdata \
  -Darcadedb.server.defaultDatabases=openbeer[root]{import:https://github.com/ArcadeData/arcadedb-datasets/raw/main/orientdb/OpenBeer.gz} \
  -Darcadedb.server.plugins=GremlinServer:com.arcadedb.server.gremlin.GremlinServerPlugin" \
  arcadedata/arcadedb:latest
```

#### Connecting to the Gremlin Server

```java
var cluster = Cluster.build()
    .port(8182)
    .addContactPoint("localhost")
    .credentials("root", "password")
    .create();

// Connect using the database name as the alias
var g = traversal().withRemote(DriverRemoteConnection.using(cluster, "openbeer"));

// Or use "g" for the default database
var gDefault = traversal().withRemote(DriverRemoteConnection.using(cluster, "g"));
```

### Multi-Database Support (v26.2.1+)

Starting from version 26.2.1, ArcadeDB's Gremlin Server includes `ArcadeGraphManager`, a custom implementation of TinkerPop's `GraphManager` interface that provides **dynamic, on-demand registration** of databases as Gremlin graphs.

#### Key Features

- **Automatic Registration**: When a Gremlin client requests a graph/traversal source by database name, `ArcadeGraphManager` automatically registers it if the database exists in ArcadeDB Server
- **Multi-Database Access**: Access any database on the server through Gremlin without static configuration
- **The `g` Alias**: The standard `g` alias automatically maps to the default database ("graph" if it exists, otherwise the first available database)

#### How It Works

Instead of static graph configuration in YAML files, databases are registered dynamically when first accessed. Each database name becomes a valid traversal source alias:

```java
// Connect to a specific database using its name as the alias
var cluster = Cluster.build()
    .port(8182)
    .addContactPoint("localhost")
    .credentials("root", "password")
    .create();

// Access database "mydb" - registered automatically on first access
var g = traversal().withRemote(DriverRemoteConnection.using(cluster, "mydb"));

// Or use the standard "g" alias for the default database
var gDefault = traversal().withRemote(DriverRemoteConnection.using(cluster, "g"));
```

#### Script-Based Queries

For script-based Gremlin queries (e.g., via `client.submit()`), the `g` variable is automatically bound to the default database:

```java
Client client = cluster.connect();
// The "g" variable is automatically available
ResultSet results = client.submit("g.V().hasLabel('Person').count()");
```

#### Accessing Multiple Databases

To access a specific database, use the database name as the traversal source alias:

```java
// Before v26.2.1 - required manual configuration
// Now - just use the database name directly
var g = traversal().withRemote(DriverRemoteConnection.using(cluster, "customers"));
var g2 = traversal().withRemote(DriverRemoteConnection.using(cluster, "products"));
```

#### Transactions via Remote Driver

Gremlin transactions work correctly via the remote driver:

```java
GraphTraversalSource g = traversal().withRemote(
    DriverRemoteConnection.using(cluster, "mydb"));

Transaction tx = g.tx();
GraphTraversalSource gtx = tx.begin();
try {
    gtx.addV("Person").property("name", "Alice").iterate();
    tx.commit();
} catch (Exception e) {
    tx.rollback();
}
```

#### Configuration Migration (v26.2.1)

If upgrading from a version prior to 26.2.1:

1. **Graph Registration**: Graphs are now registered dynamically by `ArcadeGraphManager` instead of statically in configuration files
2. **Configuration Files**: The `graphs:` section in `gremlin-server.yaml` is no longer required
3. **The `graph` Binding**: The internal `graph` variable is now managed by `ArcadeGraphManager`

**Backward Compatibility**: Existing Gremlin queries, the `g` alias, script-based queries, and bytecode-based queries all continue to work without modification.

If you customized `gremlin-server.groovy`, ensure you are NOT overwriting the `graph` binding:

```groovy
// CORRECT: Only define the traversal source 'g'
globals << [g : traversal().withEmbedded(graph)]

// WRONG: Do NOT overwrite 'graph' - this breaks session support
// globals << [graph : traversal().withEmbedded(graph)]  // DON'T DO THIS
```

### Known Limitations

- **Type conversion**: ArcadeDB automatically handles conversion between compatible types (such as strings and numbers) when possible. Gremlin does not. If you define a schema with the ArcadeDB API and then use Gremlin, ensure you use the same types defined in the schema. For example, if a property "id" is defined as a string, executing traversals using integers for the ids will produce unpredictable results.

- **Complex traversal optimization**: ArcadeDB's Gremlin implementation always tries to optimize the traversal by using ArcadeDB's internal query. While this is easy with simple traversals using `.has()` and `.hasLabel()`, it is unable to optimize more complex traversals with `select()` and `where()`. Instead of executing an optimized query, it could result in a full scan of the type, leaving Gremlin to do the filtering. The result is still correct, but performance would be heavily impacted. **Recommend using ArcadeDB's SQL or OpenCypher for complex traversals.**

### Recommended Tools

If you are using Gremlin with ArcadeDB, check out [G.V()](https://gdotv.com/) -- a graphic tool compatible with ArcadeDB that provides a visual debugger, advanced graph analytics, and more.

---

## GraphQL

### Overview

ArcadeDB Server supports a subset of the GraphQL specification. GraphQL provides a schema-driven approach to querying your data, letting you define types that map to your ArcadeDB types and use directives to express graph relationships, SQL queries, Gremlin traversals, and Cypher patterns.

### Maven Dependency (Embedded)

```xml
<dependency>
    <groupId>com.arcadedb</groupId>
    <artifactId>arcadedb-graphql</artifactId>
    <version>26.2.1</version>
</dependency>
```

### Execution Methods

**Java API** using the non-idempotent `.command()` and the idempotent `.query()` with `"graphql"` as language:

```java
Resultset resultset = db.query("graphql",
    "{ bookById(id: \"book-1\"){ id name authors { firstName, lastName } } }");
```

**HTTP API** using `/command` and `/query` with `"graphql"` as language:

```
HTTP POST /command with language="graphql"
HTTP GET  /query  with language="graphql"
```

**PostgreSQL Driver** by prefixing with `{graphql}`:

```
"{graphql} { bookById(id: \"book-1\"){ id name authors { firstName, lastName } } }"
```

### Type Definition

GraphQL requires you to define the types used. If you are using the Document Model and links to connect documents, map the GraphQL type 1-1 to the ArcadeDB type:

```graphql
type Book {
    id: ID
    name: String
    pageCount: Int
    authors: [Author]
}
```

If you are using a Graph Model, declare the relationship with a GraphQL directive to describe how the relationship is translated on the graph model. In this example, `authors` is a collection of Author retrieved by looking at the incoming (`direction: IN`) edges of type "IS_AUTHOR_OF":

```graphql
type Book {
    id: ID
    name: String
    pageCount: Int
    authors: [Author] @relationship(type: "IS_AUTHOR_OF", direction: IN)
}
```

> **Important**: Type definitions are NOT saved in the database. They must be declared after the database is open, before executing any GraphQL queries.

You can define your model incrementally and apply it by executing a command containing the type definition. You can add new types or replace existing types by submitting the type(s) again -- the GraphQL module will update the current definition:

```java
String types = "type Query {" +
               "  bookById(id: ID): Book" +
               "}" +
               "type Book {" +
               "  id: ID" +
               "  name: String" +
               "  pageCount: Int" +
               "  authors: [Author] @relationship(type: \"IS_AUTHOR_OF\", direction: IN)" +
               "}" +
               "type Author {" +
               "  id: ID" +
               "  firstName: String" +
               "  lastName: String" +
               "}";

database.command("graphql", types);
```

### Supported Directives

Directives can be defined on both types and queries. Directives defined in queries override any directives defined in types, only for the query execution context.

#### @relationship

Applies to: **Query Field** and **Field Definition**

Syntax: `@relationship([type: "<type-name>"] [, direction: <OUT|IN|BOTH>])`

Where:
- `type` is the edge type, optional. If not specified, all edge types are considered
- `direction` is the direction of the edge, optional. If not specified, BOTH is used

Example:

```graphql
friends: [Account] @relationship(type: "FRIEND", direction: BOTH)
```

#### @sql

Applies to: **Query Field** and **Field Definition**

Syntax: `@sql(statement: "<sql-statement>")`

Executes a SQL query. The query can use parameters passed at invocation time.

Example of defining a query using SQL in GraphQL:

```graphql
bookByName(bookNameParameter: String): Book @sql(statement: "select from Book where name = :bookNameParameter")
```

Invoke the query:

```java
ResultSet resultSet = database.query("graphql",
    "{ bookByName(bookNameParameter: \"Harry Potter and the Philosopher's Stone\")}");
```

#### @gremlin

Applies to: **Query Field** and **Field Definition**

Syntax: `@gremlin(statement: "<gremlin-statement>")`

Executes a Gremlin query. The query can use parameters passed at invocation time.

Example of defining a query using Gremlin in GraphQL:

```graphql
bookByName(bookNameParameter: String): Book @gremlin(statement: "g.V().has('name', bookNameParameter)")
```

Invoke the query:

```java
ResultSet resultSet = database.query("graphql",
    "{ bookByName(bookNameParameter: \"Harry Potter and the Philosopher's Stone\")}");
```

#### @cypher

Applies to: **Query Field** and **Field Definition**

Syntax: `@cypher(statement: "<cypher-statement>")`

Executes a Cypher query. The query can use parameters passed at invocation time.

Example of defining a query using Cypher in GraphQL:

```graphql
bookByName(bookNameParameter: String): Book @cypher(statement: "MATCH (b:Book {name: $bookNameParameter}) RETURN b")
```

Invoke the query:

```java
ResultSet resultSet = database.query("graphql",
    "{ bookByName(bookNameParameter: \"Harry Potter and the Philosopher's Stone\")}");
```

#### @rid

Applies to: **Query Field** and **Field Definition**

Syntax: `@rid`

Marks the field as the record identity (Record ID). Use this to include the ArcadeDB RID in query results.

Example:

```graphql
{ bookById(id: "book-1")
    {
        rid @rid
        id
        name
        authors {
            firstName
            lastName
        }
    }
}
```

---

## MongoDB Query Language

### Overview

ArcadeDB provides support for both the MongoDB Query Language and MongoDB BSON wire protocol. This allows you to use standard MongoDB drivers and tools to interact with ArcadeDB, or use the MongoDB query syntax through ArcadeDB's other access methods.

### Maven Dependency (Embedded)

```xml
<dependency>
    <groupId>com.arcadedb</groupId>
    <artifactId>arcadedb-mongodbw</artifactId>
    <version>26.2.1</version>
</dependency>
```

### MongoDB Query Language

If you want to use MongoDB Query Language from the Java API, keep the relevant jars in your classpath and execute a query or command with `"mongo"` as language:

```java
// CREATE A NEW DATABASE
Database database = new DatabaseFactory("heroes").create();

// CREATE THE DOCUMENT TYPE 'Heroes'
database.getSchema().createDocumentType("Heroes");

// CREATE A NEW DOCUMENT
database.transaction((tx) -> {
    database.newDocument("Heroes").set("name", "Jay").set("lastName", "Miner")
        .set("id", i).save();
});

// EXECUTE A QUERY USING MONGO AS QUERY LANGUAGE
for (ResultSet resultset = database.query("mongo",  // <-- USE 'mongo' INSTEAD OF 'sql'
    "{ collection: 'Heroes', query: { $and: [ { name: { $eq: 'Jay' } }, " +
    "{ lastName: { $exists: true } }, { lastName: { $eq: 'Miner' } }, " +
    "{ lastName: { $ne: 'Miner22' } } ], $orderBy: { id: 1 } } }");
    resultset.hasNext(); ) {
    Result doc = resultset.next();
    // ...
}
```

### Mongo Queries through PostgreSQL Driver

Prefix the query with `{mongo}`:

```
"{mongo} { collection: 'Heroes', query: { $and: [ { name: { $eq: 'Jay' } }, { lastName: { $exists: true } }, { lastName: { $eq: 'Miner' } } ] } }"
```

### Mongo Queries through HTTP/JSON

**Idempotent query (HTTP GET)**:

```bash
curl "http://localhost:2480/query/graph/mongo/{ collection: 'Heroes', query: { $and: [ { name: { $eq: 'Jay' } }, { lastName: { $exists: true } }, { lastName: { $eq: 'Miner' } } ]} }"
```

**Non-idempotent query (HTTP POST)** with language and command in payload:

```bash
curl -X POST "http://localhost:2480/query/graph" \
  -d "{'language': 'mongo', 'command': '{ collection: \"Heroes\", query: { $and: [ { name: { $eq: \"Jay\" } }, { lastName: { $exists: true } }, { lastName: { $eq: \"Miner\" } } ] } }'}"
```

### MongoDB Protocol Plugin

If your application is written for MongoDB, you can replace the MongoDB server with ArcadeDB using the MongoDB Plugin. This plugin supports the MongoDB BSON wire protocol, so you can use any MongoDB driver for any supported programming language. ArcadeDB supports a subset of the MongoDB protocol (CRUD operations and queries).

#### Enable the Plugin

```bash
~/arcadedb $ bin/server.sh -Darcadedb.server.plugins\
  ="MongoDB:com.arcadedb.mongo.MongoDBProtocolPlugin"
```

#### Docker Example (port 27017 is the default MongoDB binary port)

```bash
docker run --rm -p 2480:2480 -p 2424:2424 -p27017:27017 \
  --env JAVA_OPTS="-Darcadedb.server.rootPassword=playwithdata \
  -Darcadedb.server.plugins=MongoDB:com.arcadedb.mongo.MongoDBProtocolPlugin" \
  arcadedata/arcadedb:latest
```

---

## Redis

### Overview

ArcadeDB Server supports a subset of the Redis protocol. This allows you to use standard Redis clients and tools (such as `redis-cli`) to interact with ArcadeDB for key/value operations, caching, session management, and persistent document storage.

The Redis plugin works in two modes:
- **Transient (RAM only)**: Key/value pairs stored in memory only (not persisted to database). Useful for user sessions and caching. Values are reset when the server restarts.
- **Persistent (Disk)**: Key/value pairs stored and retrieved from the ArcadeDB database. Can store and read documents, vertices, and edges.

### Maven Dependency (Embedded)

```xml
<dependency>
    <groupId>com.arcadedb</groupId>
    <artifactId>arcadedb-redisw</artifactId>
    <version>26.2.1</version>
</dependency>
```

### Installation

Enable the Redis plugin via `server.plugins`:

```bash
~/arcadedb $ bin/server.sh -Darcadedb.server.plugins\
  ="Redis:com.arcadedb.redis.RedisProtocolPlugin"
```

#### Docker Example (port 6379 is the default Redis port)

```bash
docker run --rm -p 2480:2480 -p 2424:2424 -p 6379:6379 \
  --env JAVA_OPTS="-Darcadedb.server.rootPassword=playwithdata \
  -Darcadedb.server.plugins=Redis:com.arcadedb.redis.RedisProtocolPlugin" \
  arcadedata/arcadedb:latest
```

### Settings

| Setting                  | Default     | Description                                      |
|--------------------------|-------------|--------------------------------------------------|
| `arcadedb.redis.host`    | `0.0.0.0`  | Host where the Redis protocol is listening        |
| `arcadedb.redis.port`    | `6379`      | Port where the Redis protocol is listening        |

### Transient (RAM Only) Commands

These commands do not take a bucket as a parameter. They work on a shared, thread-safe hashmap in memory. All stored values are reset when the server restarts.

| Command        | Description                                                        |
|----------------|-------------------------------------------------------------------|
| `GET`          | Return the value associated with a key                             |
| `SET`          | Set a value associated with a key                                  |
| `DEL`          | Delete a key (alias: `GETDEL` removes and returns the value)       |
| `EXISTS`       | Check if key exists                                                |
| `INCR`         | Increment a value by 1                                             |
| `INCRBY`       | Increment a value by a specific amount (64-bit precision)          |
| `INCRBYFLOAT`  | Increment a value by a float amount (64-bit precision)             |
| `DECR`         | Decrement a value by 1                                             |
| `DECRBY`       | Decrement a value by a specific amount (64-bit precision)          |

### Persistent (Disk) Commands

Persistent commands act on persistent buckets in the database. Records (documents, vertices, and edges) are always in the form of JSON embedded in strings. The bucket name is mapped as: database name first, then type, the index or the record's RID, based on the use case. An index must exist on the property you use to retrieve the document, otherwise an error is returned.

#### Setup Example

Create a schema with indexes for persistent Redis access:

```sql
CREATE DOCUMENT TYPE Account;

CREATE PROPERTY Account.id LONG;
CREATE INDEX ON Account (id) UNIQUE;

CREATE PROPERTY Account.email STRING;
CREATE INDEX ON Account (email) UNIQUE;

CREATE PROPERTY Account.firstName STRING;
CREATE PROPERTY Account.lastName STRING;
CREATE INDEX ON Account (firstName,lastName) UNIQUE;
```

> You can run Redis commands using the `redis-cli` tool (under Debian/Ubuntu this is part of the `redis-tools` package).

#### HSET - Create or Update Records

Create a new document with the Redis protocol using `HSET`:

```
HSET MyDatabase.Account
  "{'id':123,'email':'jay.miner@commodore.com','firstName':'Jay','lastName':'Miner'}"
```

#### HGET - Retrieve Records

Retrieve by RID (O(1) complexity):

```
HGET MyDatabase "#1:0"
```

Response:
```
"{'@rid':'#1:0','@type':'Account','id':123,'email':'jay.miner@commodore.com','firstName':'Jay','lastName':'Miner'}"
```

Retrieve by indexed property `id` (O(logN) complexity):

```
HGET MyDatabase.Account[id] 123
```

Retrieve by indexed property `email` (O(logN) complexity):

```
HGET MyDatabase.Account[email] "jay.miner@commodore.com"
```

Retrieve by composite key `firstName` and `lastName` (O(logN) complexity):

```
HGET MyDatabase.Account[firstName,lastName] "[\"Jay\",\"Miner\"]"
```

#### HMGET - Retrieve Multiple Records

Retrieve multiple records by RID in one call:

```
HMGET MyDatabase "#1:0" "#1:1" "#1:2"
```

Retrieve multiple records by indexed key:

```
HMGET MyDatabase.Account[id] 123 232 12
```

#### HDEL - Delete Records

Delete a record by indexed property:

```
HDEL MyDatabase.Account[email] "jay.miner@commodore.com"
```

#### Persistent Command Reference

| Command   | Description                                                              |
|-----------|-------------------------------------------------------------------------|
| `HGET`    | Retrieve a record by a key, a composite key, or record's RID             |
| `HMGET`   | Retrieve multiple records by a key, a composite key, or record's RID     |
| `HSET`    | Create and update one or more records by a key, composite key, or RID    |
| `HDEL`    | Delete one or more records by a key, composite key, or record's RID      |
| `HEXISTS` | Check if a key exists                                                    |

#### Miscellaneous Commands

| Command | Description                                              |
|---------|----------------------------------------------------------|
| `PING`  | Returns its argument (for testing server readiness or latency) |

### Redis via HTTP API

In addition to the native Redis wire protocol, ArcadeDB also supports Redis commands as a query language via the HTTP API. This allows you to execute Redis commands through the standard ArcadeDB HTTP `/command` endpoint by specifying `language=redis`.

This is useful when:
- You want to use Redis commands without a Redis client library
- You need to integrate Redis operations into existing HTTP-based workflows
- You want to combine Redis commands with other ArcadeDB query languages in the same application

#### Basic Usage

Send a POST request to the `/command` endpoint with `language` set to `redis`:

```bash
curl -X POST "http://localhost:2480/api/v1/command/MyDatabase" \
  -H "Content-Type: application/json" \
  -u root:password \
  -d '{"language": "redis", "command": "PING"}'
```

Response:

```json
{"result": [{"value": "PONG"}]}
```

#### HTTP Examples

```bash
# SET a transient value
curl -X POST "http://localhost:2480/api/v1/command/MyDatabase" \
  -H "Content-Type: application/json" \
  -u root:password \
  -d '{"language": "redis", "command": "SET mykey myvalue"}'

# GET a transient value
curl -X POST "http://localhost:2480/api/v1/command/MyDatabase" \
  -H "Content-Type: application/json" \
  -u root:password \
  -d '{"language": "redis", "command": "GET mykey"}'

# Store a persistent document
curl -X POST "http://localhost:2480/api/v1/command/MyDatabase" \
  -H "Content-Type: application/json" \
  -u root:password \
  -d '{"language": "redis", "command": "HSET Account {\"id\":1,\"name\":\"John\",\"age\":30}"}'

# Retrieve by index
curl -X POST "http://localhost:2480/api/v1/command/MyDatabase" \
  -H "Content-Type: application/json" \
  -u root:password \
  -d '{"language": "redis", "command": "HGET Account[id] 1"}'
```

#### Batch Commands

Execute multiple commands in a single request by separating them with newlines. Comments are supported using `#` or `//` prefixes:

```bash
curl -X POST "http://localhost:2480/api/v1/command/MyDatabase" \
  -H "Content-Type: application/json" \
  -u root:password \
  -d '{"language": "redis", "command": "SET key1 value1\nSET key2 value2\nGET key1\nGET key2"}'
```

#### Transactions with MULTI/EXEC

Redis transactions are supported using `MULTI` and `EXEC`. Commands between them are queued and executed atomically. Use `DISCARD` to abort a transaction:

```bash
curl -X POST "http://localhost:2480/api/v1/command/MyDatabase" \
  -H "Content-Type: application/json" \
  -u root:password \
  -d '{"language": "redis", "command": "MULTI\nSET tx1 value1\nSET tx2 value2\nINCR counter\nEXEC"}'
```

### Programmatic Access (Java)

You can use Redis commands directly from Java code via the query engine:

```java
// PING
try (ResultSet rs = database.query("redis", "PING")) {
    Result result = rs.next();
    String value = result.getProperty("value"); // Returns "PONG"
}

// SET and GET
database.command("redis", "SET mykey myvalue");
try (ResultSet rs = database.query("redis", "GET mykey")) {
    String value = rs.next().getProperty("value"); // Returns "myvalue"
}
```

---

## Choosing the Right Query Language

| Scenario                                       | Recommended Language |
|------------------------------------------------|---------------------|
| Complex graph traversals with pattern matching | OpenCypher or SQL   |
| Simple graph traversals, TinkerPop ecosystem   | Gremlin             |
| Schema-driven API with mixed query backends    | GraphQL             |
| Migrating from MongoDB                         | MongoDB QL          |
| Session management, caching, K/V operations    | Redis               |
| Complex joins, aggregations, analytics         | SQL                 |

> **Performance Note**: For complex graph queries, ArcadeDB's native SQL and OpenCypher implementations will generally outperform Gremlin, which may fall back to full scans for traversals involving `select()` and `where()`. Use SQL or OpenCypher when performance matters most.
