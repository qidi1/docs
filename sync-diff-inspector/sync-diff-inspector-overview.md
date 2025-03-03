---
title: sync-diff-inspector User Guide
summary: Use sync-diff-inspector to compare data and repair inconsistent data.
aliases: ['/docs/dev/sync-diff-inspector/sync-diff-inspector-overview/','/docs/dev/reference/tools/sync-diff-inspector/overview/']
---

# sync-diff-inspector User Guide

[sync-diff-inspector](https://github.com/pingcap/tidb-tools/tree/master/sync_diff_inspector) is a tool used to compare data stored in the databases with the MySQL protocol. For example, it can compare the data in MySQL with that in TiDB, the data in MySQL with that in MySQL, or the data in TiDB with that in TiDB. In addition, you can also use this tool to repair data in the scenario where a small amount of data is inconsistent.

This guide introduces the key features of sync-diff-inspector and describes how to configure and use this tool. To download sync-diff-inspector, use one of the following methods:

+ Binary package. Click [tidb-enterprise-tools-nightly-linux-amd64](https://download.pingcap.org/tidb-enterprise-tools-nightly-linux-amd64.tar.gz) to download.
+ Docker image. Execute the following command to download:

    {{< copyable "shell-regular" >}}

    ```shell
    docker pull pingcap/tidb-enterprise-tools
    ```

## Key features

* Compare the table schema and data
* Generate the SQL statements used to repair data if the data inconsistency exists
* Support [data check for tables with different schema or table names](/sync-diff-inspector/route-diff.md)
* Support [data check in the sharding scenario](/sync-diff-inspector/shard-diff.md)
* Support [data check for TiDB upstream-downstream clusters](/sync-diff-inspector/upstream-downstream-diff.md)

## Usage of sync-diff-inspector

### Restrictions

* Online check is not supported for data migration between MySQL and TiDB. Ensure that no data is written into the upstream-downstream checklist, and that data in a certain range is not changed. You can check data in this range by setting `range`.

* `JSON`, `BIT`, `BINARY`, `BLOB` and other types of data are not supported. When you perform a data check, you need to set `ignore-columns` to skip checking these types of data.

* In TiDB and MySQL, `FLOAT`, `DOUBLE` and other floating-point types are implemented differently, so checksum might be calculated differently. If the data checks are inconsistent due to these data types, set `ignore-columns` to skip checking these columns.

* Support checking tables that do not contain the primary key or the unique index. However, if data is inconsistent, the generated SQL statements might not be able to repair the data correctly.

### Database privilege

sync-diff-inspector needs to obtain the information of table schema, to query data, and to build the `checkpoint` database to save breakpoint information. The required database privileges are as follows:

* Upstream database
    - `SELECT` (checks data for comparison)
    - `SHOW_DATABASES` (views database name)
    - `RELOAD` (views table schema)
* Downstream database
    - `SELECT` (checks data for comparison)
    - `CREATE` (creates the `checkpoint` database and tables)
    - `DELETE` (deletes information in the `checkpoint` table)
    - `INSERT` (writes data into the `checkpoint` table)
    - `UPDATE` (modifies the `checkpoint` table)
    - `SHOW_DATABASES` (views database name)
    - `RELOAD` (views table schema)

### Configuration file description

The configuration of sync-diff-inspector consists of three parts:

- `Global config`: General configurations, including log level, chunk size, number of threads to check.
- `Tables config`: Configures the tables for checking. If some tables have a certain mapping relationship between the upstream and downstream databases or have some special requirements, you must configure these tables.
- `Databases config`: Configures the instances of the upstream and downstream databases.

Below is the description of a complete configuration file:

``` toml
# Diff Configuration.

######################### Global config #########################

# The log level. You can set it to "info" or "debug".
log-level = "info"

# sync-diff-inspector divides the data into multiple chunks based on the primary key,
# unique key, or the index, and then compares the data of each chunk.
# Uses "chunk-size" to set the size of a chunk.
chunk-size = 1000

# The number of goroutines created to check data
check-thread-count = 4

# The proportion of sampling check. If you set it to 100, all the data is checked.
sample-percent = 100

# If enabled, the chunk's checksum is calculated and data is compared by checksum.
# If disabled, data is compared line by line.
use-checksum = true

# If it is set to true, data is checked only by calculating checksum. Data is not checked after inspection, even if the upstream and downstream checksums are inconsistent.
only-use-checksum = false

# Whether to use the checkpoint of the last check. If it is enabled, the inspector only checks the last unverified chunks and chunks that failed the verification.
use-checkpoint = true

# If it is set to true, data check is ignored.
# If it is set to false, data is checked.
ignore-data-check = false

# If it is set to true, the table struct comparison is ignored.
# If set to false, the table struct is compared.
ignore-struct-check = false

# The name of the file which saves the SQL statements used to repair data
fix-sql-file = "fix.sql"

######################### Tables config #########################

# To compare the data of a large number of tables with different schema names or table names, or check the data of multiple upstream sharded tables and downstream table family, use the table-rule to configure the mapping relationship. You can configure the mapping rule only for the schema or table. Also, you can configure the mapping rules for both the schema and the table.
#[[table-rules]]
    # schema-pattern and table-pattern support the wildcard *?
    # schema-pattern = "test_*"
    # table-pattern = "t_*"
    # target-schema = "test"
    # target-table = "t"

# Configures the tables of the target database that need to be compared.
[[check-tables]]
    # The name of the schema in the target database
    schema = "test"

    # The list of tables that need to be checked in the target database
    tables = ["test1", "test2", "test3"]

    # Supports using regular expressions to configure tables to be checked.
    # You need to start with '~'. For example, the following configuration checks
    # all the tables with the prefix 'test' in the table name.
    # tables = ["~^test.*"]
    # The following configuration checks all the tables in the database.
    # tables = ["~^"]

# Special configuration for some tables
# The configured table must be included in "check-tables'.
[[table-config]]
    # The name of the schema in the target database
    schema = "test"

    # The table name
    table = "test3"

    # Specifies the column used to divide data into chunks. If you do not configure it,
    # sync-diff-inspector chooses an appropriate column (primary key, unique key, or a field with index).
    index-fields = "id"

    # Specifies the range of the data to be checked
    # It needs to comply with the syntax of the WHERE clause in SQL.
    range = "age > 10 AND age < 20"

    # Sets it to "true" when comparing the data of multiple sharded tables
    # with the data of the combined table.
    is-sharding = false

    # The collation of the string type of data might be inconsistent in some conditions.
    # You can specify "collation" to guarantee the order consistency.
    # You need to keep it corresponding to the "charset" setting in the database.
    # collation = "latin1_bin"

    # Ignores checking some columns such as some types (json, bit, blob, etc.)
    # that sync-diff-inspector does not currently support.
    # The floating-point data type behaves differently in TiDB and MySQL. You can use
    # `ignore-columns` to skip checking these columns.
    # ignore-columns = ["name"]

# Configuration example of comparing two tables with different schema names and table names.
[[table-config]]
    # The name of the target schema
    schema = "test"

    # The name of the target table
    table = "test2"

    # Sets it to "false" in non-sharding scenarios.
    is-sharding = false

    # Configuration of the source data
    [[table-config.source-tables]]
        # The instance ID of the source schema
        instance-id = "source-1"
        # The name of the source schema
        schema = "test"
        # The name of the source table
        table  = "test1"

######################### Databases config #########################

# Configuration of the source database instance
[[source-db]]
    host = "127.0.0.1"
    port = 3306
    user = "root"
    password = "123456"
    # The instance ID of the source database, the unique identifier of a database instance
    instance-id = "source-1"
    # Uses the snapshot function of TiDB.
    # If enabled, the history data is used for comparison.
    # snapshot = "2016-10-08 16:45:26"
    # Sets the `sql-mode` of the database to parse table structures.
    # sql-mode = ""

# Configuration of the target database instance
[target-db]
    host = "127.0.0.1"
    port = 4000
    user = "root"
    password = "123456"
    # Uses the snapshot function of TiDB.
    # If enabled, the history data is used for comparison.
    # snapshot = "2016-10-08 16:45:26"
    # Sets the `sql-mode` of the database to parse table structures.
    # sql-mode = ""
```

### Run sync-diff-inspector

Run the following command:

``` bash
./bin/sync_diff_inspector --config=./config.toml
```

This command outputs a check report to the log and describes the check result of each table. sync-diff-inspector generates the SQL statements to fix inconsistent data and stores these statements in a `fix.sql` file.

#### Log

The running sync-diff-inspector periodically (every 10 seconds) prints the progress in log in the following format:

```log
[2020/11/12 17:47:00.170 +08:00] [INFO] [checkpoint.go:276] ["summary info"] [instance_id=target] [schema=test] [table=test_table] ["chunk num"=1000] ["success num"=80] ["failed num"=1] ["ignore num"=0]
```

- `chunk num`: The total number of chunks to be checked.
- `success num`: The number of chunks that have been checked as consistent.
- `failed num`: The number of chunks that fail the check. Check failure might be caused by errors or data inconsistency.
- `ignore num`: The number of chunks that are ignored for the check. If the value of `sample-percent` is smaller than `100`, sync-diff-inspector checks data by sampling, where some chunks are ignored.

#### Result

After the check is finished, sync-diff-inspector outputs a report.

+ If the data is consistent, a log example is as follows:

    ```log
    [2020/11/12 17:47:00.174 +08:00] [INFO] [report.go:80] ["check result summary"] ["check passed num"=1] ["check failed num"=0]
    [2020/11/12 17:47:00.174 +08:00] [INFO] [report.go:87] ["table check result"] [schema=test] [table=test_table] ["struct equal"=true] ["data equal"=true]
    [2020/11/12 17:47:00.174 +08:00] [INFO] [main.go:75] ["check data finished"] [cost=353.462744ms]
    [2020/11/12 17:47:00.174 +08:00] [INFO] [main.go:69] ["check pass!!!"]
    ```

+ If the data is inconsistent or some errors occur, a log example is as follows:

    ```log
    [2020/11/12 18:16:17.068 +08:00] [INFO] [checkpoint.go:276] ["summary info"] [instance_id=target] [schema=test] [table=test1] ["chunk num"=1] ["success num"=0] ["failed num"=1] ["ignore num"=0]
    [2020/11/12 18:16:17.071 +08:00] [INFO] [report.go:80] ["check result summary"] ["check passed num"=0] ["check failed num"=1]
    [2020/11/12 18:16:17.071 +08:00] [INFO] [report.go:87] ["table check result"] [schema=test] [table=test_table] ["struct equal"=true] ["data equal"=false]
    [2020/11/12 18:16:17.071 +08:00] [INFO] [main.go:75] ["check data finished"] [cost=319.849706ms]
    [2020/11/12 18:16:17.071 +08:00] [WARN] [main.go:66] ["check failed!!!"]
    ```

The number of tables that have passed and failed the check is printed in `check result summary`. The results of all tables are printed in `table check result`.

### Note

- sync-diff-inspector consumes a certain amount of server resources when checking data. Avoid using sync-diff-inspector to check data during peak business hours.
- TiDB uses the `utf8_bin` collation. If you need to compare the data in MySQL with that in TiDB, pay attention to the collation configuration of MySQL tables. If the primary key or unique key is the `varchar` type and the collation configuration in MySQL differs from that in TiDB, then the final check result might be incorrect because of the collation issue. You need to add collation to the sync-diff-inspector configuration file.
- sync-diff-inspector divides data into chunks first according to TiDB statistics and you need to guarantee the accuracy of the statistics. You can manually run the `analyze table {table_name}` command when the TiDB server's *workload is light*.
- Pay special attention to `table-rules`. If you configure `schema-pattern="test1"` and `target-schema="test2"`, the `test1` schema in the source database and the `test2` schema in the target database are compared. If the source database has a `test2` schema, this `test2` schema is also compared with the `test2` schema in the target database.
- The generated `fix.sql` is only used as a reference for repairing data, and you need to confirm it before executing these SQL statements to repair data.
