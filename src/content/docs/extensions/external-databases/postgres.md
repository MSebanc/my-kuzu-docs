---
title: PostgreSQL Extension
---
The PostgreSQL extension allows Kùzu to directly scan data from PostgreSQL databases.
Currently, this is done by `LOAD FROM` statements.
This allows users to not only view their PostgreSQL tables in Kùzu, but also facilitates seamless
migration of data from PostgreSQL to Kùzu for deeper graph analysis. Currently, the extension is read-only
from PostgreSQL and does not support write operations.

:::note[Notes]
- This extension works for PostgreSQL versions 14 and above (and possibly on older versions,
  though this hasn't been tested extensively).
- If you experience an error while loading the extension, ensure that `postgres` 14 or above is installed
  on your system. On MacOS for PostgreSQL 16, this is done via `brew install postgres@16`. See the PostgreSQL
  [installation guide](https://www.postgresql.org/download/) for specific instructions for your OS.
  :::

## Usage

`postgres` is an official extension developed and maintained by Kùzu.
It can be installed and loaded by running the following commands using the CLI or your preferred language
client API:

```
INSTALL postgres;
LOAD EXTENSION postgres;
```

## Direct scan from PostgreSQL

This section shows how to directly scan from PostgreSQL tables to Kùzu using the `LOAD FROM` statement.

### Set up a PostgreSQL server via Docker

It's convenient to set up a PostgreSQL server using Docker. Run the following command to start a
PostgreSQL server on your local machine:

```sh
docker run --name kuzu-postgres -e POSTGRES_PASSWORD=testpassword -p 5432:5432 --rm postgres:latest
```

Note that the storage volume for this database is not persistent and will be deleted once the
container is stopped. Moreover, the password is provided via plain text, which is not recommended
in a real use case, so the below example is for testing purposes only.

### Create a sample PostgreSQL database

To illustrate the usage of the extension, we create a sample Postgres database of university
students. We will use [asyncpg](https://magicstack.github.io/asyncpg/current/index.html),
an asynchronous PostgreSQL client library for Python, to create the database and insert some sample data
via a Python script.
This step assumes you have run `pip install asyncpg` to install the library, and are using Python 3.10+.

```py
import asyncio
import asyncpg

async def main():
    conn = await asyncpg.connect('postgresql://postgres:testpassword@localhost:5432/postgres')
    # Create and insert data to a new table
    try:
        await conn.execute("CREATE TABLE person (name VARCHAR, age INTEGER);")
        await conn.execute("INSERT INTO person (name, age) VALUES ('Alice', 30)")
        await conn.execute("INSERT INTO person (name, age) VALUES ('Bob', 27)")
        await conn.execute("INSERT INTO person (name, age) VALUES ('Carol', 19)")
        await conn.execute("INSERT INTO person (name, age) VALUES ('Dan', 25)")
    except asyncpg.exceptions.DuplicateTableError:
        print(f"Table already exists, skipping creation and insertion...")
    # Check results
    print(await conn.fetch("SELECT * FROM person"))

asyncio.run(main())
```

### Attach PostgreSQL instance in Kùzu

```sql
ATTACH [PG_CONNECTION_STRING] AS [alias] (dbtype 'postgres')
```

The below example shows how the `university` PostgreSQL database can be attached to Kùzu using
the alias `uw`:

```sql
ATTACH 'dbname=university user=postgres host=localhost password=testpassword port=5432' AS uw (dbtype 'postgres');
```

The `ATTACH` statement requires the following parameters:

- `PG_CONNECTION_STRING`: PostgreSQL connection string with the necessary parameters
- `alias`: Database alias to use in Kùzu - If not provided, the database name from PostgreSQL will be used.
  When attaching multiple databases, it's recommended to use aliasing.

The below table lists some common connection string parameters:

| Parameter | Description | Default |
|-----------|-------------| ------- |
| `dbname`    | Database name | [user defined] |
| `host`      | Host IP address | `localhost` |
| `user`      | Postgres username | `postgres` |
| `password`  | Postgres password | [empty] |
| `port`      | Port number | 5432 |

### Scan from PostgreSQL tables

Finally, we can utilize the `LOAD FROM` statement to scan the `Person` table.

```sql
LOAD FROM uw.person
RETURN *
```

Result:

```
---------------
| name  | age |
---------------
| Alice | 30  |
---------------
| Bob   | 27  |
---------------
| Carol | 19  |
---------------
| Dan   | 25  |
---------------
```

## Data migration from PostgreSQL tables

One important use case of the PostgreSQL extension is to facilitate seamless data transfer from PostgreSQL to Kùzu.
In this example, we continue using the `university` database created in the last step, but this time,
we copy the data and persist it to Kùzu. This is done with the [`COPY FROM` query results feature](../import/copy-from-query-results).

### Create a `Person` table in Kùzu

We first create a `Person` table in Kùzu which has the same schema as the one defined in PostgreSQL.

```sql
CREATE NODE TABLE Person (name STRING, age INT32, PRIMARY KEY(name));
```

### Use `COPY FROM` to migrate data

We can reference the created alias `uw` to copy data from the PostgreSQL table to the Kùzu table.

```sql
COPY Person FROM (LOAD FROM uw.person RETURN *);
```

### Query the data in Kùzu

Finally, we can verify the data in the `Person` table in Kùzu.

```cypher
MATCH (p:Person) RETURN p.*;
```

Result:
```
------------------
| s.name | s.age |
------------------
| Alice  | 30    |
------------------
| Bob    | 27    |
------------------
| Carol  | 19    |
------------------
| Dan    | 25    |
------------------
```

## Clear attached database schema cache

To avoid redundantly retrieving schema information from attached databases, Kùzu maintains a schema cache
including table names and their respective columns and types. Should modifications occur in the schema
via an alternate connection to attached databases (PosgreSQL or some other attachad database), such as creation or deletion of tables, the cached
schema data may become obsolete. You can use the `clear_attached_db_cache()` function to refresh cached
schema information in such cases.

```sql
CALL clear_attached_db_cache() RETURN *;
```

Note: this call function will clear cache of all attached databases.

## `USE` statement

The `USE` statement for attached databases sets a default database name to use for future operations.
This can be used when reading from an attached database to avoid specifying the full database name
as a prefix to the table name.

Consider the same attached database as above:

```sql
ATTACH 'university' AS uw (dbtype 'postgres');
```

Instead of defining the Postgres database name for each subsequent clause like this:

```sql
LOAD FROM uw.person
RETURN *
```

You can do:

```sql
USE uw;
LOAD FROM person
RETURN *
```
