# ArcadeDB Administration Guide Reference

## Table of Contents
1. [Installation](#installation)
2. [Server](#server)
3. [Changing Settings](#changing-settings)
4. [Backup and Restore](#backup-and-restore)
5. [Security](#security)
6. [High Availability](#high-availability)
7. [Docker](#docker)
8. [Kubernetes](#kubernetes)

---

## Installation

### Binary Installation

ArcadeDB is released for both Java 17 and Java 21.

**Installation Steps:**
1. Download the package from:
   - Java 21: [GitHub releases page](https://github.com/ArcadeData/arcadedb/releases) or [Maven central](https://mvnrepository.com)
   - Java 17: [Github packages page](https://github.com/ArcadeData/arcadedb/packages)

2. Unpack: `tar -xzf arcadedb-26.2.1.tar.gz`

3. Change into directory: `cd arcadedb-26.2.1`

4. Launch server:
   - Linux/MacOS: `bin/server.sh`
   - Windows: `bin\server.bat`

5. Access via:
   - Studio: `http://localhost:2480`
   - Console: `bin/console.sh` (Linux/MacOS) or `bin\console.bat` (Windows)

### Mac OS X Installation

1. Download latest release from [GitHub releases](https://github.com/ArcadeData/arcadedb/releases)
2. Extract to preferred location (e.g., `/usr/local/arcadedb`)
3. Add `bin` directory to PATH:
   ```bash
   echo 'export PATH="/usr/local/arcadedb/bin:$PATH"' >> ~/.zshrc
   source ~/.zshrc
   ```

### Windows via Scoop

```bash
scoop bucket add extras
scoop install arcadedb
```

Creates two commands: `arcadedb-console` and `arcadedb-server`

### Custom Package Builder

The **ArcadeDB Modular Distribution Builder** (`arcadedb-builder.sh`) creates custom packages with only needed modules.

**Prerequisites:**
- `curl` or `wget` - for downloading files
- `tar` - for extracting/creating archives
- `unzip` and `zip` - for ZIP archives
- `sha256sum` or `shasum` - for checksum verification
- `docker` (optional) - for Docker image generation

**Core Modules (always included):**
- `engine` - Database engine
- `server` - HTTP/REST API, clustering
- `network` - Network communication

**Optional Modules:**
| Module | Description |
|--------|-------------|
| console | Interactive database console |
| studio | Web-based administration interface |
| gremlin | Apache Tinkerpop Gremlin support |
| postgresw | PostgreSQL wire protocol compatibility |
| mongodbw | MongoDB wire protocol compatibility |
| redisw | Redis wire protocol compatibility |
| grpcw | gRPC wire protocol support |
| graphql | GraphQL API support |
| metrics | Prometheus metrics integration |

**Quick Start:**
```bash
# One-liner
curl -fsSL https://github.com/ArcadeData/arcadedb/releases/download/26.2.1/arcadedb-builder.sh | bash -s -- --version=26.2.1 --modules=gremlin,studio

# Interactive
curl -fsSL https://github.com/ArcadeData/arcadedb/releases/download/26.2.1/arcadedb-builder.sh
chmod +x arcadedb-builder.sh
./arcadedb-builder.sh
```

**Usage Examples:**

Minimal build (PostgreSQL protocol only):
```bash
./arcadedb-builder.sh --version=26.2.1 --modules=postgresw
```

Development build:
```bash
./arcadedb-builder.sh \
  --version=26.2.1 \
  --modules=console,gremlin,studio \
  --output-name=arcadedb-dev
```

Production build (no Studio):
```bash
./arcadedb-builder.sh \
  --version=26.2.1 \
  --modules=postgresw,metrics \
  --output-name=arcadedb-prod
```

CI/CD build:
```bash
./arcadedb-builder.sh \
  --version=26.2.1 \
  --modules=gremlin,studio \
  --quiet \
  --skip-docker \
  --output-dir=/tmp/builds
```

**Command-Line Options:**
| Option | Description |
|--------|-------------|
| `--version=VERSION` | ArcadeDB version to build (required for non-interactive mode) |
| `--modules=MODULES` | Comma-separated list of modules |
| `--output-name=NAME` | Custom name for distribution |
| `--output-dir=DIR` | Output directory (default: current directory) |
| `--local-repo=[PATH]` | Use local Maven repository or custom JAR directory |
| `--local-base=FILE` | Use local base distribution file |
| `--docker-tag=TAG` | Build Docker image with specified tag |
| `--skip-docker` | Skip Docker image build |
| `--dockerfile-only` | Only generate Dockerfile, don't build image |
| `--dry-run` | Show what would be done without executing |
| `-v, --verbose` | Enable verbose output |
| `-q, --quiet` | Suppress non-error output |
| `-h, --help` | Show help message |

**Output Files:**
- `{output-name}.zip` - Zip archive
- `{output-name}.tar.gz` - Compressed tarball
- Docker image with tag `{docker-tag}` (if not skipped)

### Docker Installation

**Java 21 (DockerHub):**
```bash
docker pull arcadedb/arcadedb:26.2.1
```

**Java 17 (GitHub Container Registry):**
```bash
docker pull ghcr.io/arcadedata/arcadedb:26.2.1-java17
```

---

## Server

### Starting the Server

**Command:**
```bash
$ bin/server.sh
```

On first run, you'll be prompted to set the root password. To skip this:

```bash
-Darcadedb.server.rootPassword=password
```

Or use a file-based approach:
```bash
-Darcadedb.server.rootPasswordPath=/run/secrets/root
```

**First Run Output:**
- Root password must be 8-256 characters
- Password hash (+salt) stored in `config/server-users.json`
- Server displays HTTP URL and port (default: 2480)
- Studio accessible at `http://192.168.1.102:2480`

### Server Startup Hint

To start server from different location:
```bash
export ARCADEDB_HOME=/path/to/arcadedb
bin/server.sh
```

### Server Modes

Server can run in three modes affecting Studio availability and logging:

| Mode | Studio | Logging |
|------|--------|---------|
| development | Yes | Detailed |
| test | Yes | Brief |
| production | No | Brief |

**Control:** `server.mode` setting (default: `development`)

```bash
bin/server.sh -Darcadedb.server.mode=production
```

### Create Default Database(s)

Create databases on startup using `server.defaultDatabases` setting:

```bash
bin/server.sh "-Darcadedb.server.defaultDatabases=Universe[albert:einstein]"
```

Creates database "Universe" with user "albert", password "einstein".

**Note:** Command line arguments containing `[]` must be quoted.

Multiple databases:
```bash
-Darcadedb.server.defaultDatabases=Multiverse[]
```

(Empty brackets = no users)

### Logging

**Log Location:** `./log/arcadedb.log.X` (X = 0-9, rotated)

**Configuration File:** `config/arcadedb-log.properties` (standard Java logging)

**Default Configuration:**
```
handlers = java.util.logging.ConsoleHandler, java.util.logging.FileHandler
.level = INFO
com.arcadedb.level = INFO

java.util.logging.ConsoleHandler.level = INFO
java.util.logging.ConsoleHandler.formatter = com.arcadedb.utility.AnsiLogFormatter

java.util.logging.FileHandler.level = INFO
java.util.logging.FileHandler.formatter = com.arcadedb.log.LogFormatter
java.util.logging.FileHandler.limit=100000000
java.util.logging.FileHandler.count=10
```

**Key Settings:**
- Line 1: Console and file loggers
- Line 2: Global log level (FINER, FINE, INFO, WARNING, SEVERE)
- Line 3: ArcadeDB-specific level
- Line 4: Console minimum level (below INFO discarded)
- Line 5: Console formatter (AnsiLogFormatter supports ANSI color codes)
- Line 6: File minimum level (below INFO discarded)
- Line 7: Formatter used for file
- Line 9: Max file size (100MB default)
- Line 10: Number of files to keep (10 default; oldest removed after 10th file)

**For Embedded Mode:**
```bash
java ... -Djava.util.logging.config.file=$ARCADEDB_HOME/config/arcadedb-log.properties ...
```

### Server Plugins (Extend The Server)

Create custom plugins by implementing `com.arcadedb.server.ServerPlugin`:

```java
public interface ServerPlugin {
  void startService();
  default void stopService() { }
  default void configure(ArcadeDBServer arcadeDBServer, ContextConfiguration configuration) { }
  default void registerAPI(final HttpServer httpServer, final PathHandler routes) { }
}
```

**Lifecycle Methods:**
- `startService()` - Called at server startup
- `stopService()` - Called when server shuts down
- `configure()` - Called during server initialization
- `registerAPI()` - Register custom HTTP commands

**Example Plugin:**
```java
package com.yourpackage;

public class MyPlugin implements ServerPlugin {
  @Override
  public void startService() {
    System.out.println( "Plugin started" );
  }

  @Override
  public void stopService() {
    System.out.println( "Plugin halted" );
  }

  @Override
  public default void configure(ArcadeDBServer arcadeDBServer, ContextConfiguration configuration) {
    System.out.println( "Plugin configured" );
  }

  @Override
  public default void registerAPI(final HttpServer httpServer, final PathHandler routes) {
    System.out.println( "Registering HTTP commands" );
  }
}
```

**Register Plugin:**
```bash
java ... -Darcadedb.server.plugins=MyPlugin:com.yourpackage.MyPlugin ...
```

Multiple plugins:
```bash
-Darcadedb.server.plugins=MyPlugin:com.yourpackage.MyPlugin,AnotherPlugin:com.yourpackage.AnotherPlugin
```

### Metrics

Enable metrics collection:
```bash
-Darcadedb.serverMetrics=true
```

Log metrics to output:
```bash
-Darcadedb.serverMetrics.logging=true
```

Publish as Prometheus metrics via HTTP:
```bash
-Darcadedb.server.plugins="Prometheus:com.arcadedb.metrics.prometheus.PrometheusMetricsPlugin"
```

Access metrics at: `http://localhost:2480/prometheus` (requires server credentials)

---

## Changing Settings

### Setting Syntax

All settings require `arcadedb.` prefix:

**Command Line:**
```bash
java -Darcadedb.dumpConfigAtStartup=true ...
```

**Java Code:**
```java
GlobalConfiguration.findByKey("arcadedb.dumpConfigAtStartup").setValue(true);
```

See [Appendix](#appendix) for all settings.

### Environment Variables

Server script parses environment variables:

| Variable | Description |
|----------|-------------|
| `JAVA_HOME` | JVM location |
| `JAVA_OPTS` | JVM options |
| `ARCADEDB_HOME` | ArcadeDB location |
| `ARCADEDB_PID` | `arcadedb.pid` file location |
| `ARCADEDB_OPTS_MEMORY` | JVM memory options |

**Default values:** See `server.sh` and `server.bat` scripts

### RAM Configuration

ArcadeDB uses dynamic RAM allocation by default. To limit:

```bash
export ARCADEDB_OPTS_MEMORY="-Xms800M -Xmx800M"
bin/server.sh
```

ArcadeDB runs with minimum 16M RAM. For <800M systems, use low-RAM profile:

```bash
export ARCADEDB_OPTS_MEMORY="-Xms128M -Xmx128M"
bin/server.sh -Darcadedb.profile=low-ram
```

**For memory latency issues (Linux):**
```bash
export ARCADEDB_OPTS_MEMORY="-XX:+PerfDisableSharedMem"
```

**Heap memory configuration:**
```bash
export ARCADEDB_OPTS_MEMORY="-XX:InitialRAMPercentage=50.0 -XX:MaxRAMPercentage=75.0"
```

See [JVM Memory Configuration](https://www.evanjones.ca/jvm-mmap-pause.html) for details.

---

## Backup and Restore

### Backup Overview

ArcadeDB supports non-stop backups (no write blocking):

1. **Manual Backup** - One-time via SQL or Studio UI
2. **Automatic Backup** - Scheduled with retention policies

### Manual Backup Configuration

**Options:**
- `-f <backup-file>` (string) - Filename or path to backup file to create
- `-d <database-path>` (string) - Local filesystem path to ArcadeDB database
- `-o` (boolean) - Overwrite if backup exists (default: false)

**Example:**
```bash
bin/backup.sh -f /backups/mydb.zip -d databases/mydb
```

### Cloud Backups

For container deployments requiring S3 backups (not directly supported):

1. **Intermediary Container** - Use `docker-s3-volume` to forward volume to S3
2. **S3 Filesystem Mount** - Mount S3 bucket via filesystem in userspace

### Automatic Backup Scheduler

ArcadeDB provides automatic backup scheduling as a server plugin. Activated when `backup.json` exists in server's `config` directory.

**Features:**
- Frequency-based scheduling (every N minutes)
- CRON-based scheduling (precise scheduling)
- Time windows (e.g., off-peak hours)
- Tiered retention (hourly, daily, weekly, monthly, yearly backups)
- Per-database configuration overrides
- HA cluster awareness
- Studio integration

**Configuration File:** `config/backup.json`

**Example Configuration:**
```json
{
  "version": 1,
  "enabled": true,
  "backupDirectory": "./backups",
  "defaults": {
    "enabled": true,
    "runOnServer": "$leader",
    "schedule": {
      "type": "frequency",
      "frequencyMinutes": 60,
      "timeWindow": {
        "start": "02:00",
        "end": "06:00"
      }
    },
    "retention": {
      "maxFiles": 10,
      "tiered": {
        "hourly": 24,
        "daily": 7,
        "weekly": 4,
        "monthly": 12,
        "yearly": 3
      }
    }
  },
  "databases": {
    "production": {
      "schedule": {
        "type": "cron",
        "expression": "0 0 2 * * ?"
      },
      "retention": {
        "maxFiles": 20
      }
    },
    "dev-db": {
      "enabled": false
    }
  }
}
```

#### Global Settings

| Setting | Default | Description |
|---------|---------|-------------|
| version | 1 | Configuration file version (for future compatibility) |
| enabled | true | Enable/disable auto-backup scheduler globally |
| backupDirectory | ./backups | Directory where backups are stored (must be relative path within server root) |

#### Default Settings

Applied to all databases unless overridden:

| Setting | Default | Description |
|---------|---------|-------------|
| defaults.enabled | true | Enable backups for databases by default |
| defaults.runOnServer | $leader | Which server runs backups (see HA Cluster Support) |

#### Schedule Settings

| Setting | Default | Description |
|---------|---------|-------------|
| schedule.type | frequency | Schedule type: `frequency` or `cron` |
| schedule.frequencyMinutes | 60 | Minutes between backups (when type is frequency) |
| schedule.expression | - | CRON expression (when type is cron) |
| schedule.timeWindow.start | - | Time window start (HH:mm format, e.g., "02:00") |
| schedule.timeWindow.end | - | Time window end (HH:mm format, e.g., "06:00") |

#### Retention Settings

| Setting | Default | Description |
|---------|---------|-------------|
| retention.maxFiles | 10 | Maximum backup files to keep per database |
| retention.tiered.hourly | 24 | Number of hourly backups to retain |
| retention.tiered.daily | 7 | Number of daily backups to retain |
| retention.tiered.weekly | 4 | Number of weekly backups to retain |
| retention.tiered.monthly | 12 | Number of monthly backups to retain |
| retention.tiered.yearly | 3 | Number of yearly backups to retain |

**Tiered Retention:** Scheduler keeps most recent backup for each time bucket. E.g., with `hourly: 24`, keeps one backup per hour for last 24 hours.

#### Per-Database Overrides

The `databases` section allows overriding any default setting for specific databases:

```json
{
  "databases": {
    "critical-db": {
      "schedule": {
        "type": "cron",
        "expression": "0 0 * * * ?"
      },
      "retention": {
        "maxFiles": 48,
        "tiered": {
          "hourly": 48,
          "daily": 14
        }
      }
    },
    "dev-db": {
      "enabled": false
    }
  }
}
```

#### HA Cluster Support

In high availability clusters, control which server performs backups:

| Value | Description |
|-------|-------------|
| $leader | Only the current leader node runs backups (default, recommended) |
| * | All nodes run backups independently |
| node-name | Only the specified node runs backups (e.g., "ArcadeDB_Replica1") |

Using `$leader` is recommended because:
- Ensures exactly one backup per schedule
- Automatically fails over if leader changes
- Avoids duplicate backups across cluster
- Prevents backup contention

#### Backup Directory Structure

```
backups/
├── database1/
│   ├── database1-backup-20250115-020000.zip
│   ├── database1-backup-20250116-020000.zip
│   └── database1-backup-20250117-020000.zip
└── database2/
    ├── database2-backup-20250115-020000.zip
    └── database2-backup-20250116-020000.zip
```

Filename format: `<database>-backup-<YYYYMMDD>-<HHMMSS>.zip`

#### Monitoring

Scheduler logs activity:
```
INFO [AutoBackupSchedulerPlugin] Auto-backup scheduler started. Backup directory: ./backups
INFO [AutoBackupSchedulerPlugin] Scheduled automatic backup for database 'production'
INFO [BackupScheduler] Starting scheduled backup for database 'production'
INFO [BackupScheduler] Backup completed: production-backup-20250115-020000.zip (15.2 MB)
INFO [BackupRetentionManager] Retained 10 backups, deleted 2 for database 'production'
```

Backup events recorded in Server Event Log (accessible from Studio Server panel).

#### Studio Integration

Configure and monitor backups through ArcadeDB Studio:
- View and modify backup configuration
- Trigger immediate backups for any database
- View list of available backups per database
- Monitor backup statistics and disk usage

#### Triggering Manual Backups

**HTTP API:**
```bash
curl -X POST "http://localhost:2480/api/v1/server" \
  -u root:password \
  -H "Content-Type: application/json" \
  -d '{"command": "trigger backup myDatabase"}'
```

**SQL Command:**
```sql
BACKUP DATABASE
```

#### Troubleshooting

**Scheduler Not Starting:**
1. Verify `backup.json` exists in `config` directory
2. Check JSON syntax is valid
3. Verify `enabled` field is `true`
4. Check server logs for error messages

**Backups Not Being Created:**
1. Check `backup` directory exists and is writable
2. Check `timeWindow` settings (backups only run within window)
3. In HA mode, verify `runOnServer` matches current server
4. Ensure sufficient disk space available

**Permission Denied Errors:**
- `backupDirectory` must be relative path (security restriction)
- Absolute paths rejected to prevent writing outside server directory

### Restore a Database

**Requirement:** Server must be restarted to access restored database

**Example:**
```bash
bin/restore.sh -f backups/mysb/mydb-backup-20210921-172750.zip -d databases/mydb
```

**Configuration:**
- `-f <backup-file>` (string) - Path to backup file to restore
- `-d <database-path>` (string) - Local filesystem path where to create database
- `-o` (boolean) - Overwrite if database exists (default: false)

---

## Security

### Security Policy

ArcadeDB manages security at server level only. In embedded mode, no default security unless implemented.

**Default Rules:**
- Username: 4-256 characters
- Password: 8-256 characters

**Custom Policy:**
```java
server.getSecurity().setCredentialsValidator( new DefaultCredentialsValidator(){
  @Override
  public void validateCredentials(final String userName, final String userPassword)
      throws ServerSecurityException {
    if( userPassword.equals("12345678") )
      throw new ServerSecurityException("Guess who was not attending security lesson!");
  }
});
```

### Users

**Storage:** `config/server-users.jsonl` (JSONL format - one JSON per line)

**On First Startup:** If user file empty, root user created interactively

**User File Example:**
```json
{"name":"root","password":"PBKDF2WithHmacSHA256v0Jg+qKVmcwlUqhEq1m2j80VntEkVjzkeq=82q2u4rjU1jgoLBX9sG0rV0b0h6aHo+RHlsQkxneGkM=","databases":{"*":["admin"]}}
```

**User Information Stored:**
- **name** (mandatory) - Username
- **password** - Hashed with PBKDF2 algorithm, configurable salt (default: 32). Always hashed; mandatory for all users except root (interactive prompt if missing)
- **databases** - Map of database name and groups. `"*"` wildcard means all databases; default group `"*"` has no access

**User-Group-Database Relationship:**
```
User → Group → Database
```

Example: "jay" user belongs to "BlogWriters" group in "Blog" database and "Editors" group in "Library" database:

```json
{
  "databases": {
    "Blog": ["BlogWriters"],
    "Library": ["Editors"]
  }
}
```

Users unassigned to groups get default group `"*"` (no access by default).

**User Management:**

Via Console:
```cypher
SHOW USERS
CREATE USER bob SET PASSWORD 'SecurePass123!'
ALTER USER bob SET PASSWORD 'NewPass456!'
DROP USER bob
```

Via Cypher (Server Mode):
```cypher
SHOW USERS
CREATE USER bob SET PASSWORD 'SecurePass123!'
ALTER USER bob SET PASSWORD 'NewPass456!'
DROP USER bob
```

See [Cypher User Management](link) for full reference.

### Groups

**Default Group `"*"`:**
```json
{
  "*": {
    "types": {
      "*": {
        "access": []
      }
    },
    "access": [],
    "readTimeout": -1,
    "resultSetLimit": -1
  }
}
```

**Where:**
- **types** - Map of type and access level. Wildcard `"*"` represents all types not defined
- **access** - Array of allowed permissions:
  - `updateSecurity` - Update security settings (create/modify/delete users, groups)
  - `updateSchema` - Update database schema (create/modify/drop buckets, types, indexes)
- **readTimeout** - Max timeout for read operations (-1 = no limits). Useful limiting users executing expensive queries
- **resultSetLimit** - Max number of entries in result set (-1 = no limits). Useful limiting users retrieving huge result sets

**Permission Profiling:**
- `createRecord` - Create new records
- `readRecord` - Read records
- `updateRecord` - Update records
- `deleteRecord` - Delete records

**Note:** Creating edges is 2 operations: (1) create edge record, (2) update vertices. Requires `createRecord` on edge type and `updateRecord` on vertex type.

**Example: Blog Writer Group:**
```json
{
  "types": {
    "*": {
      "access": []
    },
    "Blog": {
      "access": [
        "readRecord"
      ]
    },
    "Post": {
      "access": [
        "createRecord",
        "readRecord",
        "updateRecord",
        "deleteRecord"
      ]
    }
  }
}
```

**Admin Group (Default):**
```json
{
  "access": [
    "updateSecurity",
    "updateSchema"
  ],
  "resultSetLimit": -1,
  "readTimeout": -1,
  "types": {
    "*": {
      "access": [
        "createRecord",
        "readRecord",
        "updateRecord",
        "deleteRecord"
      ]
    }
  }
}
```

Allows executing any operation against security, schema, and records.

**Append-Only Group Example:**
```json
{
  "appendonly": {
    "access": [],
    "resultSetLimit": -1,
    "readTimeout": -1,
    "types": {
      "*": {
        "access": [
          "createRecord",
          "readRecord"
        ]
      }
    }
  }
}
```

Group members can read and create, but not update/delete. Useful for ledgers, block chains, data provenance.

**Edit Groups:** Use JSON editor to edit `config/server-groups.json`. Keep backup copy before editing; restore if errors.

### Handling Secrets

Passing secrets (passwords) via command line exposes them in:
- `htop` / `ps` output
- `docker compose top`
- Logs (if using `JAVA_OPTS` or `ARCADEDB_SETTINGS`)

**Examples of Problematic Approaches:**

```bash
-Darcadedb.server.rootPassword=password
# or
-Darcadedb.server.defaultDatabases=database[username:password]
```

Passwords readable in plain text in applications and Docker logs.

**Solutions:**

**File-Based (Recommended):**
```bash
-Darcadedb.server.rootPasswordPath=/run/secrets/root
# or
-Darcadedb.server.defaultDatabases=database[user1:'cat /path/to/user_secret']"
```

**Workaround Script** (for non-root secrets):
```bash
#!/bin/sh

export JAVA_TOOL_OPTIONS="-Darcadedb.server.defaultDatabases=database1[user1:'cat /path/to/user_secret']"

PID="$!"

unset JAVA_TOOL_OPTIONS

wait $PID
```

Steps:
1. Export secret via `JAVA_TOOL_OPTIONS` (passed to JVM before resolution)
2. Start server in background, save PID
3. Unset `JAVA_TOOL_OPTIONS` immediately (removes from environment)
4. Wait for server process (secrets not lingering in environment)

**Benefits:**
- Secrets passed directly to JVM without being resolved in shell first
- Immediate unset prevents secrets appearing in stderr logging
- Works with both `JAVA_OPTS` and `JDK_JAVA_OPTIONS`

### Gremlin Security

ArcadeDB provides two Gremlin execution engines with different security profiles.

**CRITICAL SECURITY NOTICE:** The Groovy Gremlin engine is NOT SECURE and should NOT be used in production environments or with untrusted users. Despite security restrictions, it remains vulnerable to remote code execution (RCE) attacks. Always use the Java engine for production deployments.

#### Execution Engines

**Java Engine (Default - Secure)**

Parses Gremlin queries using restricted grammar and generates bytecode.

Features:
- Only accepts valid Gremlin traversal syntax
- Cannot execute arbitrary Java code
- Blocks `Runtime.getRuntime().exec()` and similar attacks
- Recommended for ALL deployments
- Required for production and security-critical environments

**Groovy Engine (Legacy - INSECURE)**

Provides dynamic scripting capabilities but has critical security limitations:

Features:
- Supports Groovy-specific syntax and dynamic features
- Cannot be adequately secured - `SecureASTCustomizer` has known bypasses
- Vulnerable to: `Runtime.exec()`, `ProcessBuilder`, file system access, reflection
- Only use for development/testing in fully trusted environments
- Will display security warning at startup

#### Configuration

**Set Engine via Database Configuration:**
```sql
-- Secure (default)
ALTER DATABASE `arcadedb.gremlin.engine` 'java';

-- Legacy with security restrictions (use only if needed)
ALTER DATABASE `arcadedb.gremlin.engine` 'groovy';

-- Auto-detect (not recommended for production)
ALTER DATABASE `arcadedb.gremlin.engine` 'auto';
```

**Set Engine via JVM Startup Parameters:**
```bash
# Set to Java engine (secure)
java -Darcadedb.gremlin.engine=java ...

# Set to Groovy engine (use only if needed)
java -Darcadedb.gremlin.engine=groovy ...
```

#### Security Best Practices

1. **Use Java Engine: ALWAYS use `java` engine in production** - this is non-negotiable for security
2. **Never Use Groovy in Production** - the Groovy engine is vulnerable to RCE and cannot be secured
3. **Least Privilege** - grant Gremlin query permissions only to fully trusted administrators
4. **Input Validation** - never accept Gremlin queries from untrusted sources
5. **Audit Logging** - monitor all Gremlin query execution for suspicious patterns
6. **Network Isolation** - run ArcadeDB in isolated network segments
7. **Authentication Required** - always require authentication for Gremlin access

#### Groovy Engine Limitations

Despite attempts to restrict dangerous operations, the Groovy engine remains vulnerable to:

- **Process execution:** `Runtime.exec()`, `ProcessBuilder` - NOT BLOCKED
- **File system access:** `File`, `FileInputStream`, `Files`, `Paths` - NOT BLOCKED
- **Reflection:** `Method`, `Field`, `Constructor`, `Class.forName()` - NOT BLOCKED
- **Class loading:** `GroovyClassLoader`, `GroovyClassLoaderResource` - NOT BLOCKED
- **Network operations:** `URL`, `URLConnection`, `Socket` - NOT BLOCKED
- **System operations:** `System.exit()`, `System.load()` - NOT BLOCKED

The `SecureASTCustomizer` security restrictions have known bypasses and are insufficient for production use.

#### Migration from Groovy to Java Engine

If currently using Groovy engine:

1. **Test Your Queries** - run existing Gremlin queries with Java engine to verify compatibility
2. **Update Configuration** - change `arcadedb.gremlin.engine` setting to `'java'`
3. **Refactor if Needed** - if you have Groovy-specific syntax, refactor to standard Gremlin traversal syntax
4. **Verify Results** - ensure query results are consistent between engines

Most standard Gremlin queries work identically on both engines. The Java engine is faster and more secure.

**Example: RCE Vulnerability (Do NOT Use This)**
```
Runtime.getRuntime().exec("touch /tmp/owned")
new File("/etc/passwd").text
```

The Java engine (default) is NOT vulnerable to these attacks.

---

## High Availability

ArcadeDB supports High Availability via Leader/Replica model using RAFT consensus for election and replication.

### Prerequisites

To enable HA:

1. **Enable HA Module:** Set `arcadedb.ha.enabled=true`

2. **Define Servers:** Set `arcadedb.ha.serverList` with cluster members in format: `<hostname/ip:port>`

   Example:
   ```bash
   -Darcadedb.ha.serverList=192.168.0.2,192.168.0.2:2424,192.168.0.3
   ```

   Default port is 2424 if not specified.

3. **Set Server Name:** Each node must have unique name via `arcadedb.server.name` setting

   Default: `ArcadeDB_0`

   In HA clusters, this is mandatory for all servers.

   Example:
   ```bash
   bin/server.sh -Darcadedb.ha.enabled=true \
                 -Darcadedb.server.name=FloridaServer \
                 -Darcadedb.ha.serverList=192.168.0.2,192.168.0.2:2424,japan-server:8888
   ```

**Log Output:**
```
[HttpServer] <FloridaServer> - HTTP Server started (host=0.0.0.0 port=2480 httpsPort=[2490, 2491])
[LeaderNetworkListener] <FloridaServer> Listening for replication connections on 127.0.0.1:2424 (protocol v.-1)
[HAServer] <FloridaServer> Unable to find any Leader, start election (cluster=arcadedb configured_servers=1 majorityOfVotes=1)
[HAServer] <FloridaServer> Change election status from DONE to VOTING_FOR_ME
[HAServer] <FloridaServer> Starting election of local server asking for votes from []
[HAServer] <FloridaServer> Current server elected as new Leader (turn=1 totalVotes=1 majority=1)
[HAServer] <FloridaServer> Change election status from VOTING_FOR_ME to LEADER_WAITING_FOR_QUORUM
[HAServer] <FloridaServer> Contacting all the servers for the new leadership (turn=1 ),...)
```

At startup, server looks for existing cluster based on server list. If not found, creates new cluster. Leader election begins if cluster exists and has no leader. For existing leader, existing leader is kept.

**Cluster Name:** Default "arcadedb". For multiple clusters in same network, customize via `arcadedb.ha.clusterName` setting. Example:
```bash
-Darcadedb.ha.clusterName=projectB
```

**Note:** Cluster name by default is "arcadedb", but you can have multiple clusters in the same network.

### Architecture

ArcadeDB uses **Leader/Replica model** with **RAFT consensus** for election and replication.

```
ArcadeDB_0 (Leader) → ArcadeDB_1 (Replica) → Journal (d)
        ↓
        └──→ ArcadeDB_2 (Replica) → Journal (d)
        ↓
        └──→ Journal (d)
```

**Each server has its own journal** used for recovery to get most updated replica and align other nodes.

**Read Operations:**
- Executed on any server in cluster
- No coordination needed

**Write Operations:**
- Only leader executes writes
- Coordinated by leader node
- All write operations must be coordinated by leader

**Client Behavior:**
- Clients can connect directly to leader OR replica servers
- For reads: connect to any server (read executed on that server)
- For writes: can connect to any server (replica forwards write request to leader)
- Everything transparent to end user - both leader and replica can read and write

### Auto Failover

Cluster uses quorum to ensure database integrity across servers. **Quorum default: MAJORITY** - majority of servers must return same result to accept transaction and propagate to all servers.

**If quorum not met:**
- Transaction rolled back on all servers
- Database returns to previous state
- Transaction error thrown to client

ArcadeDB manages failover automatically in most cases.

**Server Unreachable Reasons:**
- ArcadeDB server process has been terminated
- Physical/virtual server hosting process has been shut off or is rebooting
- Physical/virtual server hosting process has network issues and can't reach other servers
- Network issues preventing ArcadeDB server from communicating with rest of servers

### HA Settings

| Setting | Description | Default |
|---------|-------------|---------|
| arcadedb.ha.clusterName | Cluster name (useful for multiple clusters) | arcadedb |
| arcadedb.ha.serverList | Servers in cluster as list of `<hostname/ip:port>` separated by comma | NOT DEFINED (auto discovery by default) |
| arcadedb.ha.serverRole | Enforce role in cluster: "any" or "replica" | "any" |
| arcadedb.ha.quorum | Default quorum: 'none', 1, 2, 3, 'majority' and 'all' | MAJORITY |
| arcadedb.ha.quorumTimeout | Timeout waiting for quorum | 10000 |
| arcadedb.ha.k8s | Server running inside Kubernetes | false |
| arcadedb.ha.k8sSuffix | Suffix to reach other servers inside Kubernetes | arcadedb.default.svc.cluster.local |
| arcadedb.ha.replicationQueueSize | Queue size for replicating messages between servers | 512 |
| arcadedb.ha.replicationFileMaxSize | Maximum file size for replicating messages between servers | 1GB |
| arcadedb.ha.replicationChunkMaxSize | Maximum channel chunk size for replicating messages between servers | 16777216 |
| arcadedb.ha.replicationIncomingHost | TCP/IP host name for incoming replication connections | localhost |
| arcadedb.ha.replicationIncomingPorts | TCP/IP port number (range) for incoming replication connections | 2424-2433 |

---

## Docker

### Running with Docker

**Basic Example:**
```bash
docker run --rm -p 2480:2480 -p 2424:2424 --name my_arcadedb \
  --env JAVA_OPTS="-Darcadedb.server.rootPassword=playwithdata" \
  --hostname my_arcadedb arcadedata/arcadedb:26.2.1
```

**Docker Console Access:**
```bash
docker exec -it my_arcadedb bin/console.sh
```

**With Podman:**
The ArcadeDB image works with Podman - just replace `docker` with `podman` in commands.

### Persistence

By default, data created in Docker container is lost when container stops. To preserve data:

#### Understanding Database Directory

ArcadeDB stores databases in `/home/arcadedb/databases` by default. To persist data beyond container lifecycle, mount this directory.

#### Docker Volumes vs. Bind Mounts

**Docker Volumes (Recommended):**
- Docker-managed storage sections on host filesystem
- Portable and easier to manage
- Recommended for most use cases

**Bind Mounts:**
- Direct mapping to specific path on host machine
- Full control over exact location
- Useful when you need to inspect, backup, or manipulate database files directly

#### Persisting the Database Directory

Mount the default location with bind mount:

```bash
docker run --rm -p 2480:2480 -p 2424:2424 --name my_arcadedb \
  -v /path/on/host/databases:/home/arcadedb/databases \
  --env JAVA_OPTS="-Darcadedb.server.rootPassword=playwithdata" \
  --hostname my_arcadedb arcadedata/arcadedb:26.2.1
```

Replace `/path/on/host/databases` with your desired directory path. All database read and write operations occur on this mounted volume.

#### Alternative: Custom Database Directory

If you need to change where ArcadeDB stores databases, use the `-Darcadedb.server.databaseDirectory` setting and mount your volume to match:

```bash
docker run --rm -p 2480:2480 -p 2424:2424 --name my_arcadedb \
  -v /path/on/host/databases:/mydatabases \
  --env JAVA_OPTS="-Darcadedb.server.databaseDirectory=/mydatabases" \
  -Darcadedb.server.rootPassword=playwithdata" \
  --hostname my_arcadedb arcadedata/arcadedb:26.2.1
```

This example changes database directory to `/mydatabases` and mounts volume to that location.

#### Alternative: Backup-Only Persistence

For better performance, keep databases in container storage and only persist backups:

```bash
docker run --rm -p 2480:2480 -p 2424:2424 --name my_arcadedb \
  -v /path/on/host/backups:/home/arcadedb/backups \
  --env JAVA_OPTS="-Darcadedb.server.rootPassword=playwithdata" \
  --hostname my_arcadedb arcadedata/arcadedb:26.2.1
```

This approach:
- Keeps databases in fast container storage
- Runs regular `BACKUP DATABASE` operations
- Persists only backup files to mounted volume
- Still achieves data persistence with potentially better performance

The optimal strategy depends on your infrastructure characteristics - bare metal, VMs, or container orchestration platforms each have different "local storage" performance characteristics.

#### Comprehensive Persistence Example

Mount multiple directories for complete data, backup, log, and configuration persistence:

```bash
docker run --rm -p 2480:2480 -p 2424:2424 --name my_arcadedb \
  -v /path/on/host/databases:/home/arcadedb/databases \
  -v /path/on/host/backups:/home/arcadedb/backups \
  -v /path/on/host/log:/home/arcadedb/log \
  -v /path/on/host/config:/home/arcadedb/config \
  --env JAVA_OPTS="-Darcadedb.server.rootPassword=playwithdata" \
  --hostname my_arcadedb arcadedata/arcadedb:26.2.1
```

### Tuning

Default Docker setup allocates 2GB RAM to ArcadeDB (`-Xms26 -Xmx26`). For systems with less RAM:

**With 1G container:**
```bash
docker run ... -e ARCADEDB_OPTS_MEMORY="-Xms800M -Xmx800M" ...
```

**With <800M RAM:**
```bash
docker run ... -e ARCADEDB_OPTS_MEMORY="-Xms800M -Xmx800M" \
               -e arcadedb.profile=low-ram ...
```

**RAM allocation:** JVM RAM should be <80% of container RAM. For 1G container, allocate 800M to JVM.

### Quick Start with OpenBeer Database

Run ArcadeDB with pre-loaded OpenBeer dataset in <1 minute:

```bash
docker run --rm -p 2480:2480 -p 2424:2424 -p 6379:6379 -p 5432:5432 -p 8182:8182 \
  -e JAVA_OPTS="\
    -Darcadedb.server.rootPassword=playwithdata \
    -Darcadedb.server.defaultDatabases=Imported[root]{import:https://github.com/ArcadeData/arcadedb-datasets/raw/main/orientsdb/OpenBeer.gz}" \
  -Darcadedb.server.plugins="Redis:com.arcadedb.redis.RedisProtocolPlugin, \
    MongoDB:com.arcadedb.mongo.MongoDBProtocolPlugin, \
    Postgres:com.arcadedb.postgres.PostgresProtocolPlugin, \
    GremlinServer:com.arcadedb.server.gremlin.GremlinServerPlugin" \
  arcadedata/arcadedb:latest
```

Navigate to `http://localhost:2480` and login with:
- Username: `root`
- Password: `playwithdata`

Click "Database" > "OpenBeer" vertex type > "Display first 100 records of brewery together with all vertices that are directly connected" to see the first 100 beers and their connections.

---

## Kubernetes

### Prerequisites

Before starting Kubernetes (K8S) cluster, set ArcadeDB root password as secret:

```bash
kubectl create secret generic arcadedb-credentials --from-literal=rootPassword='<password>'
```

Replace `<password>` with desired root password.

### Quick Start

Start K8S cluster with 3 servers using default configuration:

```bash
kubectl apply -f config/arcadedb-statefulset.yaml
```

This deploys:
- StatefulSet with 3 ArcadeDB server replicas
- Service for cluster communication
- PersistentVolumeClaims for data storage

### Scaling

Scale up or down number of replicas:

```bash
kubectl scale statefulsets arcadedb-server --replicas=<new-number-of-replicas>
```

Example - scale to 3 replicas:
```bash
kubectl scale statefulsets arcadedb-server --replicas=3
```

Scaling up/down doesn't affect current workload - no pauses when servers enter/exit cluster.

### Helm Charts

For easier deployment on K8S clusters, use Helm chart with release name `my-arcadedb`:

```bash
helm install my-arcadedb ./arcadedb
```

To uninstall/delete the `my-arcadedb` deployment:

```bash
helm uninstall my-arcadedb
```

The command removes all K8S components associated with the chart and deletes the release.

See dedicated [README](link) for configurable parameters of Helm chart, persistence, ingress, and resource configuration.

### Troubleshooting

**Common K8S Issues:**

**Display pod status:**
```bash
kubectl describe pod arcadedb-0
```

Replace "arcadedb-0" with the server container you want to check.

**Display server logs:**
```bash
kubectl logs arcadedb-0
```

Replace "arcadedb-0" with the server container you want to check.

**Remove all containers and restart cluster:**
```bash
kubectl delete all --all --namespace default
kubectl apply -f config/arcadedb-statefulset.yaml
```

**Most Common Issue:** Root password not set as secret at beginning of section.

---

## Appendix

### All Configuration Settings

See main ArcadeDB documentation Appendix for complete list of configuration settings available via `-Darcadedb.` prefix.

