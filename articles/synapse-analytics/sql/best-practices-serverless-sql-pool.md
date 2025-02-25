---
title: Best practices for serverless SQL pool
description: Recommendations and best practices for working with serverless SQL pool. 
services: synapse-analytics
author: filippopovic
manager: craigg
ms.service: synapse-analytics
ms.topic: conceptual
ms.subservice: sql
ms.date: 05/01/2020
ms.author: fipopovi
ms.reviewer: jrasnick
---

# Best practices for serverless SQL pool in Azure Synapse Analytics

In this article, you'll find a collection of best practices for using serverless SQL pool. Serverless SQL pool is a resource in Azure Synapse Analytics. If you're working with dedicated SQL pool, see [Best practices for dedicated SQL pools](best-practices-dedicated-sql-pool.md) for specific guidance.

Serverless SQL pool allows you to query files in your Azure storage accounts. It doesn't have local storage or ingestion capabilities. All files that the queries target are external to serverless SQL pool. Everything related to reading files from storage may have an impact on query performance.

Some generic guidelines are:
- Make sure that your client applications are collocated with the serverless SQL pool.
  - If you are using client applications outside of Azure (for example Power BI Desktop, SSMS, ADS), make sure that you are using the serverless pool in a region that is close to your client computer.
- Make sure that the storage (Azure Data Lake, Cosmos DB) and serverless SQL pool are in the same region.
- Try to [optimize storage layout](#prepare-files-for-querying) using partitioning and keeping your files in the range between 100 MB and 10 GB.
- If you are returning a large number of results, make sure that you are using SSMS or ADS and not Synapse Studio. Synapse Studio is a web tool that is not designed for large result-sets. 
- If you are filtering results by string column, try to use a `BIN2_UTF8` collation.
- Try to cache the results on the client side by using Power BI import mode or Azure Analysis Services, and periodically refresh them. The serverless SQL pools cannot provide interactive experience in Power BI Direct Query mode if you are using complex queries or processing a large amount of data.

## Client applications and network connections

Make sure that your client application is connected to the closest possible Synapse workspace with the optimal connection.
- Colocate a client application with the Synapse workspace. If you are using applications such as Power BI or Azure Analysis Service, make sure that they are in the same region where you have placed your Synapse workspace. If needed, create the separate workspaces that are paired with your client applications. Placing a client application and the Synapse workspace in different region could cause bigger latency and slower streaming of results.
- If you are reading data from your on-premises application, make sure that the Synapse workspace is in the region that is close to your location.
- Make sure that you don't have some network bandwidth issues while reading a large amount of data.
- Do not use Synapse studio to return a large amount of data. Synapse studio is web tool that uses HTTPS protocol to transfer data. Use Azure Data Studio or SQL Server Management Studio to read a large amount of data.

## Storage and content layout

### Colocate your storage and serverless SQL pool

To minimize latency, colocate your Azure storage account or CosmosDB analytic storage and your serverless SQL pool endpoint. Storage accounts and endpoints provisioned during workspace creation are located in the same region.

For optimal performance, if you access other storage accounts with serverless SQL pool, make sure they're in the same region. If they aren't in the same region, there will be increased latency for the data's network transfer between the remote region and the endpoint's region.

### Azure Storage throttling

Multiple applications and services might access your storage account. Storage throttling occurs when the combined IOPS or throughput generated by applications, services, and serverless SQL pool workload exceed the limits of the storage account. As a result, you'll experience a significant negative effect on query performance.

When throttling is detected, serverless SQL pool has built-in handling to resolve it. Serverless SQL pool will make requests to storage at a slower pace until throttling is resolved.

> [!TIP]
> For optimal query execution, don't stress the storage account with other workloads during query execution.

### Azure AD Pass-through performance

Serverless SQL pool allows you to access files in storage by using Azure Active Directory (Azure AD) Pass-through or SAS credentials. You might experience slower performance with Azure AD Pass-through than you would with SAS.

If you need better performance, try using SAS credentials to access storage.

### Prepare files for querying

If possible, you can prepare files for better performance:

- Convert large CSV and JSON to Parquet. Parquet is a columnar format. Because it's compressed, its file sizes are smaller than CSV or JSON files that contain the same data. Serverless SQL pool is able to skip the columns and rows that are not needed in query if you are reading Parquet files. Serverless SQL pool will need less time and fewer storage requests to read it.
- If a query targets a single large file, you'll benefit from splitting it into multiple smaller files.
- Try to keep your CSV file size between 100 MB and 10 GB.
- It's better to have equally sized files for a single OPENROWSET path or an external table LOCATION.
- Partition your data by storing partitions to different folders or file names. See [Use filename and filepath functions to target specific partitions](#use-filename-and-filepath-functions-to-target-specific-partitions).

### Colocate your CosmosDB analytical storage and serverless SQL pool

Make sure that your CosmosDB analytical storage is placed in the same region as Synapse workspace. Cross-region queries might cause huge latencies. Use region property in the connection string to explicitly specify the region where analytical store is placed (see [query CosmosDb using serverless SQL pool](query-cosmos-db-analytical-store.md#overview)):

```
'account=<database account name>;database=<database name>;region=<region name>'
```

## CSV optimizations

### Use PARSER_VERSION 2.0 to query CSV files

You can use a performance-optimized parser when you query CSV files. For details, see [PARSER_VERSION](develop-openrowset.md).

### Manually create statistics for CSV files

Serverless SQL pool relies on statistics to generate optimal query execution plans. Statistics will be automatically created for columns in Parquet files when needed. At this moment, statistics are not automatically created for columns in CSV files and you should create statistics manually for columns that you use in queries, particularly those used in DISTINCT, JOIN, WHERE, ORDER BY and GROUP BY. Check [statistics in serverless SQL pool](develop-tables-statistics.md#statistics-in-serverless-sql-pool) for details.


## Data types

### Use appropriate data types

The data types you use in your query affect performance. You can get better performance if you follow these guidelines: 

- Use the smallest data size that will accommodate the largest possible value.
  - If the maximum character value length is 30 characters, use a character data type of length 30.
  - If all character column values are of fixed size, use **char** or **nchar**. Otherwise, use **varchar** or **nvarchar**.
  - If the maximum integer column value is 500, use **smallint** because it's the smallest data type that can accommodate this value. You can find integer data type ranges in [this article](/sql/t-sql/data-types/int-bigint-smallint-and-tinyint-transact-sql?view=azure-sqldw-latest&preserve-view=true).
- If possible, use **varchar** and **char** instead of **nvarchar** and **nchar**.
- Use integer-based data types if possible. SORT, JOIN, and GROUP BY operations complete faster on integers than on character data.
- If you're using schema inference, [check inferred data types](#check-inferred-data-types).

### Check inferred data types

[Schema inference](query-parquet-files.md#automatic-schema-inference) helps you quickly write queries and explore data without knowing file schemas. The cost of this convenience is that inferred data types may be larger than the actual data types. This happens when there isn't enough information in the source files to make sure the appropriate data type is used. For example, Parquet files don't contain metadata about maximum character column length. So serverless SQL pool infers it as varchar(8000).

You can use [sp_describe_first_results_set](/sql/relational-databases/system-stored-procedures/sp-describe-first-result-set-transact-sql?view=sql-server-ver15&preserve-view=true) to check the resulting data types of your query.

The following example shows how you can optimize inferred data types. This procedure is used to show the inferred data types: 
```sql  
EXEC sp_describe_first_result_set N'
	SELECT
        vendor_id, pickup_datetime, passenger_count
	FROM 
		OPENROWSET(
        	BULK ''https://sqlondemandstorage.blob.core.windows.net/parquet/taxi/*/*/*'',
	        FORMAT=''PARQUET''
    	) AS nyc';
```

Here's the result set:

|is_hidden|column_ordinal|name|system_type_name|max_length|
|----------------|---------------------|----------|--------------------|-------------------|
|0|1|vendor_id|varchar(8000)|8000|
|0|2|pickup_datetime|datetime2(7)|8|
|0|3|passenger_count|int|4|

After you know the inferred data types for the query, you can specify appropriate data types:

```sql  
SELECT
    vendorID, tpepPickupDateTime, passengerCount
FROM 
	OPENROWSET(
		BULK 'https://azureopendatastorage.blob.core.windows.net/nyctlc/yellow/puYear=2018/puMonth=*/*.snappy.parquet',
		FORMAT='PARQUET'
    ) 
	WITH (
		vendor_id varchar(4), -- we used length of 4 instead of the inferred 8000
		pickup_datetime datetime2,
		passenger_count int
	) AS nyc;
```

## Filter optimization

### Push wildcards to lower levels in the path

You can use wildcards in your path to [query multiple files and folders](query-data-storage.md#query-multiple-files-or-folders). Serverless SQL pool lists files in your storage account, starting from the first * using storage API. It eliminates files that don't match the specified path. Reducing the initial list of files can improve performance if there are many files that match the specified path up to the first wildcard.

### Use filename and filepath functions to target specific partitions

Data is often organized in partitions. You can instruct serverless SQL pool to query particular folders and files. Doing so will reduce the number of files and the amount of data the query needs to read and process. An added bonus is that you'll achieve better performance.

For more information, read about the [filename](query-data-storage.md#filename-function) and [filepath](query-data-storage.md#filepath-function) functions and see the examples for [querying specific files](query-specific-files.md).

> [!TIP]
> Always cast the results of the filepath and filename functions to appropriate data types. If you use character data types, be sure to use the appropriate length.

> [!NOTE]
> Functions used for partition elimination, filepath and filename, aren't currently supported for external tables, other than those created automatically for each table created in Apache Spark for Azure Synapse Analytics.

If your stored data isn't partitioned, consider partitioning it. That way you can use these functions to optimize queries that target those files. When you [query partitioned Apache Spark for Azure Synapse tables](develop-storage-files-spark-tables.md) from serverless SQL pool, the query will automatically target only the necessary files.

### Use proper collation to utilize predicate pushdown for character columns

Data in parquet file is organized in row groups. Serverless SQL pool skips row groups based on specified predicate in WHERE clause and thus reduce IO which results in increased query performance. 

Please note that predicate pushdown for character columns in parquet files is supported for Latin1_General_100_BIN2_UTF8 collation only. You can specify collation for particular column using WITH clause. If you don't specify this collation using WITH clause, database collation will be used.

## Optimize repeating queries

### Use CETAS to enhance query performance and joins

[CETAS](develop-tables-cetas.md) is one of the most important features available in serverless SQL pool. CETAS is a parallel operation that creates external table metadata and exports the SELECT query results to a set of files in your storage account.

You can use CETAS to materialize frequently used parts of queries, like joined reference tables, to a new set of files. You can then join to this single external table instead of repeating common joins in multiple queries.

As CETAS generates Parquet files, statistics will be automatically created when the first query targets this external table, resulting in improved performance for subsequent queries targeting table generated with CETAS.

## Next steps

Review the [troubleshooting](resources-self-help-sql-on-demand.md) article for solutions to common problems. If you're working with dedicated SQL pool rather than serverless SQL pool, see [Best practices for dedicated SQL pools](best-practices-dedicated-sql-pool.md) for specific guidance.
