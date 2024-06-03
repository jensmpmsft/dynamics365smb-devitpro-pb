---
title: Performance article for developers
description: Provides information for developers to help improve performance in Business Central
ms.custom: bap-template
ms.date: 10/11/2023
ms.reviewer: jswymer
ms.topic: conceptual
author: KennieNP
ms.author: kepontop
---

# Performance articles for developers

In this article, you can read about ways to tune performance when developing for [!INCLUDE[prod_short](../developer/includes/prod_short.md)].

- [Writing efficient pages](performance-developer.md#writing-efficient-pages)  
- [Writing efficient Web Services](performance-developer.md#writing-efficient-web-services)  
- [Writing efficient reports](performance-developer.md#writing-efficient-al-reports)  
- [AL performance patterns](performance-developer.md#al-performance-patterns)  
- [Efficient Data access](performance-developer.md#efficient-data-access)  
- [Testing and validating performance](performance-developer.md#testing-and-validating-performance)  
- [Tuning the Development Environment](performance-developer.md#tuning-the-development-environment)  
- [Using the AL Profiler to analyze performance](../developer/devenv-al-profiler-overview.md)

## Writing efficient pages

There are many patterns that a developer can use to get a page to load faster. Consider the following patterns:

- Avoid unnecessary recalculation
- Do less
- Offloading the UI thread

### Pattern - Avoid unnecessary recalculation

To avoid unnecessary recalculation of expensive results, consider caching the data and refresh the cache regularly. Let's say you want to show the top five open sales orders or a VIP customers list on the role center. The content of such a list probably doesn't change significantly every hour. There's no need to calculate that from raw data every time the page is loaded. Instead, create a table that can contain the calculated data and refresh every hour/day using a background job.

Another example of unexpected recalculation is when using query objects. In contrast to using the record API, query results aren't cached in the primary key cache in the [!INCLUDE[prod_short](../developer/includes/prod_short.md)] server. Any use of a query object always goes to the database. So, sometimes it's faster to not use a query object. 

### Pattern - Do less

One way to speed up things is to reduce the amount of work that the system must do. For example, to reduce slowness of role centers, consider how many page parts are needed for the user. Another benefit of a simple page with few UI elements can also be ease of use and navigation.

Remove calculated fields from lists if they aren't needed, especially on larger tables. Setting the field's Enabled or Visible properties to false isn't enough. The field definition needs to be removed from the page or page extension definition. Also, if indexing is inadequate, calculated fields can significantly slow down a list page.

Consider creating dedicated lookup pages instead of the normal pages when adding a lookup (the one that looks like a dropdown) from a field. Default list pages run all triggers and fact boxes even if they aren't shown in the lookup. For example, [!INCLUDE[prod_short](../developer/includes/prod_short.md)] 2019 release wave 1 added dedicated lookup pages for Customer, Vendor, and Item to the Base Application.
 
### Pattern - Offloading the UI thread

To get to a responsive UI fast, consider using Page Background Tasks for calculated values, for example, the values shown in cues.

For more information about Page Background Tasks, see [Page Background Tasks](../developer/devenv-page-background-tasks.md).

### Making Edit-in-Excel faster

The **Edit in Excel** feature uses UI pages exposed through OData, which means that triggers need to be run for all records returned from the [!INCLUDE[prod_short](../developer/includes/prod_short.md)] server to Excel. As a developer, you need to make your AL code conditional on the ClientType. Specifically, avoid updating fact boxes, avoid calculation, and avoid defaulting logic.

## Writing efficient Web Services

[!INCLUDE[prod_short](../developer/includes/prod_short.md)]  supports for Web services to make it easier to integrate with external systems. As a developer, you need to think about performance of web services both seen from the [!INCLUDE[prod_short](../developer/includes/prod_short.md)] server (the endpoint) and as seen from the consumer (the client). 

### Endpoint performance

#### General anti-patterns (don't do this)

[!INCLUDE[perf_ws_antipatterns](../includes/include-webservices-performance-anti-patterns.md)]

#### Performance patterns (do this)

[!INCLUDE[perf_ws_patterns](../includes/include-webservices-performance-patterns.md)]

### API/OData client performance patterns

[!INCLUDE[perf_ws_odata_patterns](../includes/include-webservices-odata-performance-patterns.md)]

### How to handle large throughput of web service calls

[!INCLUDE[perf_ws_throughput](../includes/include-webservices-throughput-performance-patterns.md)]

## Performance patterns for business intelligence and reporting

### Writing efficient AL reports

[!INCLUDE[report_performance](../includes/include-report-performance.md)]

For more information on how to use telemetry to analyze the performance of reports, see [Report performance](../developer/devenv-report-performance.md)

### Stop using report properties **DefaultLayout**, **ExcelLayout**, **RDLLayout**, and **WordLayout**

[!INCLUDE[single_layouts](../developer/includes/include-single-layout-obsolete.md)]

### Loading data efficiently to Power BI

[!INCLUDE[perf_ws_pbi_patterns](../includes/include-webservices-pbi-performance-patterns.md)]

### Efficient extracts to data lakes or data warehouses

When establishing a data lake or a data warehouse, you typically need to do two types of data extraction:

1. A historical load (all data from a given point-in-time)
2. Delta loads (what's changed since the historical load)

The fastest (and least disruptive) way to get a historical load from [!INCLUDE[prod_short](../developer/includes/prod_short.md)] online is to get a database export as a BACPAC file (using the [!INCLUDE[prod_short](../developer/includes/prod_short.md)] admin center) and restore it in Azure SQL Database or on a SQL Server. For on-premises installations, you can just take a backup of the tenant database.

The fastest (and least disruptive) way to get delta loads from [!INCLUDE[prod_short](../developer/includes/prod_short.md)] online is to set up API queries configured with read-scaleout and use the data audit field **LastModifiedOn** (introduced in version 17.0) on filters.

For more information, see [Extract data from Business Central](../developer/devenv-extract-data.md).

## AL performance patterns

Knowledge about different AL performance patterns can greatly improve the performance of the code you write. In this section, we describe the following patterns and their impact on performance.

- [Use built-in data structures](#builtindatastructure)  
- [Run async (and parallelize)](#runasync)  
- [Use set-based methods instead of looping](#setbasedmethods)  

### <a name="builtindatastructure"></a>Pattern - Use built-in data structures

AL comes with built-in data structures that have been optimized for performance and server resource consumption. Make sure that you're familiar with them to make your AL code as efficient as possible.  

When working with strings, make sure to use the `TextBuilder` data type and not repeated use of the `+=` operator on a `Text` variable. General guidance is to use a `Text` data type if you concatenate fewer than five strings (here the internal allocation of a `TextBuilder` and the final `ToText` invocation is more expensive). If you need to concatenate five strings or more or concatenate strings in a loop, then `TextBuilder` is faster. Also, use a `TextBuilder` data type instead of `BigText` when possible. For more information, see [TextBuilder Data Type](../developer/methods-auto/textbuilder/textbuilder-data-type.md). 

If you need a key-value data structure that is optimized for fast lookups, use a `Dictionary` data type. For more information, see [Dictionary Data Type](../developer/methods-auto/dictionary/dictionary-data-type.md).

Use a `List` data type if you need an unbound "array" (where you would previously create a temporary table object). For more information, see [List Data Type](../developer/methods-auto/list/list-data-type.md).

Use the `Media` or `Mediaset` data types instead of the `Blob` data type. The `Media` and `MediaSet` data types have a couple advantages over the `Blob` data type when working with images. First of all, a thumbnail version of the image is generated when you save the data. You can use the thumbnail when loading a page and then load the larger image asynchronously using a page background task. Second, data for `Media` and `MediaSet` data types is cached on the client. Data for the `Blob` data type is never cached on the server. It's always fetched from the database.

### <a name="runasync"></a>Pattern - Run async (and parallelize)

[!INCLUDE[async-overview](../includes/include-async-overview.md)]

### <a name="setbasedmethods"></a>Pattern - Use set-based methods instead of looping

The AL methods such as `FindSet`, `CalcFields`, `CalcSums`, and `SetAutoCalcFields` are examples of set-based operations that are faster than looping over a result set and do the calculation for each row.

- [CalcFields, CalcSums, and Count](../administration/optimize-sql-al-database-methods-and-performance-on-server.md#calc) 
- [FindSet Method](../developer/methods-auto/recordref/recordref-findset-method.md)

One common use of the `CalcSums` method is to efficiently calculate totals. 

Try to minimize work done in the `OnAfterGetRecord` trigger code. Common performance coding patterns in this trigger are:

- Avoiding `CalcFields` calls. Defer them until the end.
- Avoiding `CurrPage.Update()` calls.
- Avoiding repeated calculations. Move them outside the loop, if possible. 
- Avoid changing filters. This pattern requires the server to throw away the result set.
- Never do any database writes here. With more than one user on the system, this gives database locking issues and even deadlock errors.

Consider using a query object if you want to use a set-based coding paradigm. These pros and cons for using query objects:

|Pros for using a query object|Cons for using a query object | 
|-----------------------------|------------------------------|
| - Bypasses the AL record API where server reads all fields. <br> - With a covering index, you can get fast read performance for tables with many fields. <br> - Can join multiple tables. | - Query object result sets aren't cached in the servers primary key (data) cache. <br> - No writes are allowed. <br> - You can't add a page on a query object. |

Read more about query objects here:

- [Using Queries Instead of Record Variables](../developer/devenv-query-using-instead-record-variables.md)  
- [Query object](../developer/devenv-query-object.md)  
- [Query overview](../developer/devenv-query-overview.md)  
- [TopNumberOfRows Property](../developer/properties/devenv-topnumberofrows-property.md)  
- [Query Objects and Performance](../administration/optimize-sql-query-objects-and-performance.md)

### <a name="partialrecords"></a>Pattern - Use partial records when looping over data, in reports, or when table extension fields aren't needed

When writing AL code for which the fields needed on a record or recordref are known, you can use the partial records capability to only load out these fields initially. The remaining fields are still accessible, but they'll be loaded as needed.

Partial records improve performance in two major ways. First, they limit the fields that need to be loaded from the database. Loading more fields leads to more data being read, sent over the connection, and created on the record. Second, partial records limit the number of table extensions that need to be joined.

> [!NOTE]  
> With [!INCLUDE [prod_short](../developer/includes/prod_short.md)] 2023 release wave 2, the data model for table extensions has changed. In this new model, data from all table extensions on a table are stored in the same companion table. This means that there will be at most one SQL join involved when doing data operations on the table (as seen from AL). Partial records can still eliminate the join to the single companion table in [!INCLUDE [prod_short](../developer/includes/prod_short.md)] 2023 release wave 2, if all loaded fields reside in the base table.

The performance gains compound when looping over many records, because both effects scale with the number of rows loaded.

For more information, see [Using Partial Records](../developer/devenv-partial-records.md).

### Table extension impact on performance (for [!INCLUDE [prod_short](../developer/includes/prod_short.md)] 2023 release wave 1 and earlier)

> [!NOTE]  
> In version 23, the data model for table extensions was changed. In this new model, data from all table extensions on a table are stored in the same companion table. This means that there will be at most one SQL join involved when doing data operations on the table (as seen from AL).

Table extensions are separate tables in the database and therefore need to be joined together in the data stack when accessed via a record. With tables extensions being stored individually, the amount of joins necessary grows with the number of table extensions extending a table. Together with the current inability to define indexes that span base and extension fields, one should avoid splitting one's code into too many table extensions.

With central tables to the application, such as General Ledger Entry (G/L Entry), one should be extra cautious adding table extensions since these tables are frequently used throughout the application.

The adverse affects of many table extensions can be mitigated with the application of partial records, see [Using Partial Records](../developer/devenv-partial-records.md). However, since the developer may not have ownership of all executed code, and therefore isn't able to apply partial records everywhere, the above recommendation still stands.

An alternative approach when doing data modeling for extending a table with new fields is to use a related table and define a FlowField on the base table.

Here are the pros and cons of the two data models:

|Data model for extending a table | Properties |
|---------------------------------|-------------|
|Table extension | Fields can be added to lists and are searchable. <br> Always loaded with the base table. <br> Expensive at runtime but easy to use. <br> Use only for critical fields. |
| Related tables | Need to set up table relations. <br> Dedicated page for editing. <br> Requires flow field to be shown in lists. <br> Doesn't affect performance of base table. <br> Excellent for FactBoxes. | 



### Limit your event subscriptions

The following are best practices for getting performant events:

- There's no significant cost of having a publisher defined.
- Static automatic has a cost over manually binding (there's an overhead of creating and disposing objects).
- Codeunit size of the subscriber matters. Try to have smaller codeunits.
- Use single instance codeunits for subscribers, if possible.

Table events change the behavior of SQL optimizations on the [!INCLUDE[server](../developer/includes/server.md)] in the following ways:

- The [!INCLUDE[server](../developer/includes/server.md)] will issue SQL update/delete statements row in a for loop rather than one SQL statement.
- They impact `ModifyAll` and `DeleteAll` methods that normally do bulk SQL operations to be forced to do single row operations.

### Outgoing web service calls block AL execution

[!INCLUDE[httpclientPerformance](../includes/performance-outgoing-http.md)] 

### Limit work done in login event subscribers
[!INCLUDE[login_performance](../includes/include-telemetry-login-performance.md)])

## Efficient data access 

Many performance issues are related to how data is defined, accessed, and modified. It's important to know how concepts in AL metadata and the AL language translate to their counterparts in SQL.  
  
### Tables and keys 

Many performance issues can be traced back to missing indexes (also called keys in [!INCLUDE[prod_short](../developer/includes/prod_short.md)]), but index design is often not a key skill for AL developers. For best performance, even with large amounts of data, it's imperative to design appropriate indexes according to the way your code will access data. 

These articles on indexing are worth knowing as an AL developer:

- [About Table Keys](../developer/devenv-table-keys.md)
- [Table Keys and Performance in Business Central](../administration/optimize-sql-table-keys-and-performance.md)  
- [About SQL Server indexes](/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described)
- [Missing Indexes](../administration/database-missing-indexes.md)

Indexes have a cost to update, so it's recommended to not add too many of them on a table. 

### Using data audit fields to only read recent data

Every table in [!INCLUDE[prod_short](../developer/includes/prod_short.md)]) includes the following two system fields, which can be used for filtering records:

- `SystemCreatedAt`
- `SystemModifiedAt`

One example is to use the system field `SystemModifiedAt` to implement delta reads. For more information about system fields, see [System Fields](../developer/devenv-table-system-fields.md).  

### Non-clustered Columnstore Indexes (NCCI)

Starting in the 2021 release wave 2 of [!INCLUDE[prod_short](../developer/includes/prod_short.md)], non-clustered columnstore indexes (sometimes referred to as NCCIs) are supported on tables. 

You can use a non-clustered columnstore index to efficiently run real-time operational analytics on the [!INCLUDE[prod_short](../developer/includes/prod_short.md)] database without the need to define SIFT indexes up front (and without the locking issues that SIFT indexes sometimes impose on the system.)

Read more about non-clustered columnstore indexes here:

- [ColumnStoreIndex table property](../developer/properties/devenv-columnstoreindex-property.md)
- [Columnstore indexes overview](/sql/relational-databases/indexes/columnstore-indexes-overview)

### SumIndexField Technology (SIFT)

SumIndexField Technology (SIFT) lets you quickly calculate the sums of numeric data type columns in tables, even in tables with thousands of records. The data type includes Decimal, Integer, BigInteger, and Duration. SIFT optimizes the performance of FlowFields and query results in a [!INCLUDE[prod_short](../developer/includes/prod_short.md)] application. 

Ensure appropriate SIFT indices for all FlowFields of type sum or count. 

Read more about SIFT here:

- [SumIndexField Technology (SIFT)](../developer/devenv-sift-technology.md)  
- [SIFT and Performance](../developer/devenv-sift-performance.md)  
- [Tuning and Tracing](../developer/devenv-sift-tuning-and-tracing.md)  
- [SIFT and SQL Server](../developer/devenv-sift-and-sql-server.md)  

The following article can help you find missing SIFT indexes on FlowFields:

[Troubleshooting: Long Running SQL Queries Involving FlowFields by Disabling SmartSQL](../administration/troubleshooting-queries-involving-flowfields-by-disabling-smartsql.md).

### How AL relates to SQL 

The AL programming language, to some degree, hides how data is read and written to the database. To effectively code for performance, you need to know how AL statements translate to the equivalent SQL statements.

The following articles cover how AL relates to SQL:

- [AL Database Methods and Performance on SQL Server](../administration/optimize-sql-al-database-methods-and-performance-on-server.md)  
- [Data Access](../administration/optimize-sql-data-access.md)  
- [Data read/write performance](../administration/optimize-sql-data-access.md#readwrite)  
- [Bulk Inserts](../administration/optimize-sql-bulk-inserts.md)  

### How to get insights into how AL translates to SQL 

If you want to track how [!INCLUDE[server](../developer/includes/server.md)] translates AL statements to SQL statements, use either database statistics in the AL debugger or telemetry on long running queries.

Read more here:

- [About database statistics in the AL debugger](../developer/devenv-debugging.md#DebugSQL)
- [Telemetry on Long Running SQL Queries](../administration/monitor-long-running-sql-queries-event-log.md)

### How to reduce database locking
Sometimes, performance issues aren't due to resource starvation, but due to processes waiting for other processes to release locks on shared objects. When AL code needs to update data, it's customary to take a database lock on it to ensure that other processes don't change the data at the same time. 

Using the `Record.LockTable` method, this will apply the `WITH (updlock)` hint on all subsequent read calls to the database against the table of the record called on, until the transaction is committed. Hence, it's good practice to defer the `Record.LockTable` call as late as possible in your AL code, to make sure that only the data that is in scope for being updated, is locked. Read more here: [Record.LockTable Method](../developer/methods-auto/record/record-locktable-method.md)

Some tips for avoiding locking:

-	Read setup information before starting write transactions
-	If possible, limit the time you hold locks
-	If possible, limit transaction size (divide into smaller operations that can be committed)
-	Make sure you have indexes on ranges you update
-	Locking is much less of an issue if you have a logical separation across companies
-	Set record instance isolation level on transactions to isolate them from other transactions to prevent problems in concurrent situations. [Learn more about record instance isolation level](../developer/devenv-read-isolation.md).  

#### Database locking caused by web service calls

Don't insert child records belonging to the same parent record in parallel. This condition causes locks on both the parent table and the integration record table because parallel calls try to update the same parent record. The solution is to wait for the first call to finish or use OData `$batch`, which will make sure calls get run one after another.

#### Non-blocking number sequences
If you need a fast, non-blocking number sequence that can be used from AL, refer to the number sequence object type. Use a number sequence object if you: 

- Don't want to use a number series 
- Can accept holes in the number range

For more information, see [NumberSequence Data Type](../developer/methods-auto/numbersequence/numbersequence-data-type.md).

#### Analyzing database locks

There are two tools that you can use to analyze database locks happening in the environment: the **Database Locks** page, and database lock timeout telemetry.

The **Database Locks** page gives a snapshot of all current database locks in SQL Server. It provides information like the table and database resource affected by the lock, and sometimes also the AL object or method that ran the transaction that caused the lock. These details can help you better understand the locking condition.

Database lock timeout telemetry gathers information about database locks that have timed out. The telemetry data allows you to troubleshoot what caused these locks.

Read more here:
- [Viewing Database Locks](/dynamics365/business-central/admin-view-database-locks)
- [Monitoring SQL Database Locks](../administration/monitor-database-locks.md)
- [Analyzing Database Lock Timeout Trace Telemetry](../administration/telemetry-database-locks-trace.md)

### Using Read-Scale Out

[!INCLUDE[prod_short](../developer/includes/prod_short.md)] supports the **Read Scale-Out** feature in Azure SQL Database and SQL Server. **Read Scale-Out** is used to load-balance analytical workloads in the database that only read data.  **Read Scale-Out** is built in as part of [!INCLUDE[prod_short](../developer/includes/prod_short.md)] online, but it can also be enabled for on-premises.

**Read Scale-Out** applies to queries, reports, or API pages. With these objects, instead of sharing the primary, they can be set up to run against a read-only replica. This setup   essentially isolates them from the main read-write workload so that they won't affect the performance of business processes.

As a developer, you control **Read Scale-Out** on report, API page, and query objects by using the [DataAccessIntent property](../developer/properties/devenv-dataaccessintent-property.md). For more information, see [Using Read Scale-Out for Better Performance](../administration/database-read-scale-out-overview.md).

## Testing and validating performance 

It's imperative to test and validate a [!INCLUDE[prod_short](../developer/includes/prod_short.md)] project before deploying it to production. In this section, you find resources on how to analyze and troubleshoot performance issues and guidance on how to validate performance of a system. 

### Performance Unit Testing

You can use the `SessionInformation` data type in unit tests that track the number of SQL statements or rows read. Use it  before and after the code to be tested. Then, have assert statements that check for normal behavior.

For more information, see [SessionInformation Data Type](../developer/methods-auto/sessioninformation/sessioninformation-data-type.md).

### Performance Scenario and Regression Testing

Use the Performance Toolkit to simulate the amount of resources that customers use in realistic scenarios to compare performance between builds of their solutions.

The Performance Toolkit helps answer questions such as, "Does my solution for Business Central support X number of users doing this, that, and the other thing at the same time?" 

For more information, see [The Performance Toolkit Extension](../developer/devenv-performance-toolkit.md).

> [!NOTE]  
> To test insert/update performance, make sure to un-install the test framework first. If the test framework is installed, then no insert/update statements can utilize bulk mode and will instead run row-by-row. 

### Performance Throughput Analysis

The Performance Toolkit doesn't answer questions such as, "How many orders can Business Central process per hour?" For this kind of analysis, test the time to execute key scenarios using the Performance Toolkit, and then use the guidance on [Operational Limits for Business Central Online](../administration/operational-limits-online.md). For advanced analysis, consider using a queueing model such as a [M/M/1 queue](https://en.wikipedia.org/wiki/M/M/1_queue) to answer whether the system can process the workload you intend.

### Performance telemetry

The following performance telemetry is available in Azure Application Insights (if that has been configured for the environment): 
- Database locks
- Long Running AL operations
- Long Running SQL Queries
- Page views
- Reports
- Sessions started
- Web Service Requests

For more information, see the [Analyzing performance issues using telemetry](performance-work-perf-problem.md#analyzing-performance-issues-using-telemetry) section.

### Troubleshooting

The following articles can be of help in troubleshooting performance issues:

- [Find missing SIFT indexes for FlowFields by Disabling SmartSQL](../administration/troubleshooting-queries-involving-flowfields-by-disabling-smartsql.md)  
- [Use Page Inspection to find extensions participating on a page](../developer/devenv-inspecting-pages.md)
- [Viewing Table Sizes](/dynamics365/business-central/admin-view-table-information)

## Tuning the Development Environment 

The following articles explain what you can do as a developer to tune your development environment for better performance:

- [Optimizing Visual Studio Code for AL Development](../developer/devenv-optimize-visual-studio-code.md)  
- [Code Analysis on large projects](../developer/devenv-using-code-analysis-tool.md#largeprojects)

## See Also

[Performance Overview](performance-overview.md)  
[How Application Configurations Affect Performance](performance-application.md)  
[Performance Online](performance-online.md)  
[Performance of On-Premises Installations](performance-onprem.md)  
[How to Work with a Performance Problem](performance-work-perf-problem.md)  
[Performance tips for business users](performance-users.md)  
[Database Missing Indexes](../administration/database-missing-indexes.md)  
[AL Database Methods and Performance on SQL Server](../administration/optimize-sql-al-database-methods-and-performance-on-server.md)

