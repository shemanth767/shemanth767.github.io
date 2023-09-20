---
title: Demystifying Columnstore Indexes
date: 2023-09-19 10:00:00 +0530
categories: [Database]
tags: []
toc: true
published: true
mermaid: false
---

> This post assumes a working knowledge of SQL Server and an understanding of database indexes. If you're new to these topics, you may want to brush up on them before continuing.
{: .prompt-warning }

Let's dive in!

## Introduction

Database indexes can be both a blessing and a curse for those seeking to optimize query performance. Optimizing indexes
on SQL Server, in particular, can be a tricky task due to its ability to handle a wide range of workloads. These
workloads can typically be divided into two broad categories:
1. **OLTP Transactions** - These are real-time transaction processing scenarios that frequently update, insert, and
delete individual records. Examples of such scenarios include bank transactions or flight bookings.
2. **OLAP Transactions** - These are primarily used for analysis over transactions. In these scenarios, thousands or millions
of records are scanned and aggregated together. Examples of such scenarios include data warehouses.

Standard indexes created in SQL Server are [generally intended](# "take this with a spoon of salt") for OLTP transactions.
With SQL Server 2012, a new type of indexes were introduced that were meant especially for OLAP transactions. These are
columnstore indexes!

We'll be using an employment trail table tracking millions of employees to follow throughout this post,
```sql
create table dbo.fact_employment_trail
(
    id                    INT IDENTITY (1,1) PRIMARY KEY, -- Default clustered index column
    employee_id           INT          NOT NULL,
    employee_status       NVARCHAR(30) NOT NULL, -- Contains "Active" or "Terminated"
    employment_start_date DATE         NOT NULL,
    employment_end_date   DATE         NOT NULL
)

select * -- Sample OLTP Query. Employment details of a single employee.
from dbo.fact_employment_trail
where employee_id = '10035'

select count(*) -- Sample OLAP Query. Number of Active employees today.
from dbo.fact_employment_trail
where employee_status = 'Active'
  and CURRENT_TIMESTAMP between employment_start_date and employment_end_date

```


Some sample employment trails look like this,
![](/assets/img/columnstore-indexes-post/random-data-sample.png)

**Note:** We're using "2100-01-01" as the end date for an open ended record.

Before we dive into the specifics of columnstore indexes, let's take a moment to review how tables and rows are
typically stored on disk. In traditional rowstore storage, data is organized row by row, with all data belonging
to a single row stored together. This storage format is used by heaps and B-tree based indexes. It's well-suited
for OLTP transactions, which typically seek a particular key and retrieve the entire row. However, it may not be
the most efficient storage method for OLAP transactions.

Columnstore storage is a storage format that differs from rowstore by emphasizing columns over rows. In this format,
tables are logically organized as a table with rows and columns, but physically stored in a column-wise data format.
This unique storage format creates opportunities for a whole new set of optimizations, making columnstore indexes a
powerful tool for improving query performance. When a table has a clustered columnstore index, columnstore storage
is the data format in which the table is physically stored.

```sql
CREATE CLUSTERED COLUMNSTORE INDEX cci ON dbo.fact_employment_trail;
```

## Architecture

To truly understand how Columnstore Indexes benefit OLAP queries, it's important to understand how it's implemented
in SQL Server.

![](/assets/img/columnstore-indexes-post/columnstore-physicalstorage.gif)

### Rowgroups

Every table is split into groups of approximately a million rows before being stored on disk. These groups are called rowgroups. Rowgroups
are a key feature of columnstore indexes because they allow for efficient compression and processing of data. Each rowgroup is compressed
independently, which allows for better compression ratios and faster query performance. Additionally, rowgroups can be processed in parallel,
which can further improve query performance on large datasets.

In our example, the table `fact_employment_trail` consists of ~ 4 million rows, which get split into 4 separate row groups.


### Column Segments

In columnstore storage, each column in a table is compressed and stored together as a column segment. The index also stores metadata
per column segment for efficient filtering and querying, sometimes without decompressing the data. This compression allows the query
engine to reduce memory footprint and to operate on multiple rows together, making it particularly helpful. There is one column segment
for every column in the table.

### Deltastore

Columnstore indexes optimize updates and deletes by using temporary clustered B-Tree indexes called delta rowgroups, collectively known as
the Deltastore. These intermediate rowstore rowgroups speed up inserts and deletes by avoiding de-compressing the column segments. SQL Server
merges results from both the compressed column segments and the deltastore to handle queries.

A background process takes care of merging rows from the deltastore into the columnstore index when the number of rows reaches a threshold.


## Benefits and Disadvantages of Columnstore indexes
In which particular scenarios are they expected to perform really well.

## Choosing the right design

## Conclusion