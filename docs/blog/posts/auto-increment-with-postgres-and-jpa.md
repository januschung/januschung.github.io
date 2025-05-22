---
date: 2023-04-28
categories:
  - sql
  - postgres
  - java
---
# Auto Increment with Postgres and JPA
![banner](../../assets/blog/auto-increment-with-postgres-and-jpa/banner.avif)
## Background

I want to have Postgres to auto generate ID for my Springboot application entity, using Liquibase.

## What did't work

At first I had the following pair for the Java Entity Class and Liquibase setup: 

``` java
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
@Column(name = "ID")
private Long id;
```

<!-- more -->

``` xml
<createTable tableName="TICKET">
    <column autoIncrement="true" name="ID" type="BIGINT" remarks="Primary key">
        <constraints primaryKey="true" />
    </column>
```

which produced the following error:

``` bash
org.postgresql.util.PSQLException: ERROR: relation "support_system.ticket_seq" does not exist
```

Since it was complainting about the missing sequence, I modified the Liquibase setting to

``` xml
<createTable tableName="TICKET">
    <column defaultValueSequenceNext="seq_name" name="ID" type="BIGINT" remarks="Primary key">
        <constraints primaryKey="true" />
    </column>
```

which brought me to a different [bug](https://stackoverflow.com/questions/34010183/unable-to-insert-values-in-a-table-using-sequence-in-liquibase) since I am using a non public schema.

## The Solution

After some desperating debugging hours, I consulted my good friend Mark, the SQL guru, who convinced me that I do not need to setup a sequence. Instead, let Postgres generates one for me.

!!! tip inline "Mark's tips"
    You don’t generally need to reference the sequence object explicitly

By doing so, I can just create a column using the standard SQL identity, for example, `id int generated always as identity`.
That will create the sequence automatically and automatically call nextval to get a new id value when a new row is inserted.

After listened to Mark, I was convinced that I actually did not need to setup a sequence. Therefore, I reversed my chance in Liquibase and continued to try different thing on the JPA side. Here are the setting that finally works:

``` java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
@Column(name = "ID")
private Long id;
```

``` xml
<createTable tableName="TICKET">
    <column autoIncrement="true" name="ID" type="BIGINT" remarks="Primary key">
        <constraints primaryKey="true" />
    </column>
```

## Notes
The PRs which fixed the problem:

- [Liquibase](https://github.com/januschung/support-system-db/pull/8)
- [JPA](https://github.com/januschung/support-system-server/pull/11)