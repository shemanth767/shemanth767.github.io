---
title: Advanced SQL Server Security Concepts
date: 2023-09-24 10:00:00 +0530
categories: [Database]
tags: []
toc: true
published: true
mermaid: false
---

In this post, we'll explore some advanced security concepts that I've found invaluable in my work with Data
Warehousing products. We'll break down these concepts in a way that's easy to understand, without the need
for extensive documentation. Let's dive in!

## Row-Level Security (RLS)

### Overview

SQL Server allows you to restrict access to individual records based on a pre-defined security policy.
This policy is controlled using a user-defined SQL function that determines whether a specific row should
be accessible to a user. For example, you can use this feature to restrict employees to only accessing data
within their department from a table containing firmwide data.

To understand how RLS works, let's use an example,

1. Create a table containing basic employee details,
    ```sql
    create table dbo.employee_details (
        id              INT IDENTITY (1,1) PRIMARY KEY,
        employee_id     INT           NOT NULL,
        username        NVARCHAR(30)  NOT NULL, 
        department      NVARCHAR(30)  NOT NULL,
        first_name      NVARCHAR(100) NOT NULL,
        last_name       NVARCHAR(100) NOT NULL
    )
    ```

2. Define a function that specifies how the rows should be restricted. In this case, we've created a function that only
    allows access when the department matches.
    ```sql
    -- It's recommended to put security objects in a separate schema.
    create schema security;
    go

    create function security.rls_func_employee_details_rls(@department AS nvarchar(30))
        returns table
    with SCHEMABINDING
    as
        return select 1
            where @department = (select department from dbo.employee_details where username = user_name());
    ```

3. Create a security policy that assigns the earlier defined function with the table.
    ```sql
    create security policy EmployeeDetailsFilter
        add filter predicate Security.rls_func_employee_details_rls(department)
            on dbo.employee_details
        with (state = ON);
    ```

4. Any reads on `dbo.employee_details` will now only return the employees for the querying user's department.
    For example, for my login, I'll only get employees from 'Dev' department.
    ```sql
    select * from dbo.employee_details
    ```

    | id | employee_id | username | department | first_name | last_name |
    |---|---|---|---|---|---|
    | 1 | 1 | beeraka | Dev | Sai | Beeraka |
    | 2 | 2 | rajaa | Dev | Ajith | Raja |
    | 5 | 5 | vasantis | Dev | Sameera | Vasanthi | 

### Predicates

In the previous example, we added the security policy using a **filter** security predicate. This is one of two types of
security predicates.

#### FILTER Predicate

As its name suggests, a filter predicate quietly filters out rows that don't match the security policy. The querying
user won't see any errors or warnings, as this predicate effectively acts as a filter over the rows the user can see
and modify.

When a filter predicate is added, the following operations are affected,
1. **SELECT** operations will only return rows matching the security predicate. If there are no matching rows, empty
    set is returned. 
2. **UPDATE** operations will only affect rows matching the security predicate.
3. **DELETE** operations will only delete rows matching the security predicate.

> The querying user should have SELECT/UPDATE/DELETE access to the base table in addition to passing the security predicate.
{: .prompt-info }

#### BLOCK Predicate

Block predicates explicitly block updates on a table based on the security predicate. The user will see an
error when an update fails because of the predicate.

1. **`AFTER INSERT` Predicate** - Ensures that a new row inserted is not violating the predicate. The new row
    should be accessible by the user inserting it.
2. **`AFTER UPDATE` Predicate** - Ensures that the updated row is not violating the security predicate. The updated row
    should be accessible by the user updating it.
3. **`BEFORE UPDATE` Predicate** - Prevents users from updating rows that currently violate the predicate.
4. **`BEFORE DELETE` Predicate** - Prevents users from deleting rows that currently violate the predicate.

While the FILTER predicate restricts users from accessing rows that violate the security policy, the `AFTER INSERT`
and `AFTER UPDATE` predicates go a step further by preventing users from creating new rows that violate the policy.
The `BEFORE UPDATE` and `BEFORE DELETE` checks are also performed by the FILTER predicate, but these additional
predicates can be useful in cases where different conditions are required for restricting filters and updates.

```sql
create security policy EmployeeDetailsBlockFilter
    add BLOCK predicate Security.rls_func_employee_details_rls(department)
        on dbo.employee_details AFTER INSERT
    with (state = ON);
```

### Additional Notes

1. The security predicates will apply to db_owners and sysadmins. However, they will have access to turn off
   the security policy.
2. When creating a security function with `SCHEMABINDING = OFF`, users will need to be granted explicit access to the
   function in order to query the base table. Using `SCHEMABINDING` simplifies permissions and eliminates the
   need for explicit access grants.
3. RLS was introduced in SQL Server 2016.

Refer the [official documentation](https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security?view=sql-server-ver16) for more details.


## Dynamic Data Masking (DDM)

### Overview

DDM is a security feature in SQL Server that allows you to mask sensitive data (such as phone numbers, email addresses
etc) to non-privileged users. It works by replacing the original data with a masked version, which can be customized
to show only a portion of the data or a completely different value.

With DDM, users with UNMASK access can view the original value, while others will only see the masked value. This
feature can be customized at the database, schema, table, or column level, providing granular control over data access.
However, at the column level, it is not possible to customize UNMASK access based on the row data.

### Example

1. Create a table containing employee details,
    ```sql
    create table dbo.employee_details (
        id              INT IDENTITY (1,1) PRIMARY KEY,
        employee_id     INT           NOT NULL,
        username        NVARCHAR(30)  NOT NULL, 
        department      NVARCHAR(30)  NOT NULL,
        first_name      NVARCHAR(100) NOT NULL,
        last_name       NVARCHAR(100) NOT NULL
    )
    ```

2. Add a mask to the `last_name` column. The below mask will only show the first character to non-privileged users.
    ```sql
    alter table dbo.employee_details
    alter column last_name add masked with (function = 'partial(1,"xxxx",0)')
    ```

3. Grant UNMASK permission to user on the column.
    ```sql
    -- Column Level, or
    GRANT UNMASK ON dbo.employee_details(last_name) TO beeraka;
    -- Schema Level
    GRANT UNMASK ON SCHEMA::dbo TO beeraka;
    ```

### Details


#### Masking Functions

SQL Server provides a few in-built masking functions which can be used. Users cannot define custom masking functions.

1. **default()** - Default value substitution for common types. E.g., Strings are replaced with XXXX.
2. **email()** - Exposes only the first letter and the suffix.
3. **random(start, end)** - Masking function to substitute random number for numeric type columns.
4. **partial(prefix, padding, suffix)** - Exposes the given prefix and suffix letters. Substitutes padding for the rest.
5. **datetime(unit)** - Masks only the given unit in the date column. E.g., `datetime("Y")` masks only the year.

#### Security Notes

1. DDM doesn't restrict updates to the masked column, if the user has UPDATE permission on it. 
2. DB Owners and Sysadmins have CONTROL access which grants them access to unmasked value in all columns.

Refer the [official documentation](https://learn.microsoft.com/en-us/sql/relational-databases/security/dynamic-data-masking?view=sql-server-ver16) for more details.
