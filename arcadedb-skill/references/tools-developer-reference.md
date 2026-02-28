# ArcadeDB Tools & Developer Reference

## Table of Contents
1. [Studio (Web UI)](#studio-web-ui)
2. [Console (CLI)](#console-cli)
3. [Importer](#importer)
4. [Embedded Server](#embedded-server)
5. [User Functions](#user-functions)
6. [Triggers](#triggers)

---

## Studio (Web UI)

Access ArcadeDB Studio by pointing a web browser to the server host and port. The server must be in `development` or `test` mode. Default URL: `http://localhost:2480`.

### Main Menu

The vertical tab on the left provides navigation:

- **Query** - Execute queries and commands against the database
- **Database** - View database info, schema, and manage databases
- **Server** - Monitor server metrics, events, settings, and backup configuration
- **Cluster** - High availability cluster status and management (when HA is enabled)
- **API** - Public HTTP API documentation exposed on the current server
- **Info** - Quick reference to online documentation

### Command Panel

Execute queries by selecting a language first. Default is SQL.

**Supported languages:**
- **SQL** - For any models, including graphs and documents
- **SQL Script** - Multiple commands/queries in a batch
- **Apache Tinkerpop Gremlin** - For graphs only
- **Open Cypher** - For graphs only
- **MongoDB** - For documents only
- **GraphQL** - For structured queries

The command text area adjusts syntax highlighting based on the selected language. Results appear in the Command Result area as a **Graph**, **Table**, or **JSON** panel.

### Database Panel

Shows information about the selected database and its schema, with common management operations.

**Key elements:**
- **Server Version** - The version of ArcadeDB in use
- **User** - The logged-in user; the database list is filtered by user permissions. Use the `admin` user to access all databases
- **Selected Database** - Click to switch between available databases
- **Database Commands:**
  - **Create** - Create a new database by entering its name in the popup
  - **Drop** - Drop the current database (irreversible)
  - **Backup** - Execute an immediate backup; stored in `backups/<db-name>/<db-name>-backup-<timestamp>.zip`
  - **Reset** - Drop all data and schema from the database
- **Types** - Vertical tab to select and inspect types, showing configured indexes and properties
- **Actions** - Quick actions against the selected type:
  - **Display the first 30 records** of the selected type
  - **Display the first 30 records with all the vertices connected** - Shows records and their direct neighbors as a graph

**Additional tabs:**
- **Database Schema** - View and manage vertex, edge, and document types
- **Database Settings** - View and modify database configuration settings
- **Backup** - View backup configuration and available backups for the current database

### Database Backup Tab

Displays backup information specific to the selected database:

- **Backup Configuration** - Current backup schedule and retention settings
- **Statistics** - Total number of backups and disk space used
- **Available Backups** - List of all backup files with timestamps and file sizes

Actions: **Refresh** (reload backup list), **Backup Now** (trigger immediate backup), view backup history.

### Graph Panel

Interactive graph visualization of query results.

**Context menu** (hold selection on a node to show):
- Load incoming vertices
- Load outgoing vertices
- Load both incoming and outgoing vertices
- Hide the current node (remove from graph)

**Node Panel** (right side, when a node is selected):
- Element type: `Node` or `Edge`
- Record ID (RID)
- Type
- Properties table
- Actions (quick actions against the selected node)
- Layout

**Node Layout:**
Click the `+` button to expand and customize the layout panel per node type. Change the label to an attribute that represents the node (e.g., `title` for Movie, `name` for Person).

**Export/Import Settings:**
Click `Export` > `Settings` to download style settings. Click `Import` > `Settings` to restore them. Share setting files with collaborators to maintain consistent graph visualization.

**Selection operations:**
- **Direct Neighbors** - Select nodes directly connected to the selected ones (`Select > Direct Neighbors`)
- **Orphan Vertices** - Select nodes not connected to any other node (`Select > Orphan Vertices`)
- **Invert Node Selection** - Invert the current selection (`Select > Invert Node Selection`)
- **Shortest Path** - Select 2 nodes, click `Select > Shortest Path`. Uses Dijkstra algorithm with fixed weight 1 per node

### Table Panel

Renders result sets as a table. Vertices, edges, and documents are flattened into rows. The `@in` column shows incoming edge count, `@out` shows outgoing edge count.

- Click on a **RecordID (RID)** in the first column to display the record in the graph view
- **Search** input field for real-time filtering on the current result set
- Pagination controls at the bottom of the table

**Export formats:**
- **Copy** - Copy to clipboard (CTRL+V / CMD+V)
- **Excel** - Microsoft Excel format
- **CSV** - Comma Separated Values
- **PDF** - PDF format
- **Print** - Print all pages

### JSON Panel

Displays the raw JSON result returned from the HTTP/JSON API. Press `Copy to clipboard` to copy the entire content.

### API Panel

Contains the description of the public HTTP API exposed on the current server.

### Swagger-UI

Available at `/api/v1/docs`. Default URL: `http://localhost:2480/api/v1/docs`.

Provides interactive API documentation with the ability to explore and test HTTP requests. Sections include Server, Health, Database, Query, Command, Transaction, and Schemas.

### Server Panel

Provides monitoring and administration capabilities.

**Tabs:**
- **Summary** - Overview of server resources (CPU, RAM, disk usage) and request metrics
- **Sessions** - Active authentication sessions on the server
- **Events** - Server event log with filtering options
- **Metrics** - Detailed server metrics and statistics
- **Server Settings** - View server configuration settings
- **Backup** - Configure automatic backup scheduling

### Backup Configuration Tab

Configure the automatic backup scheduler in the Server Panel.

**General Settings:**
- **Auto-Backup Enabled** - Enable or disable automatic backups
- **Backup Directory** - Directory where backups are stored (relative to server root)
- **Run On Server** - In HA clusters, which server runs backups (`$leader` or `*`)

**Schedule:**
- **Schedule Type** - Frequency-based or CRON-based scheduling
- **Frequency** - Minutes between backups (frequency-based)
- **CRON Expression** - Cron expression for precise scheduling (CRON-based)
- **Time Window** - Optional time window to restrict when backups can run

**Retention Policy:**
- **Max Backup Files** - Maximum number of backup files to keep per database
- **Use Tiered Retention** - Enable grandfathering backup rotation
- **Tiered Settings** - Number of hourly, daily, weekly, monthly, and yearly backups to retain

Configuration is stored in `config/backup.json`. After saving a new configuration for the first time, a server restart is required to activate the auto-backup scheduler plugin.

---

## Console (CLI)

### Launch

```bash
# Linux/MacOS
$ bin/console.sh

# Windows
> bin\console.bat

# With custom database directory
$ ./bin/console.sh -Darcadedb.server.databaseDirectory=/databases
```

### Command-Line Flags

| Flag | Description |
|------|-------------|
| `-D` | Pass settings (e.g., `-Darcadedb.server.databaseDirectory=/databases`) |
| `-b` | Batch mode - exits after executing all commands passed via command-line |
| `-fae` | Fail-at-end - continue executing if error occurs during batch (normally breaks at error) |

### Direct Commands

These commands are processed directly by the console (not evaluated by the query engine):

| Category | Commands |
|----------|----------|
| **Console** | `HELP` / `?`, `QUIT` / `EXIT`, `PWD` |
| **Server** | `CONNECT`, `CLOSE`, `LIST DATABASES` |
| **User** | `CREATE USER`, `DROP USER` |
| **Database** | `CREATE DATABASE`, `DROP DATABASE`, `CHECK DATABASE` |
| **Schema** | `INFO TYPES`, `INFO TYPE`, `INFO TRANSACTION` |
| **Transaction** | `BEGIN`, `COMMIT`, `ROLLBACK` |
| **Scripting** | `SET LANGUAGE`, `LOAD`, `--` (Comment) |

### Tutorial: First Database

```
-- Create a local database
> CREATE DATABASE mydb

-- Or create on a remote server
> CREATE DATABASE remote:localhost/mydb root arcadedb-password

-- Connect to it
> CONNECT mydb

-- Or connect to remote
> CONNECT remote:localhost/mydb root arcadedb-password

-- Create a user with access
> CREATE USER albert IDENTIFIED BY einstein GRANT CONNECT TO mydb

-- Create a document type
{mydb}> CREATE DOCUMENT TYPE Profile

-- Check types
{mydb}> INFO TYPES

-- Insert a record
{mydb}> INSERT INTO Profile SET name = 'Jay', lastName = 'Miner'

-- Query records
{mydb}> SELECT FROM Profile

-- Persist changes (auto-committed on exit/quit)
{mydb}> COMMIT
```

### Settings

```
> SET <setting> = <value>
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `asyncmode` | boolean | false | Asynchronous mode |
| `expandresultset` | boolean | false | Expand result set display |
| `language` | string | sql | Query language (`sql`, `sqlscript`, `cypher`, `gremlin`, `mongo`, `graphql`) |
| `limit` | integer | 20 | Max results to display |
| `maxmultivalueentries` | integer | 10 | Max multi-value entries |
| `maxwidth` | integer | 150 | Max column width |
| `transactionbatchsize` | integer | 0 | Transaction batch size |
| `verbose` | integer | 3 | Verbosity level |

### Scripting

Run local SQL scripts:

```bash
# Via batch mode
$ bin/console.sh -b "LOAD myscript.sql"

# Via inline commands
$ bin/console.sh "CREATE DATABASE test; CREATE DOCUMENT TYPE doc; BACKUP DATABASE; exit;"
```

Wrap scripts in transactions for write operations:

```
BEGIN; LOAD myscript.sql; COMMIT;
```

### Console-Server Interaction

The console cannot access a database via local connection when a server is running (the server locks databases). Connect via remote connection instead:

```
> CONNECT remote:localhost/mydb root arcadedb-password
```

### Line Processing Rules

- Commands and queries must be on a single line
- Use `\` for line breaks
- Use `--` for comments (single-line only)
- Multi-line comment syntax `/* ... */` is NOT supported

---

## Importer

ArcadeDB provides ETL capabilities for importing datasets in multiple formats.

### Supported Formats

- OrientDB database export
- Neo4j database export
- GraphML database export
- GraphSON database export
- Generic XML files
- Generic JSON files
- Generic JSONL files
- Generic CSV files
- Generic RDF files

### Compression and Locations

**File types:** Plain text, ZIP (first file is read), GZip

**Source locations:**
- **Local** - File path or `file://` prefix
- **Remote** - `http://` or `https://` URLs
- **Classpath** - `classpath://` prefix

### Usage

**Console (SQL IMPORT DATABASE command):**

```sql
IMPORT DATABASE <url> [ WITH ( <setting-name> = <setting-value> [,] )* ]
```

**Java API:**

```java
com.arcadedb.integration.importer.Importer
```

### Import Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `url` | | URL of the file to import |
| `database` | `./databases/imported` | Path of the final imported database |
| `forceDatabaseCreate` | false | Create database brand new at every import |
| `wal` | false | Use WAL (journal) for importing; slower but safer |
| `commitEvery` | 5000 | Create transactions that commit every X records |
| `parallel` | | Number of parallel threads |
| `typeIdProperty` | | Property to use as type identifier |
| `typeIdType` | | Type of the type identifier |
| `typeIdUnique` | | Whether type identifier must be unique |
| `trimEdges` | | Trim edges during import |

### Example Imports

**Freebase RDF dataset:**

```
> CREATE DATABASE FreeBase
{FreeBase}> IMPORT DATABASE http://commondatastorage.googleapis.com/freebase-public/rdf/freebase-rdf-latest.gz
```

**Discogs XML dataset from remote S3:**

```
> IMPORT DATABASE https://discogs-data.s3-us-west-2.amazonaws.com/data/2018/discogs_20180901_releases.xml.gz
```

**NYC Taxi CSV dataset:**

```
> IMPORT DATABASE file:///personal/Downloads/data-society-uber-pickups-in-nyc/original/uber-raw-data-april-15.csv/uber-raw-data-april-15.csv
```

**With settings:**

```
> IMPORT DATABASE file:///import/file.csv WITH forceDatabaseCreate = true, commitEvery = 100
```

### See Also

- SQL Import Database command
- JSON Importer
- Neo4j Importer
- OrientDB Importer

---

## Embedded Server

### Overview: The "ArcadeDB Box"

Embedding the server in your JVM provides all the benefits of working in embedded mode with ArcadeDB (zero cost for network transport and marshalling) while still making the database accessible from the outside via Studio, remote API, Postgres, REDIS, MongoDB, and Gremlin drivers.

### Maven Dependencies

**Server only (no Studio UI):**

```xml
<dependency>
    <groupId>com.arcadedb</groupId>
    <artifactId>arcadedb-server</artifactId>
    <version>26.2.1</version>
</dependency>
```

**With Studio UI:**

```xml
<dependency>
    <groupId>com.arcadedb</groupId>
    <artifactId>arcadedb-studio</artifactId>
    <version>26.2.1</version>
</dependency>
```

The `arcadedb-server` dependency depends on `arcadedb-network-<version>.jar`. With Maven or Gradle, it is imported automatically; otherwise add `arcadedb-network` to your classpath.

The `arcadedb-server` dependency alone starts the server and HTTP URL but returns "Not Found" if you access Studio. Add `arcadedb-studio` to enable the web UI.

### Java 17 Notes

Java 17 packages are available through GitHub packages. Add the GitHub package repository to your `pom.xml`:

```xml
<repositories>
    <repository>
        <id>github</id>
        <url>https://maven.pkg.github.com/ArcadeData/arcadedb</url>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>
</repositories>
```

Then use the `-java17` suffix on the version:

```xml
<dependency>
    <groupId>com.arcadedb</groupId>
    <artifactId>arcadedb-server</artifactId>
    <version>26.2.1-java17</version>
</dependency>
```

### Starting the Server

Create with an empty configuration (all defaults apply):

```java
ContextConfiguration config = new ContextConfiguration();
ArcadeDBServer server = new ArcadeDBServer(config);
server.start();
```

### Distributed Configuration

Set HA settings in the `ContextConfiguration` before starting:

```java
config.setValue(GlobalConfiguration.HA_SERVER_LIST,
    "192.168.10.1,192.168.10.2,192.168.10.3");
config.setValue(GlobalConfiguration.HA_REPLICATION_INCOMING_HOST, "0.0.0.0");
config.setValue(GlobalConfiguration.HA_ENABLED, true);
```

### Getting a Database

Always get the database instance from the server when embedding. Do not use `DatabaseFactory` directly, as it would attempt to lock a database already locked by the server.

```java
// Get existing database
Database database = server.getDatabase(<URL>);

// Get or create
Database database = server.getOrCreateDatabase(<URL>);
```

### Custom HTTP Commands

Create a `ServerPlugin` and implement the `registerAPI` method. Example for the HTTP POST API `/myapi/test`:

```java
package com.yourpackage;

public class MyTest implements ServerPlugin {
  // ...

  @Override
  public void registerAPI(HttpServer httpServer, final PathHandler routes) {
    routes.addPrefixPath("/myapi",
        Handlers.routing()
            .get("/account/{id}", new RetrieveAccount(this))
            .post("/test/{name}", new MyTestAPI(this))
    );
  }
}
```

**Test handler extending `DatabaseAbstractHandler`:**

```java
public class MyTestAPI extends DatabaseAbstractHandler {

  public MyTestAPI(final HttpServer httpServer) {
    super(httpServer);
  }

  @Override
  public void execute(final HttpServerExchange exchange, ServerSecurityUser user,
                      final Database database) throws IOException {
    final Deque<String> namePar = exchange.getQueryParameters().get("name");
    if (namePar == null || namePar.isEmpty()) {
      exchange.setStatusCode(400);
      exchange.getResponseSender().send("{ \"error\" : \"name is null\"}");
      return;
    }

    final String name = namePar.getFirst();
    // DO SOMETHING MEANINGFUL HERE
    exchange.setStatusCode(204);
    exchange.getResponseSender().send("");
  }
}
```

**Register the plugin** via the `arcadedb.server.plugins` setting:

```bash
$ java ... -Darcadedb.server.plugins=MyPlugin:com.yourpackage.MyPlugin ...
```

### HTTPS Configuration

Enable HTTPS before the server starts:

```java
configuration.setValue(GlobalConfiguration.NETWORK_USE_SSL, true);
configuration.setValue(GlobalConfiguration.NETWORK_SSL_KEYSTORE,
    "src/test/resources/master.jks");
configuration.setValue(GlobalConfiguration.NETWORK_SSL_KEYSTORE_PASSWORD,
    "keypassword");
configuration.setValue(GlobalConfiguration.NETWORK_SSL_TRUSTSTORE,
    "src/test/resources/master.jks");
configuration.setValue(GlobalConfiguration.NETWORK_SSL_TRUSTSTORE_PASSWORD,
    "storepassword");
```

**SSL settings:**
- `NETWORK_USE_SSL` - Enable SSL for the HTTP Server
- `NETWORK_SSL_KEYSTORE` - Path to the keystore file
- `NETWORK_SSL_KEYSTORE_PASSWORD` - Keystore password
- `NETWORK_SSL_TRUSTSTORE` - Path to the truststore file
- `NETWORK_SSL_TRUSTSTORE_PASSWORD` - Truststore password

**Default HTTPS port range:** 2490-2499 (`GlobalConfiguration.SERVER_HTTPS_INCOMING_PORT`). If HTTP or HTTPS ports are already used, the next available ports are used automatically (e.g., 2481 for HTTP and 2491 for HTTPS).

### Replication Between Boxes

You can replicate databases across multiple embedded server instances ("boxes") for true high availability. Each box runs the full ArcadeDB server inside the application JVM, and the HA module handles replication between them.

---

## User Functions

ArcadeDB supports user-defined functions written in SQL, JavaScript, OpenCypher, or Java. Functions defined in any language are callable from ALL query languages (SQL, Cypher, Gremlin, GraphQL, MongoDB).

### Universal Function Registry

Functions are stored in a global registry and accessible from any query language:

```sql
-- Define once in SQL
DEFINE FUNCTION math.sum "SELECT :a + :b" PARAMETERS [a,b] LANGUAGE sql;

-- Call from SQL
SELECT 'math.sum'(3, 5) AS result;

-- Call from Cypher
RETURN math.sum(3, 5) AS result;

-- Call from Gremlin, GraphQL, MongoDB queries via SQL bridge
```

### Defining User Functions

```sql
DEFINE FUNCTION <library>.<name> "<body>" [PARAMETERS [<parameter>,*]] LANGUAGE <language>
```

- `<library>` - A namespace to group related functions
- `<name>` - The function's name
- `<body>` - The function's body as a string in the chosen language's syntax
- `[<parameter>,*]` - Parameter identifiers used in the function body. Omit the `PARAMETERS []` block for functions without parameters
- `<language>` - One of: `sql`, `js` (JavaScript), `opencypher` (or `cypher`), or a Java class name

### SQL Language Functions

SQL functions execute SQL queries and return the result of the first row's projection. The return value is determined by the projection named `'result'` or the first column if only one column is returned.

```sql
-- Simple arithmetic function
DEFINE FUNCTION math.add "SELECT :a + :b AS result" PARAMETERS [a,b] LANGUAGE sql;
SELECT 'math.add'(10, 5) AS sum;  -- Returns 15

-- Function returning a string
DEFINE FUNCTION the.answer 'SELECT "forty-two" AS result' LANGUAGE sql;
SELECT 'the.answer'();  -- Returns "forty-two"

-- Function with no parameters
DEFINE FUNCTION utils.currentYear
    'SELECT date_format(sysdate(), "yyyy") AS result'
    LANGUAGE sql;
SELECT 'utils.currentYear'();
```

### JavaScript Language Functions

JavaScript functions use GraalVM's JavaScript engine for complex calculations and data transformations.

```sql
-- Fused multiply-add function
DEFINE FUNCTION my.fma 'return a + b * c' PARAMETERS [a,b,c] LANGUAGE js;
SELECT 'my.fma'(1, 2, 3);  -- Returns 7

-- String manipulation
DEFINE FUNCTION text.greet 'return "Hello, " + name + "!"' PARAMETERS [name]
    LANGUAGE js;
SELECT 'text.greet'('World');  -- Returns "Hello, World!"

-- Complex JSON processing
DEFINE FUNCTION json.extract 'return JSON.parse(data).users.length' PARAMETERS [data]
    LANGUAGE js;
SELECT 'json.extract'('{"users":[1,2,3]}');  -- Returns 3
```

### OpenCypher Language Functions

OpenCypher functions execute Cypher queries, ideal for graph traversals and pattern matching. Use either `opencypher` or `cypher` as the language identifier.

```sql
-- Simple calculation using Cypher
DEFINE FUNCTION cypher.double "RETURN $x * 2" PARAMETERS [x] LANGUAGE opencypher;
SELECT 'cypher.double'(5);  -- Returns 10

-- Graph query function: count neighbors
DEFINE FUNCTION graph.countNeighbors
  "MATCH (n) WHERE id(n) = $nodeId MATCH (n)-[]->() RETURN count(*) AS cnt"
  PARAMETERS [nodeId] LANGUAGE cypher;

-- Use in SQL query
SELECT 'graph.countNeighbors'('#1:0') AS neighbors;

-- Use in Cypher query
MATCH (n:Person) RETURN n.name, graph.countNeighbors(id(n)) AS friendCount;

-- Complex graph pattern: find common friends
DEFINE FUNCTION graph.findCommonFriends
  "MATCH (a)-[:KNOWS]->(common)<-[:KNOWS]-(b)
   WHERE id(a) = $id1 AND id(b) = $id2
   RETURN count(DISTINCT common) AS cnt"
  PARAMETERS [id1, id2] LANGUAGE opencypher;
```

### Calling User Functions

Functions can be called from any supported query language:

| Language | Syntax | Example |
|----------|--------|---------|
| SQL | `'library.function'(args)` | `SELECT 'math.sum'(3, 5)` |
| Cypher | `library.function(args)` | `RETURN math.sum(3, 5)` |
| Cypher CALL | `CALL library.function(args) YIELD result` | `CALL math.sum(3, 5) YIELD result RETURN result` |
| Gremlin | Standard function call | Access via SQL bridge |
| GraphQL | In query fields | Access via SQL bridge |

### Cross-Language Function Calls

Define a function in one language and call it from any other:

```sql
-- Define in SQL
DEFINE FUNCTION sql.concat "SELECT :a || :b" PARAMETERS [a,b] LANGUAGE sql;

-- Define in JavaScript
DEFINE FUNCTION js.power "return Math.pow(x, y)" PARAMETERS [x,y] LANGUAGE js;

-- Define in Cypher
DEFINE FUNCTION cypher.shortestPath
  "MATCH path=shortestPath((a)-[*]-(b))
   WHERE id(a)=$from AND id(b)=$to
   RETURN length(path)"
  PARAMETERS [from, to] LANGUAGE opencypher;

-- Call all from Cypher
RETURN sql.concat('Hello', ' World') AS greeting,
       js.power(2, 10) AS power,
       cypher.shortestPath('#1:0', '#1:100') AS pathLen;

-- Call all from SQL
SELECT 'sql.concat'('foo', 'bar') AS str,
       'js.power'(3, 3) AS cube,
       'cypher.shortestPath'('#1:0', '#1:100') AS dist;
```

### Deleting User Functions

```sql
DELETE FUNCTION <library>.<name>
```

```sql
DELETE FUNCTION extra.tsum;
DELETE FUNCTION math.obsolete;
```

### Java Functions

Java functions provide maximum performance and access to the full Java ecosystem. Register them programmatically:

```java
SQLQueryEngine sqlEngine = (SQLQueryEngine) database.getQueryEngine("sql");

// Register 'bigger' function with fixed 2 parameters (MIN/MAX=2)
SQLEngine.getInstance().registerFunction("bigger",
    new SQLFunctionAbstract("bigger", 2, 2) {
      public String getSyntax() {
        return "bigger(<first>, <second>)";
      }

      public Object execute(Object[] iParameters) {
        if (iParameters[0] == null || iParameters[1] == null)
          return null;
        if (!(iParameters[0] instanceof Number) || !(iParameters[1] instanceof Number))
          return null;
        final double v1 = ((Number) iParameters[0]).doubleValue();
        final double v2 = ((Number) iParameters[1]).doubleValue();
        return Math.max(v1, v2);
      }

      public boolean aggregateResults() {
        return false;
      }
    });
```

Use from any query language:

```java
// From SQL
ResultSet result = database.command("sql",
    "SELECT FROM Account WHERE bigger(salary, 10) > 10");

// From Cypher
ResultSet result = database.query("opencypher",
    "MATCH (a:Account) WHERE bigger(a.salary, 10) > 10 RETURN a");
```

### Best Practices

1. **Choose the Right Language:**
   - SQL for data transformations and aggregations
   - JavaScript for complex business logic and calculations
   - OpenCypher for graph traversals and pattern matching
   - Java for maximum performance

2. **Namespace Organization:** Group related functions under the same library namespace (e.g., `math.sum`, `math.avg`, `utils.formatDate`, `graph.shortestPath`)

3. **Function Naming:** Use descriptive names that indicate what the function does

4. **Performance Considerations:**
   - User functions are evaluated for each row/result
   - For heavy operations, consider caching or materialized views
   - Java functions provide the best performance

5. **Error Handling:** Functions should handle null inputs gracefully

### Advantages Over Neo4j

Unlike Neo4j which only supports Java-based user-defined procedures requiring compilation and deployment:

- **Dynamic Definition** - Define functions at runtime without restart
- **4 Languages** - SQL, JavaScript, OpenCypher, and Java
- **Universal Access** - Call from any query language
- **No Compilation** - Interpreted languages work immediately
- **Easy Management** - Simple SQL commands to define/delete functions

---

## Triggers

> Available from version 26.2.1

ArcadeDB supports database triggers that automatically execute SQL statements, JavaScript code, or Java classes in response to record events. Triggers enable business logic, data validation, audit trails, and automated workflows directly in your database.

### Overview

A **trigger** is a named database object that automatically executes when specific events occur on records of a particular type.

**Key Features:**
- **Event-driven** - Fire automatically on CREATE, READ, UPDATE, or DELETE operations
- **Timing control** - Execute BEFORE or AFTER the event
- **Multi-language** - Choose between SQL statements, JavaScript code, or Java classes
- **High performance** - Java class triggers offer maximum performance with compiled code
- **Persistent** - Triggers are stored in the schema and survive database restarts
- **Type-specific** - Each trigger applies to a specific document/vertex/edge type

### Trigger Syntax

```sql
CREATE TRIGGER <name> <timing> <event> ON TYPE <type> EXECUTE <language> '<body>';
```

### Trigger Events

Triggers can respond to eight different combinations of timing and events:

| Event | Description |
|-------|-------------|
| `BEFORE CREATE` | Before a new record is created |
| `AFTER CREATE` | After a new record is created |
| `BEFORE READ` | Before a record is read from database |
| `AFTER READ` | After a record is read from database |
| `BEFORE UPDATE` | Before a record is modified |
| `AFTER UPDATE` | After a record is modified |
| `BEFORE DELETE` | Before a record is deleted |
| `AFTER DELETE` | After a record is deleted |

### SQL Triggers

SQL triggers execute SQL statements. They have access to the current record through context variables.

**Context Variables:**
- `$record` or `record` - The current record being operated on

**Example: Audit Trail**

```sql
-- Create the audit log type
CREATE DOCUMENT TYPE AuditLog;

-- Create trigger to log user creations
CREATE TRIGGER user_audit AFTER CREATE ON TYPE User
EXECUTE SQL 'INSERT INTO AuditLog SET action = "user_created",
              userName = $record.name,
              timestamp = sysdate()';

-- Now every time a User is created, an entry is automatically added:
INSERT INTO User SET name = 'Alice', email = 'alice@example.com';

-- Check the audit log
SELECT * FROM AuditLog;
-- Returns: {action: "user_created", userName: "Alice", timestamp: ...}
```

**Example: Auto-increment Counter**

```sql
-- Create a counter type
CREATE DOCUMENT TYPE Counter;
INSERT INTO Counter SET name = 'order_sequence', value = 1000;

-- Create trigger to auto-increment order numbers
CREATE TRIGGER order_number BEFORE CREATE ON TYPE Order
EXECUTE SQL 'UPDATE Counter SET value = value + 1
              WHERE name = "order_sequence";
              UPDATE $record SET orderNumber =
              (SELECT value FROM Counter WHERE name = "order_sequence")';
```

**Example: Cascade Updates**

```sql
-- Update all orders when customer email changes
CREATE TRIGGER customer_email_update AFTER UPDATE ON TYPE Customer
EXECUTE SQL 'UPDATE Order SET customerEmail = $record.email
              WHERE customerId = $record.@rid';
```

**Example: Data Validation**

```sql
-- Ensure product prices are positive
CREATE TRIGGER validate_price BEFORE CREATE ON TYPE Product
EXECUTE SQL 'SELECT FROM Product WHERE @this = $record AND price > 0';
-- If this SELECT statement returns no results, the trigger fails and the operation is aborted.
```

### JavaScript Triggers

JavaScript triggers offer more flexibility with conditional statements, loops, and calculations.

**Context Variables:**
- `record` or `$record` - The current record being operated on
- `oldRecord` or `$oldRecord` - The original record (for UPDATE operations), null otherwise
- `database` - The database instance

**Return Value:**
JavaScript triggers can return `false` to abort the operation (for BEFORE triggers only).

**Example: Data Validation**

```sql
CREATE TRIGGER validate_email BEFORE CREATE ON TYPE User
EXECUTE JAVASCRIPT 'if (!record.email || !record.email.includes("@")) {
  throw new Error("Invalid email address");
}';
```

**Example: Auto-populate Fields**

```sql
CREATE TRIGGER user_defaults BEFORE CREATE ON TYPE User
EXECUTE JAVASCRIPT '
  // Set creation timestamp
  record.createdAt = new Date();

  // Generate username from email if not provided
  if (!record.username && record.email) {
    record.username = record.email.split("@")[0];
  }

  // Set default role
  if (!record.role) {
    record.role = "user";
  }
';
```

**Example: Complex Business Logic (Discount Calculation)**

```sql
CREATE TRIGGER calculate_discount BEFORE CREATE ON TYPE Order
EXECUTE JAVASCRIPT '
  var total = record.total || 0;
  var discount = 0;

  // Apply discount based on order total
  if (total > 1000) {
    discount = 0.15;  // 15% discount
  } else if (total > 500) {
    discount = 0.10;  // 10% discount
  } else if (total > 100) {
    discount = 0.05;  // 5% discount
  }

  record.discountPercent = discount * 100;
  record.discountAmount = total * discount;
  record.finalTotal = total - (total * discount);
';
```

**Example: Conditional Abort**

```sql
CREATE TRIGGER prevent_weekend_orders BEFORE CREATE ON TYPE Order
EXECUTE JAVASCRIPT '
  var day = new Date().getDay();
  if (day === 0 || day === 6) {
    throw new Error("Orders cannot be placed on weekends");
  }
';
```

**Example: Audit with Details**

```sql
CREATE TRIGGER audit_update AFTER UPDATE ON TYPE Product
EXECUTE JAVASCRIPT '
  database.command("sql",
    "INSERT INTO AuditLog SET action = ?, productId = ?, productName = ?, timestamp = sysdate()",
    "product_updated", record.getIdentity().toString(), record.getString("name"));
';
```

### Java Triggers

Java triggers implement listener interfaces for maximum performance. Register them programmatically via the database events system.

**Available Listener Interfaces:**
- `RecordBeforeCreateListener`
- `RecordAfterCreateListener`
- `RecordBeforeReadListener`
- `RecordAfterReadListener`
- `RecordBeforeUpdateListener`
- `RecordAfterUpdateListener`
- `RecordBeforeDeleteListener`
- `RecordAfterDeleteListener`

**Registration:**

```java
database.getEvents().registerListener(new RecordBeforeCreateListener() {
    @Override
    public boolean onBeforeCreate(Record record) {
        // Validate or modify the record
        // Return false to abort the operation
        return true;
    }
});

database.getEvents().registerListener(new RecordAfterUpdateListener() {
    @Override
    public void onAfterUpdate(Record record) {
        // Perform post-update actions (audit logging, etc.)
    }
});
```

### Managing Triggers

**List all triggers:**

```sql
SELECT FROM schema:triggers
```

**Drop a trigger:**

```sql
DROP TRIGGER <name>
```

### Best Practices

1. **Use BEFORE for validation and modification** - Validate data and set defaults before the record is persisted
2. **Use AFTER for audit logging** - Log changes after they are committed
3. **Keep logic simple** - Complex logic in triggers can slow down write operations
4. **Handle errors gracefully** - Throw descriptive errors in BEFORE triggers for validation failures
5. **Choose the right language:**
   - **SQL** for simple operations (best performance for basic queries)
   - **JavaScript** for complex business logic (conditionals, loops, calculations)
   - **Java** for maximum performance (compiled code, full ecosystem access)

### Common Use Cases

| Use Case | Timing | Language | Description |
|----------|--------|----------|-------------|
| Audit trails | AFTER CREATE/UPDATE/DELETE | SQL or JS | Log all changes to an audit table |
| Data validation | BEFORE CREATE/UPDATE | JS | Validate email, check ranges, enforce rules |
| Auto-populated fields | BEFORE CREATE | JS | Set timestamps, generate usernames, defaults |
| Cascade updates | AFTER UPDATE | SQL | Propagate changes to related records |
| Computed fields | BEFORE CREATE/UPDATE | JS | Calculate totals, discounts, derived values |
| Sequence generation | BEFORE CREATE | SQL | Auto-increment counters and sequence numbers |
