---
title: In-Memory OLTP improves SQL txn perf
description: Adopt In-Memory OLTP to improve transactional performance in an existing database in Azure SQL Database and Azure SQL Managed Instance.
author: WilliamDAssafMSFT
ms.author: wiassaf
ms.reviewer: mathoma
ms.date: 11/07/2018
ms.service: sql-database
ms.subservice: performance
ms.topic: how-to
ms.custom: sqldbrb=2
monikerRange: "=azuresql||=azuresql-db||=azuresql-mi"
---
# Use In-Memory OLTP to improve your application performance in Azure SQL Database and Azure SQL Managed Instance
[!INCLUDE[appliesto-sqldb-sqlmi](includes/appliesto-sqldb-sqlmi.md)]

[In-Memory OLTP](in-memory-oltp-overview.md) can be used to improve the performance of transaction processing, data ingestion, and transient data scenarios, in [Premium and Business Critical tier](database/service-tiers-vcore.md) databases without increasing the pricing tier.

> [!NOTE]
> Learn how [Quorum doubles key database's workload while lowering DTU by 70% with Azure SQL Database](https://customers.microsoft.com/story/quorum-doubles-key-databases-workload-while-lowering-dtu-with-sql-database)

Follow these steps to adopt In-Memory OLTP in your existing database.

## Step 1: Ensure you are using a Premium and Business Critical tier database

In-Memory OLTP is supported only in Premium and Business Critical tier databases. In-Memory is supported if the returned result is 1 (not 0):

```sql
SELECT DatabasePropertyEx(Db_Name(), 'IsXTPSupported');
```

*XTP* stands for *Extreme Transaction Processing*

## Step 2: Identify objects to migrate to In-Memory OLTP

SSMS includes a **Transaction Performance Analysis Overview** report that you can run against a database with an active workload. The report identifies tables and stored procedures that are candidates for migration to In-Memory OLTP.

In SSMS, to generate the report:

* In the **Object Explorer**, right-click your database node.
* Click **Reports** > **Standard Reports** > **Transaction Performance Analysis Overview**.

For more information, see [Determining if a Table or Stored Procedure Should Be Ported to In-Memory OLTP](/sql/relational-databases/in-memory-oltp/determining-if-a-table-or-stored-procedure-should-be-ported-to-in-memory-oltp).

## Step 3: Create a comparable test database

Suppose the report indicates your database has a table that would benefit from being converted to a memory-optimized table. We recommend that you first test to confirm the indication by testing.

You need a test copy of your production database. The test database should be at the same service tier level as your production database.

To ease testing, tweak your test database as follows:

1. Connect to the test database by using SSMS.
2. To avoid needing the WITH (SNAPSHOT) option in queries, set the database option as shown in the following T-SQL statement:

   ```sql
   ALTER DATABASE CURRENT
    SET
        MEMORY_OPTIMIZED_ELEVATE_TO_SNAPSHOT = ON;
   ```

## Step 4: Migrate tables

You must create and populate a memory-optimized copy of the table you want to test. You can create it by using either:

* The handy Memory Optimization Wizard in SSMS.
* Manual T-SQL.

### Memory Optimization Wizard in SSMS

To use this migration option:

1. Connect to the test database with SSMS.
2. In the **Object Explorer**, right-click on the table, and then click **Memory Optimization Advisor**.

   The **Table Memory Optimizer Advisor** wizard is displayed.
3. In the wizard, click **Migration validation** (or the **Next** button) to see if the table has any unsupported features that are unsupported in memory-optimized tables. For more information, see:

   * The *memory optimization checklist* in [Memory Optimization Advisor](/sql/relational-databases/in-memory-oltp/memory-optimization-advisor).
   * [Transact-SQL Constructs Not Supported by In-Memory OLTP](/sql/relational-databases/in-memory-oltp/transact-sql-constructs-not-supported-by-in-memory-oltp).
   * [Migrating to In-Memory OLTP](/sql/relational-databases/in-memory-oltp/plan-your-adoption-of-in-memory-oltp-features-in-sql-server).
4. If the table has no unsupported features, the advisor can perform the actual schema and data migration for you.

### Manual T-SQL

To use this migration option:

1. Connect to your test database by using SSMS (or a similar utility).
2. Obtain the complete T-SQL script for your table and its indexes.

   * In SSMS, right-click your table node.
   * Click **Script Table As** > **CREATE To** > **New Query Window**.
3. In the script window, add WITH (MEMORY_OPTIMIZED = ON) to the CREATE TABLE statement.
4. If there is a CLUSTERED index, change it to NONCLUSTERED.
5. Rename the existing table by using SP_RENAME.
6. Create the new memory-optimized copy of the table by running your edited CREATE TABLE script.
7. Copy the data to your memory-optimized table by using INSERT...SELECT * INTO:

```sql
INSERT INTO [<new_memory_optimized_table>]
        SELECT * FROM [<old_disk_based_table>];
```

## Step 5 (optional): Migrate stored procedures

The In-Memory feature can also modify a stored procedure for improved performance.

### Considerations with natively compiled stored procedures

A natively compiled stored procedure must have the following options on its T-SQL WITH clause:

* NATIVE_COMPILATION
* SCHEMABINDING: meaning tables that the stored procedure cannot have their column definitions changed in any way that would affect the stored procedure, unless you drop the stored procedure.

A native module must use one big [ATOMIC blocks](/sql/relational-databases/in-memory-oltp/atomic-blocks-in-native-procedures) for transaction management. There is no role for an explicit BEGIN TRANSACTION, or for ROLLBACK TRANSACTION. If your code detects a violation of a business rule, it can terminate the atomic block with a [THROW](/sql/t-sql/language-elements/throw-transact-sql) statement.

### Typical CREATE PROCEDURE for natively compiled

Typically the T-SQL to create a natively compiled stored procedure is similar to the following template:

```sql
CREATE PROCEDURE schemaname.procedurename
    @param1 type1, …
    WITH NATIVE_COMPILATION, SCHEMABINDING
    AS
        BEGIN ATOMIC WITH
            (TRANSACTION ISOLATION LEVEL = SNAPSHOT,
            LANGUAGE = N'your_language__see_sys.languages'
            )
        …
        END;
```

* For the TRANSACTION_ISOLATION_LEVEL, SNAPSHOT is the most common value for the natively compiled stored procedure. However, a subset of the other values is also supported:
  
  * REPEATABLE READ
  * SERIALIZABLE
* The LANGUAGE value must be present in the sys.languages view.

### How to migrate a stored procedure

The migration steps are:

1. Obtain the CREATE PROCEDURE script to the regular interpreted stored procedure.
2. Rewrite its header to match the previous template.
3. Ascertain whether the stored procedure T-SQL code uses any features that are not supported for natively compiled stored procedures. Implement workarounds if necessary.

   For details see [Migration Issues for Natively Compiled Stored Procedures](/sql/relational-databases/in-memory-oltp/a-guide-to-query-processing-for-memory-optimized-tables).
4. Rename the old stored procedure by using SP_RENAME. Or simply DROP it.
5. Run your edited CREATE PROCEDURE T-SQL script.

## Step 6: Run your workload in test

Run a workload in your test database that is similar to the workload that runs in your production database. This should reveal the performance gain achieved by your use of the In-Memory feature for tables and stored procedures.

Major attributes of the workload are:

* Number of concurrent connections.
* Read/write ratio.

To tailor and run the test workload, consider using the handy `ostress.exe` tool, which illustrated in this [in-memory](in-memory-oltp-overview.md) article.

To minimize network latency, run your test in the same Azure geographic region where the database exists.

## Step 7: Post-implementation monitoring

Consider monitoring the performance effects of your In-Memory implementations in production:

* [Monitor In-Memory storage](in-memory-oltp-monitor-space.md).
* [Monitoring using dynamic management views](database/monitoring-with-dmvs.md)

## Related links

* [In-Memory OLTP (In-Memory Optimization)](/sql/relational-databases/in-memory-oltp/in-memory-oltp-in-memory-optimization)
* [Introduction to Natively Compiled Stored Procedures](/sql/relational-databases/in-memory-oltp/a-guide-to-query-processing-for-memory-optimized-tables)
* [Memory Optimization Advisor](/sql/relational-databases/in-memory-oltp/memory-optimization-advisor)
