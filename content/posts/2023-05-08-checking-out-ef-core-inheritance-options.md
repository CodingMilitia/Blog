---
author: João Antunes
date: 2023-05-08 08:25:00+01:00
layout: post
title: 'Checking out EF Core inheritance options'
summary: "Recently there was a need to use a hierarchy in a domain model, and there was a question about which of the available approaches to use - table per hierarchy (TPH), table per type (TPT) or table per concrete type (TPC). This very brief post is just a quick look at some benchmarks I created on the subject."
images:
- '/images/2023/05/08/checking-out-ef-core-inheritance-options.png'
categories:
- csharp
- dotnet
tags:
- efcore
- orm
- sql
---

Recently there was a need to use a hierarchy in a domain model, and there was a question about which of the available approaches to use - table per hierarchy (TPH), table per type (TPT) or table per concrete type (TPC). This very brief post is just a quick look at some benchmarks I created on the subject, available on [GitHub](https://github.com/joaofbantunes/CheckingOutEFCoreInheritanceOptions).

There is some guidance in the [docs](https://learn.microsoft.com/en-us/ef/core/modeling/inheritance), including some [benchmarks](https://learn.microsoft.com/en-us/ef/core/performance/modeling-for-performance#inheritance-mapping) (see more [here](https://github.com/dotnet/EntityFramework.Docs/issues/3890) and [here](https://github.com/dotnet/EntityFramework.Docs/blob/45dbda231795db12f91178c815eec85f06e6247b/samples/core/Benchmarks/Inheritance.cs)). However, as normal Microsoft focus more on SQL Server, with Shay adding some info about PostgreSQL, but currently at work we use MySQL, so I wanted to make sure if the same assumptions applied.

While I ended up testing multiple databases, my goal wasn't to compare them, but compare the inheritance options EF Core provides.

## Setup and what's tested

To test things, I created a hierarchy consisting of one base type, with 5 properties, plus 10 derived types, with 5 additional columns each. This means that, for each inheritance option, we have:

- TPH - a single table 56 columns (5 base, 5 per the other types, plus 1 discriminator)
- TPT - a table for the base, with 5 columns, plus 10 tables for each derived type with 6 columns (5 for the properties, 1 foreign key pointing to the base table)
- TPC - 10 tables, one per derived type, with 10 columns (5 for the base properties, 5 for the derived type properties)

With these tables in place, seeded with 1 million rows per hierarchy, so 100 thousand rows per derived type.

There are 2 different setups for SQL Server using TPH, a "regular" one, and one where the columns for the derived types properties are set as [sparse columns](https://learn.microsoft.com/en-us/sql/relational-databases/tables/use-sparse-columns?view=sql-server-ver16), in order to minimize the negative storage impact of having many `null` values.

As for what's tested, the benchmarks consist of:

- An offset paginated query - i.e. `Skip` and `Take`
- An index paginated query - i.e. instead of `Skip`, make use of an index to fetch pages more efficiently (see more in this [community standup](https://www.youtube.com/watch?v=DIKH-q-gJNU))
- A filter by index foreign key query - i.e. imagining we always query given a foreign key (e.g. if we're loading an aggregate root's related entities)
- Single by id - i.e. fetching an entity given its id
- Insert single - i.e. insert a single entity

## Quick takeaways

- TPT doesn't seem to ever be an interesting option
  - Performance is almost never better than other options
  - Storage is also not particularly good (in fact, other than in SQL Server, it's actually worse than the others)
- TPC always wins in terms of storage space
- TPH storage space isn't as bad as one would expect*
  - While it is worse than TPC, both PostgreSQL and MySQL show it's not too bad
  - *SQL Server is completely different, with much worse TPH sizes, unless we use sparse columns, in which case it's much more acceptable
- TPH performance almost always takes the win
  - But in general, TPC isn't too far off, with a couple of exceptions in the pagination queries when using MySQL

**The usual disclaimer:** it's entirely possible I messed something up, so be sure to let me know if you detect something 🙂

Also, keep in mind I did the tests guided by the needs of the projects I'm looking into. You may have different needs, in which case you should conduct your own experiments.

Now I'll just leave the numbers referring to storage sizes and benchmarks.

## Table Sizes

### PotsgreSQL Table Sizes

|Implementation Type|Table Size (MB)|Index Size (MB)|
|------------------:|--------------:|--------------:|
|Tph|183|32|
|Tpt|197|53|
|Tpc|170|28|

### MySQL Table Sizes

|Implementation Type|Table Size (MB)|Index Size (MB)|
|------------------:|--------------:|--------------:|
|Tph|230|19|
|Tpt|253|20|
|Tpc|185|25|

Side note: on different runs, got different size results, not really sure why, but these were the more common, so good enough for this quick analysis.

### SQL Server Table Sizes

|Implementation Type|Table Size (MB)|Index Size (MB)|
|------------------:|--------------:|--------------:|
|Tph|862|57|
|Tph Sparse|508|56|
|Tpt|400|55|
|Tpc|328|47|

Side note: no idea why SQL Server sizes are so different from PostgreSQL and MySQL... maybe the way the space is counted is different. Anyway, the goal isn't to compare databases, but focus on EF's inheritance options in the same database, so don't want to waste time figuring that one out (though I'm curious, if you know the reason, let me know).

## Benchmark results

ℹ️ you'll probably be able to look at this tables better on [GitHub](https://github.com/joaofbantunes/CheckingOutEFCoreInheritanceOptions), as my blog engine doesn't make it so easy to read them here

``` ini

BenchmarkDotNet=v0.13.5, OS=Windows 11 (10.0.22621.1555/22H2/2022Update/SunValley2)
Intel Core i7-9700K CPU 3.60GHz (Coffee Lake), 1 CPU, 8 logical and 8 physical cores
.NET SDK=8.0.100-preview.3.23178.7
  [Host]     : .NET 8.0.0 (8.0.23.17408), X64 RyuJIT AVX2
  DefaultJob : .NET 8.0.0 (8.0.23.17408), X64 RyuJIT AVX2


```
|                Method |         Categories |        Database |            Mean |          Error |           StdDev |          Median |             Min |             Max |  Ratio | RatioSD | Rank |
|---------------------- |------------------- |---------------- |----------------:|---------------:|-----------------:|----------------:|----------------:|----------------:|-------:|--------:|-----:|
| **TphFilterByForeignKey** | **FilterByForeignKey** |           **mysql** |     **7,298.05 μs** |     **145.687 μs** |       **284.152 μs** |     **7,171.32 μs** |     **6,893.39 μs** |     **8,045.21 μs** |   **1.00** |    **0.00** |    **1** |
| TptFilterByForeignKey | FilterByForeignKey |           mysql |    21,892.76 μs |     366.981 μs |       450.686 μs |    21,839.76 μs |    20,986.10 μs |    22,924.88 μs |   2.98 |    0.12 |    3 |
| TpcFilterByForeignKey | FilterByForeignKey |           mysql |     9,105.79 μs |     177.535 μs |       189.960 μs |     9,046.15 μs |     8,876.40 μs |     9,498.80 μs |   1.24 |    0.04 |    2 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|    TphIndexPagination |    IndexPagination |           mysql |     5,472.05 μs |     118.578 μs |       340.223 μs |     5,351.43 μs |     5,098.57 μs |     6,437.81 μs |   1.00 |    0.00 |    1 |
|    TptIndexPagination |    IndexPagination |           mysql |    19,901.95 μs |     252.575 μs |       310.185 μs |    19,848.02 μs |    19,523.03 μs |    20,742.01 μs |   3.65 |    0.23 |    2 |
|    TpcIndexPagination |    IndexPagination |           mysql | 2,497,193.86 μs | 555,852.552 μs | 1,638,943.790 μs | 2,296,448.60 μs |     9,934.40 μs | 5,642,275.50 μs | 462.57 |  304.35 |    3 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|       TphInsertSingle |       InsertSingle |           mysql |        92.48 μs |       1.466 μs |         1.372 μs |        92.24 μs |        90.33 μs |        94.66 μs |   1.00 |    0.00 |    1 |
|       TptInsertSingle |       InsertSingle |           mysql |        92.60 μs |       1.420 μs |         1.328 μs |        92.16 μs |        90.34 μs |        94.32 μs |   1.00 |    0.02 |    1 |
|       TpcInsertSingle |       InsertSingle |           mysql |        91.89 μs |       1.482 μs |         1.387 μs |        92.03 μs |        89.09 μs |        93.96 μs |   0.99 |    0.02 |    1 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|   TphOffsetPagination |   OffsetPagination |           mysql |   686,556.03 μs |  18,207.971 μs |    53,113.505 μs |   689,618.05 μs |   578,878.10 μs |   814,183.30 μs |   1.00 |    0.00 |    1 |
|   TptOffsetPagination |   OffsetPagination |           mysql | 6,674,680.02 μs | 152,806.909 μs |   445,744.927 μs | 6,701,560.15 μs | 5,803,526.00 μs | 7,691,568.80 μs |   9.78 |    0.96 |    2 |
|   TpcOffsetPagination |   OffsetPagination |           mysql | 7,953,703.73 μs | 157,763.297 μs |   292,424.387 μs | 7,899,569.10 μs | 7,389,297.40 μs | 8,636,608.20 μs |  11.70 |    1.08 |    3 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|         TphSingleById |         SingleById |           mysql |     1,925.27 μs |      34.267 μs |        60.016 μs |     1,936.36 μs |     1,790.35 μs |     2,079.88 μs |   1.00 |    0.00 |    1 |
|         TptSingleById |         SingleById |           mysql |     2,185.29 μs |      42.111 μs |        48.496 μs |     2,186.05 μs |     2,090.20 μs |     2,265.59 μs |   1.14 |    0.04 |    2 |
|         TpcSingleById |         SingleById |           mysql |     2,822.42 μs |      51.875 μs |        43.318 μs |     2,825.38 μs |     2,738.04 μs |     2,894.05 μs |   1.47 |    0.04 |    3 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
| **TphFilterByForeignKey** | **FilterByForeignKey** |        **postgres** |     **6,253.29 μs** |     **124.031 μs** |       **297.170 μs** |     **6,123.99 μs** |     **5,909.87 μs** |     **7,286.57 μs** |   **1.00** |    **0.00** |    **1** |
| TptFilterByForeignKey | FilterByForeignKey |        postgres |    91,758.47 μs |   1,745.343 μs |     1,714.160 μs |    90,974.13 μs |    89,762.56 μs |    95,192.80 μs |  14.64 |    0.58 |    3 |
| TpcFilterByForeignKey | FilterByForeignKey |        postgres |     9,618.61 μs |     192.253 μs |       467.970 μs |     9,434.32 μs |     9,146.79 μs |    10,870.00 μs |   1.54 |    0.11 |    2 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|    TphIndexPagination |    IndexPagination |        postgres |     6,379.66 μs |     151.638 μs |       425.209 μs |     6,268.53 μs |     5,916.71 μs |     7,692.42 μs |   1.00 |    0.00 |    1 |
|    TptIndexPagination |    IndexPagination |        postgres |   160,096.54 μs |  15,609.846 μs |    46,025.983 μs |   165,390.70 μs |    50,814.28 μs |   276,115.12 μs |  24.90 |    7.22 |    3 |
|    TpcIndexPagination |    IndexPagination |        postgres |    13,696.00 μs |     272.819 μs |       704.234 μs |    13,469.21 μs |    12,827.53 μs |    15,690.74 μs |   2.15 |    0.18 |    2 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|       TphInsertSingle |       InsertSingle |        postgres |        65.56 μs |       0.689 μs |         0.576 μs |        65.59 μs |        64.18 μs |        66.43 μs |   1.00 |    0.00 |    1 |
|       TptInsertSingle |       InsertSingle |        postgres |        65.82 μs |       0.787 μs |         0.736 μs |        65.62 μs |        64.75 μs |        67.36 μs |   1.01 |    0.01 |    1 |
|       TpcInsertSingle |       InsertSingle |        postgres |        66.12 μs |       0.481 μs |         0.375 μs |        66.05 μs |        65.60 μs |        66.97 μs |   1.01 |    0.01 |    1 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|   TphOffsetPagination |   OffsetPagination |        postgres |   213,521.12 μs |   4,246.765 μs |     9,585.656 μs |   212,935.60 μs |   191,877.17 μs |   234,909.63 μs |   1.00 |    0.00 |    1 |
|   TptOffsetPagination |   OffsetPagination |        postgres | 1,357,025.55 μs |  35,411.320 μs |   100,456.020 μs | 1,352,052.60 μs | 1,169,516.90 μs | 1,636,067.80 μs |   6.37 |    0.51 |    3 |
|   TpcOffsetPagination |   OffsetPagination |        postgres |   438,948.41 μs |  13,697.443 μs |    37,955.550 μs |   432,251.20 μs |   383,813.40 μs |   542,211.50 μs |   2.05 |    0.20 |    2 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|         TphSingleById |         SingleById |        postgres |     1,340.12 μs |      26.616 μs |        44.470 μs |     1,337.76 μs |     1,237.36 μs |     1,434.88 μs |   1.00 |    0.00 |    1 |
|         TptSingleById |         SingleById |        postgres |     3,384.95 μs |      74.232 μs |       204.457 μs |     3,343.06 μs |     3,142.61 μs |     3,943.66 μs |   2.53 |    0.16 |    2 |
|         TpcSingleById |         SingleById |        postgres |     4,372.85 μs |      85.943 μs |        84.408 μs |     4,366.50 μs |     4,261.21 μs |     4,527.16 μs |   3.26 |    0.11 |    3 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
| **TphFilterByForeignKey** | **FilterByForeignKey** |       **sqlserver** |     **7,502.31 μs** |     **189.874 μs** |       **544.783 μs** |     **7,376.92 μs** |     **6,794.47 μs** |     **8,993.13 μs** |   **1.00** |    **0.00** |    **1** |
| TptFilterByForeignKey | FilterByForeignKey |       sqlserver |   149,245.55 μs |  14,873.088 μs |    43,620.191 μs |   146,197.12 μs |    66,747.35 μs |   271,158.75 μs |  19.96 |    6.02 |    3 |
| TpcFilterByForeignKey | FilterByForeignKey |       sqlserver |     9,624.02 μs |     284.445 μs |       811.539 μs |     9,298.03 μs |     8,689.16 μs |    11,887.78 μs |   1.29 |    0.14 |    2 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|    TphIndexPagination |    IndexPagination |       sqlserver |     6,537.04 μs |     177.472 μs |       503.457 μs |     6,344.45 μs |     5,910.22 μs |     8,202.34 μs |   1.00 |    0.00 |    1 |
|    TptIndexPagination |    IndexPagination |       sqlserver |    10,330.81 μs |     230.966 μs |       640.007 μs |    10,185.47 μs |     9,431.12 μs |    12,216.56 μs |   1.59 |    0.16 |    3 |
|    TpcIndexPagination |    IndexPagination |       sqlserver |     9,825.42 μs |     293.905 μs |       824.142 μs |     9,562.68 μs |     8,812.01 μs |    12,164.05 μs |   1.51 |    0.16 |    2 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|       TphInsertSingle |       InsertSingle |       sqlserver |       123.56 μs |       1.968 μs |         1.745 μs |       123.23 μs |       120.87 μs |       126.83 μs |   1.00 |    0.00 |    1 |
|       TptInsertSingle |       InsertSingle |       sqlserver |       128.59 μs |       2.313 μs |         2.163 μs |       128.82 μs |       124.88 μs |       132.89 μs |   1.04 |    0.02 |    2 |
|       TpcInsertSingle |       InsertSingle |       sqlserver |       124.48 μs |       2.189 μs |         2.048 μs |       124.64 μs |       121.31 μs |       129.41 μs |   1.01 |    0.02 |    1 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|   TphOffsetPagination |   OffsetPagination |       sqlserver |   455,908.61 μs |  12,036.343 μs |    34,534.524 μs |   450,956.20 μs |   398,003.10 μs |   533,179.00 μs |   1.00 |    0.00 |    1 |
|   TptOffsetPagination |   OffsetPagination |       sqlserver |   603,639.95 μs |  15,325.715 μs |    43,972.351 μs |   605,557.00 μs |   528,056.10 μs |   718,534.50 μs |   1.33 |    0.15 |    2 |
|   TpcOffsetPagination |   OffsetPagination |       sqlserver |   445,484.34 μs |  11,930.393 μs |    34,038.083 μs |   449,323.65 μs |   388,851.50 μs |   535,960.50 μs |   0.98 |    0.11 |    1 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|         TphSingleById |         SingleById |       sqlserver |     1,583.05 μs |      31.318 μs |        72.583 μs |     1,572.07 μs |     1,398.39 μs |     1,771.47 μs |   1.00 |    0.00 |    1 |
|         TptSingleById |         SingleById |       sqlserver |     1,723.23 μs |      34.114 μs |        83.682 μs |     1,701.20 μs |     1,602.50 μs |     1,955.27 μs |   1.09 |    0.08 |    2 |
|         TpcSingleById |         SingleById |       sqlserver |     1,972.69 μs |      39.432 μs |       111.218 μs |     1,957.83 μs |     1,789.39 μs |     2,273.89 μs |   1.26 |    0.09 |    3 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
| **TphFilterByForeignKey** | **FilterByForeignKey** | **sqlserversparse** |     **7,998.21 μs** |     **251.680 μs** |       **705.736 μs** |     **7,725.89 μs** |     **7,231.11 μs** |    **10,128.25 μs** |   **1.00** |    **0.00** |    **1** |
| TptFilterByForeignKey | FilterByForeignKey | sqlserversparse |   159,461.09 μs |  13,805.964 μs |    40,707.197 μs |   163,862.17 μs |    66,195.32 μs |   242,374.23 μs |  20.01 |    5.53 |    3 |
| TpcFilterByForeignKey | FilterByForeignKey | sqlserversparse |     9,706.81 μs |     303.706 μs |       876.261 μs |     9,343.45 μs |     8,761.20 μs |    12,405.88 μs |   1.22 |    0.16 |    2 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|    TphIndexPagination |    IndexPagination | sqlserversparse |     7,041.51 μs |     259.631 μs |       723.748 μs |     6,816.42 μs |     6,217.95 μs |     9,141.39 μs |   1.00 |    0.00 |    1 |
|    TptIndexPagination |    IndexPagination | sqlserversparse |    10,411.29 μs |     278.144 μs |       784.511 μs |    10,151.35 μs |     9,394.86 μs |    12,674.03 μs |   1.49 |    0.17 |    3 |
|    TpcIndexPagination |    IndexPagination | sqlserversparse |    10,113.96 μs |     416.151 μs |     1,187.301 μs |     9,666.66 μs |     8,745.51 μs |    13,665.55 μs |   1.45 |    0.20 |    2 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|       TphInsertSingle |       InsertSingle | sqlserversparse |       125.29 μs |       2.188 μs |         2.047 μs |       125.69 μs |       122.53 μs |       128.86 μs |   1.00 |    0.00 |    1 |
|       TptInsertSingle |       InsertSingle | sqlserversparse |       124.97 μs |       2.469 μs |         2.309 μs |       124.21 μs |       122.30 μs |       129.30 μs |   1.00 |    0.01 |    1 |
|       TpcInsertSingle |       InsertSingle | sqlserversparse |       128.27 μs |       1.660 μs |         1.552 μs |       128.00 μs |       126.06 μs |       131.39 μs |   1.02 |    0.02 |    2 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|   TphOffsetPagination |   OffsetPagination | sqlserversparse |   557,174.33 μs |  13,632.623 μs |    38,673.484 μs |   561,865.30 μs |   480,424.00 μs |   655,422.80 μs |   1.00 |    0.00 |    2 |
|   TptOffsetPagination |   OffsetPagination | sqlserversparse |   572,320.90 μs |  16,122.624 μs |    44,406.388 μs |   565,797.80 μs |   510,039.90 μs |   716,814.20 μs |   1.03 |    0.11 |    2 |
|   TpcOffsetPagination |   OffsetPagination | sqlserversparse |   445,093.99 μs |  11,507.413 μs |    32,644.616 μs |   447,922.90 μs |   389,376.40 μs |   527,289.50 μs |   0.80 |    0.07 |    1 |
|                       |                    |                 |                 |                |                  |                 |                 |                 |        |         |      |
|         TphSingleById |         SingleById | sqlserversparse |     1,582.53 μs |      31.492 μs |        82.409 μs |     1,566.34 μs |     1,420.67 μs |     1,805.21 μs |   1.00 |    0.00 |    1 |
|         TptSingleById |         SingleById | sqlserversparse |     1,722.36 μs |      39.357 μs |       111.650 μs |     1,696.56 μs |     1,509.26 μs |     2,030.45 μs |   1.10 |    0.10 |    2 |
|         TpcSingleById |         SingleById | sqlserversparse |     1,984.95 μs |      38.209 μs |       105.239 μs |     1,953.11 μs |     1,813.89 μs |     2,280.75 μs |   1.26 |    0.09 |    3 |

Relevant links:

- [Sample code](https://github.com/joaofbantunes/CheckingOutEFCoreInheritanceOptions)
- [Inheritance - EF Core docs](https://learn.microsoft.com/en-us/ef/core/modeling/inheritance)
- [Inheritance mapping - Modeling for Performance - EF Core docs](https://learn.microsoft.com/en-us/ef/core/performance/modeling-for-performance#inheritance-mapping)
- [Update performance docs to take TPC into account - GitHub Issue](https://github.com/dotnet/EntityFramework.Docs/issues/3890)
- [EF Core docs linked benchmark](https://github.com/dotnet/EntityFramework.Docs/blob/45dbda231795db12f91178c815eec85f06e6247b/samples/core/Benchmarks/Inheritance.cs)

Thanks for stopping by, cyaz! 👋