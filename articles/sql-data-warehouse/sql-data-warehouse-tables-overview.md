---
title: Designing tables - Azure SQL Data Warehouse | Microsoft Docs
description: Introduction to designing tables in Azure SQL Data Warehouse. 
services: sql-data-warehouse
author: ronortloff
manager: craigg
ms.service: sql-data-warehouse
ms.topic: conceptual
ms.subservice: implement
ms.date: 04/17/2018
ms.author: rortloff
ms.reviewer: igorstan
---

# Designing tables in Azure SQL Data Warehouse

Learn key concepts for designing tables in Azure SQL Data Warehouse. 

## Determine table category 

A [star schema](https://en.wikipedia.org/wiki/Star_schema) organizes data into fact and dimension tables. Some tables are used for integration or staging data before it moves to a fact or dimension table. As you design a table, decide whether the table data belongs in a fact, dimension, or integration table. This decision informs the appropriate table structure and distribution. 

- **Fact tables** contain quantitative data that are commonly generated in a transactional system, and then loaded into the data warehouse. For example, a retail business generates sales transactions every day, and then loads the data into a data warehouse fact table for analysis.

- **Dimension tables** contain attribute data that might change but usually changes infrequently. For example, a customer's name and address are stored in a dimension table and updated only when the customer's profile changes. To minimize the size of a large fact table, the customer's name and address do not need to be in every row of a fact table. Instead, the fact table and the dimension table can share a customer ID. A query can join the two tables to associate a customer's profile and transactions. 

- **Integration tables** provide a place for integrating or staging data. You can create an integration table as a regular table, an external table, or a temporary table. For example, you can load data to a staging table, perform transformations on the data in staging, and then insert the data into a production table.

## Schema and table names
In SQL Data Warehouse, a data warehouse is a type of database. All of the tables in the data warehouse are contained within the same database.  You cannot join tables across multiple data warehouses. This behavior is different from SQL Server, which supports cross-database joins. 

In a SQL Server database, you might use fact, dim, or integrate for the schema names. If you are migrating a SQL Server database to SQL Data Warehouse, it works best to migrate all of the fact, dimension, and integration tables to one schema in SQL Data Warehouse. For example, you could store all the tables in the [WideWorldImportersDW](/sql/sample/world-wide-importers/database-catalog-wwi-olap) sample data warehouse within one schema called wwi. The following code creates a [user-defined schema](/sql/t-sql/statements/create-schema-transact-sql) called wwi.

```sql
CREATE SCHEMA wwi;
```

To show the organization of the tables in SQL Data Warehouse, you could use fact, dim, and int as prefixes to the table names. The following table shows some of the schema and table names for WideWorldImportersDW. It compares the names in SQL Server with names in SQL Data Warehouse. 

| WideWorldImportersDW table  | Table type | SQL Server | SQL Data Warehouse |
|:-----|:-----|:------|:-----|
| City | Dimension | Dimension.City | wwi.DimCity |
| Order | Fact | Fact.Order | wwi.FactOrder |


## Table persistence 

Tables store data either permanently in Azure Storage, temporarily in Azure Storage, or in a data store external to data warehouse.

### Regular table

A regular table stores data in Azure Storage as part of the data warehouse. The table and the data persist regardless of whether a session is open.  This example creates a regular table with two columns. 

```sql
CREATE TABLE MyTable (col1 int, col2 int );  
```

### Temporary table
A temporary table only exists for the duration of the session. You can use a temporary table to prevent other users from seeing temporary results and also to reduce the need for cleanup.  Since temporary tables also utilize local storage, they can offer faster performance for some operations.  For more information, see  [Temporary tables](sql-data-warehouse-tables-temporary.md).

### External table
An external table points to data located in Azure Storage blob or Azure Data Lake Store. When used in conjunction with the CREATE TABLE AS SELECT statement, selecting from an external table imports data into SQL Data Warehouse. External tables are therefore useful for loading data. For a loading tutorial, see [Use PolyBase to load data from Azure blob storage](load-data-from-azure-blob-storage-using-polybase.md).

## Data types
SQL Data Warehouse supports the most commonly used data types. For a list of the supported data types, see [data types in CREATE TABLE reference](/sql/t-sql/statements/create-table-azure-sql-data-warehouse#DataTypes) in the CREATE TABLE statement. Minimizing the size of data types helps to improve query performance. For guidance on using data types, see [Data types](sql-data-warehouse-tables-data-types.md).

## Distributed tables
A fundamental feature of SQL Data Warehouse is the way it can store and operate on tables across 60 [distributions](massively-parallel-processing-mpp-architecture.md#distributions).  The tables are distributed using a round-robin, hash, or replication method.

### Hash-distributed tables
The hash distribution distributes rows based on the value in the distribution column. The hash-distributed table is designed to achieve high performance for query joins on large tables. There are several factors that affect the choice of the distribution column. 

For more information, see [Design guidance for distributed tables](sql-data-warehouse-tables-distribute.md).

### Replicated tables
A replicated table has a full copy of the table available on every Compute node. Queries run fast on replicated tables since joins on replicated tables do not require data movement. Replication requires extra storage, though, and is not practical for large tables. 

For more information, see [Design guidance for replicated tables](design-guidance-for-replicated-tables.md).

### Round-robin tables
A round-robin table distributes table rows evenly across all distributions. The rows are distributed randomly. Loading data into a round-robin table is fast.  However, queries can require more data movement than the other distribution methods. 

For more information, see [Design guidance for distributed tables](sql-data-warehouse-tables-distribute.md).


### Common distribution methods for tables
The table category often determines which option to choose for distributing the table. 

| Table category | Recommended distribution option |
|:---------------|:--------------------|
| Fact           | Use hash-distribution with clustered columnstore index. Performance improves when two hash tables are joined on the same distribution column. |
| Dimension      | Use replicated for smaller tables. If tables are too large to store on each Compute node, use hash-distributed. |
| Staging        | Use round-robin for the staging table. The load with CTAS is fast. Once the data is in the staging table, use INSERT...SELECT to move the data to production tables. |

## Table partitions
A partitioned table stores and performs operations on the table rows according to data ranges. For example, a table could be partitioned by day, month, or year. You can improve query performance through partition elimination, which limits a query scan to data within a partition. You can also maintain the data through partition switching. Since the data in SQL Data Warehouse is already distributed, too many partitions can slow query performance. For more information, see [Partitioning guidance](sql-data-warehouse-tables-partition.md).

## Columnstore indexes
By default, SQL Data Warehouse stores a table as a clustered columnstore index. This form of data storage achieves high data compression and query performance on large tables.  The clustered columnstore index is usually the best choice, but in some cases a clustered index or a heap is the appropriate storage structure.

For a list of columnstore features, see [What's new for columnstore indexes](/sql/relational-databases/indexes/columnstore-indexes-what-s-new). To improve columnstore index performance, see [Maximizing rowgroup quality for columnstore indexes](sql-data-warehouse-memory-optimizations-for-columnstore-compression.md).

## Statistics
The query optimizer uses column-level statistics when it creates the plan for executing a query. To improve query performance, it's important to create statistics on individual columns, especially columns used in query joins. Creating and updating statistics does not happen automatically. [Create statistics](/sql/t-sql/statements/create-statistics-transact-sql) after creating a table. Update statistics after a significant number of rows are added or changed. For example, update statistics after a load. For more information, see [Statistics guidance](sql-data-warehouse-tables-statistics.md).

## Commands for creating tables
You can create a table as a new empty table. You can also create and populate a table with the results of a select statement. The following are the T-SQL commands for creating a table.

| T-SQL Statement | Description |
|:----------------|:------------|
| [CREATE TABLE](/sql/t-sql/statements/create-table-azure-sql-data-warehouse) | Creates an empty table by defining all the table columns and options. |
| [CREATE EXTERNAL TABLE](/sql/t-sql/statements/create-external-table-transact-sql) | Creates an external table. The definition of the table is stored in SQL Data Warehouse. The table data is stored in Azure Blob storage or Azure Data Lake Store. |
| [CREATE TABLE AS SELECT](/sql/t-sql/statements/create-table-as-select-azure-sql-data-warehouse) | Populates a new table with the results of a select statement. The table columns and data types are based on the select statement results. To import data, this statement can select from an external table. |
| [CREATE EXTERNAL TABLE AS SELECT](/sql/t-sql/statements/create-external-table-as-select-transact-sql) | Creates a new external table by exporting the results of a select statement to an external location.  The location is either Azure Blob storage or Azure Data Lake Store. |

## Aligning source data with the data warehouse

Data warehouse tables are populated by loading data from another data source. To perform a successful load, the number and data types of the columns in the source data must align with the table definition in the data warehouse. Getting the data to align might be the hardest part of designing your tables. 

If data is coming from multiple data stores, you can bring the data into the data warehouse and store it in an integration table. Once data is in the integration table, you can use the power of SQL Data Warehouse to perform transformation operations. Once the data is prepared, you can insert it into production tables.

## Unsupported table features
SQL Data Warehouse supports many, but not all, of the table features offered by other databases.  The following list shows some of the table features that are not supported in SQL Data Warehouse.

- Primary key, Foreign keys, Unique, Check [Table Constraints](/sql/t-sql/statements/alter-table-table-constraint-transact-sql)

- [Computed Columns](/sql/t-sql/statements/alter-table-computed-column-definition-transact-sql)
- [Indexed Views](/sql/relational-databases/views/create-indexed-views)
- [Sequence](/sql/t-sql/statements/create-sequence-transact-sql)
- [Sparse Columns](/sql/relational-databases/tables/use-sparse-columns)
- Surrogate Keys. Implement with [Identity](sql-data-warehouse-tables-identity.md).
- [Synonyms](/sql/t-sql/statements/create-synonym-transact-sql)
- [Triggers](/sql/t-sql/statements/create-trigger-transact-sql)
- [Unique Indexes](/sql/t-sql/statements/create-index-transact-sql)
- [User-Defined Types](/sql/relational-databases/native-client/features/using-user-defined-types)

## Table size queries
One simple way to identify space and rows consumed by a table in each of the 60 distributions, is to use [DBCC PDW_SHOWSPACEUSED](/sql/t-sql/database-console-commands/dbcc-pdw-showspaceused-transact-sql).

```sql
DBCC PDW_SHOWSPACEUSED('dbo.FactInternetSales');
```

However, using DBCC commands can be quite limiting.  Dynamic management views (DMVs) show more detail than DBCC commands. Start by creating this view.

```sql
CREATE VIEW dbo.vTableSizes
AS
WITH base
AS
(
SELECT 
 GETDATE()                                                             AS  [execution_time]
, DB_NAME()                                                            AS  [database_name]
, s.name                                                               AS  [schema_name]
, t.name                                                               AS  [table_name]
, QUOTENAME(s.name)+'.'+QUOTENAME(t.name)                              AS  [two_part_name]
, nt.[name]                                                            AS  [node_table_name]
, ROW_NUMBER() OVER(PARTITION BY nt.[name] ORDER BY (SELECT NULL))     AS  [node_table_name_seq]
, tp.[distribution_policy_desc]                                        AS  [distribution_policy_name]
, c.[name]                                                             AS  [distribution_column]
, nt.[distribution_id]                                                 AS  [distribution_id]
, i.[type]                                                             AS  [index_type]
, i.[type_desc]                                                        AS  [index_type_desc]
, nt.[pdw_node_id]                                                     AS  [pdw_node_id]
, pn.[type]                                                            AS  [pdw_node_type]
, pn.[name]                                                            AS  [pdw_node_name]
, di.name                                                              AS  [dist_name]
, di.position                                                          AS  [dist_position]
, nps.[partition_number]                                               AS  [partition_nmbr]
, nps.[reserved_page_count]                                            AS  [reserved_space_page_count]
, nps.[reserved_page_count] - nps.[used_page_count]                    AS  [unused_space_page_count]
, nps.[in_row_data_page_count] 
    + nps.[row_overflow_used_page_count] 
    + nps.[lob_used_page_count]                                        AS  [data_space_page_count]
, nps.[reserved_page_count] 
 - (nps.[reserved_page_count] - nps.[used_page_count]) 
 - ([in_row_data_page_count] 
         + [row_overflow_used_page_count]+[lob_used_page_count])       AS  [index_space_page_count]
, nps.[row_count]                                                      AS  [row_count]
from 
    sys.schemas s
INNER JOIN sys.tables t
    ON s.[schema_id] = t.[schema_id]
INNER JOIN sys.indexes i
    ON  t.[object_id] = i.[object_id]
    AND i.[index_id] <= 1
INNER JOIN sys.pdw_table_distribution_properties tp
    ON t.[object_id] = tp.[object_id]
INNER JOIN sys.pdw_table_mappings tm
    ON t.[object_id] = tm.[object_id]
INNER JOIN sys.pdw_nodes_tables nt
    ON tm.[physical_name] = nt.[name]
INNER JOIN sys.dm_pdw_nodes pn
    ON  nt.[pdw_node_id] = pn.[pdw_node_id]
INNER JOIN sys.pdw_distributions di
    ON  nt.[distribution_id] = di.[distribution_id]
INNER JOIN sys.dm_pdw_nodes_db_partition_stats nps
    ON nt.[object_id] = nps.[object_id]
    AND nt.[pdw_node_id] = nps.[pdw_node_id]
    AND nt.[distribution_id] = nps.[distribution_id]
LEFT OUTER JOIN (select * from sys.pdw_column_distribution_properties where distribution_ordinal = 1) cdp
    ON t.[object_id] = cdp.[object_id]
LEFT OUTER JOIN sys.columns c
    ON cdp.[object_id] = c.[object_id]
    AND cdp.[column_id] = c.[column_id]
)
, size
AS
(
SELECT
   [execution_time]
,  [database_name]
,  [schema_name]
,  [table_name]
,  [two_part_name]
,  [node_table_name]
,  [node_table_name_seq]
,  [distribution_policy_name]
,  [distribution_column]
,  [distribution_id]
,  [index_type]
,  [index_type_desc]
,  [pdw_node_id]
,  [pdw_node_type]
,  [pdw_node_name]
,  [dist_name]
,  [dist_position]
,  [partition_nmbr]
,  [reserved_space_page_count]
,  [unused_space_page_count]
,  [data_space_page_count]
,  [index_space_page_count]
,  [row_count]
,  ([reserved_space_page_count] * 8.0)                                 AS [reserved_space_KB]
,  ([reserved_space_page_count] * 8.0)/1000                            AS [reserved_space_MB]
,  ([reserved_space_page_count] * 8.0)/1000000                         AS [reserved_space_GB]
,  ([reserved_space_page_count] * 8.0)/1000000000                      AS [reserved_space_TB]
,  ([unused_space_page_count]   * 8.0)                                 AS [unused_space_KB]
,  ([unused_space_page_count]   * 8.0)/1000                            AS [unused_space_MB]
,  ([unused_space_page_count]   * 8.0)/1000000                         AS [unused_space_GB]
,  ([unused_space_page_count]   * 8.0)/1000000000                      AS [unused_space_TB]
,  ([data_space_page_count]     * 8.0)                                 AS [data_space_KB]
,  ([data_space_page_count]     * 8.0)/1000                            AS [data_space_MB]
,  ([data_space_page_count]     * 8.0)/1000000                         AS [data_space_GB]
,  ([data_space_page_count]     * 8.0)/1000000000                      AS [data_space_TB]
,  ([index_space_page_count]  * 8.0)                                   AS [index_space_KB]
,  ([index_space_page_count]  * 8.0)/1000                              AS [index_space_MB]
,  ([index_space_page_count]  * 8.0)/1000000                           AS [index_space_GB]
,  ([index_space_page_count]  * 8.0)/1000000000                        AS [index_space_TB]
FROM base
)
SELECT * 
FROM size
;
```

### Table space summary

This query returns the rows and space by table.  It allows you to see which tables are your largest tables and whether they are round-robin, replicated, or hash -distributed.  For hash-distributed tables, the query shows the distribution column.  

```sql
SELECT 
     database_name
,    schema_name
,    table_name
,    distribution_policy_name
,      distribution_column
,    index_type_desc
,    COUNT(distinct partition_nmbr) as nbr_partitions
,    SUM(row_count)                 as table_row_count
,    SUM(reserved_space_GB)         as table_reserved_space_GB
,    SUM(data_space_GB)             as table_data_space_GB
,    SUM(index_space_GB)            as table_index_space_GB
,    SUM(unused_space_GB)           as table_unused_space_GB
FROM 
    dbo.vTableSizes
GROUP BY 
     database_name
,    schema_name
,    table_name
,    distribution_policy_name
,      distribution_column
,    index_type_desc
ORDER BY
    table_reserved_space_GB desc
;
```

### Table space by distribution type

```sql
SELECT 
     distribution_policy_name
,    SUM(row_count)                as table_type_row_count
,    SUM(reserved_space_GB)        as table_type_reserved_space_GB
,    SUM(data_space_GB)            as table_type_data_space_GB
,    SUM(index_space_GB)           as table_type_index_space_GB
,    SUM(unused_space_GB)          as table_type_unused_space_GB
FROM dbo.vTableSizes
GROUP BY distribution_policy_name
;
```

### Table space by index type

```sql
SELECT 
     index_type_desc
,    SUM(row_count)                as table_type_row_count
,    SUM(reserved_space_GB)        as table_type_reserved_space_GB
,    SUM(data_space_GB)            as table_type_data_space_GB
,    SUM(index_space_GB)           as table_type_index_space_GB
,    SUM(unused_space_GB)          as table_type_unused_space_GB
FROM dbo.vTableSizes
GROUP BY index_type_desc
;
```

### Distribution space summary

```sql
SELECT 
    distribution_id
,    SUM(row_count)                as total_node_distribution_row_count
,    SUM(reserved_space_MB)        as total_node_distribution_reserved_space_MB
,    SUM(data_space_MB)            as total_node_distribution_data_space_MB
,    SUM(index_space_MB)           as total_node_distribution_index_space_MB
,    SUM(unused_space_MB)          as total_node_distribution_unused_space_MB
FROM dbo.vTableSizes
GROUP BY     distribution_id
ORDER BY    distribution_id
;
```

## Next steps
After creating the tables for your data warehouse, the next step is to load data into the table.  For a loading tutorial, see [Loading data to SQL Data Warehouse](load-data-wideworldimportersdw.md).
