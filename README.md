# SQLAlchemy Core — Notes

## Table of Contents

- [1. What is SQLAlchemy Core?](#1-what-is-sqlalchemy-core)
- [2. Engine and connection basics](#2-engine-and-connection-basics)
- [3. Metadata and table definitions](#3-metadata-and-table-definitions)
- [4. Inserting data](#4-inserting-data)
- [5. Selecting data](#5-selecting-data)
- [6. Filtering, ordering, and limiting](#6-filtering-ordering-and-limiting)
- [7. Joins and relationships](#7-joins-and-relationships)
- [8. Aggregations and SQL functions](#8-aggregations-and-sql-functions)
- [9. Set operations](#9-set-operations)
- [10. Updating and deleting](#10-updating-and-deleting)
- [11. Transactions and execution patterns](#11-transactions-and-execution-patterns)
- [12. Core vs ORM](#12-core-vs-orm)
- [13. Quick Reference](#13-quick-reference)

## 1. What is SQLAlchemy Core?

**SQLAlchemy Core** is the SQL-expression layer of SQLAlchemy. It lets you build database statements in Python using Python objects instead of writing raw SQL strings.

It is useful when you want:

- explicit SQL generation
- schema-aware queries
- reusable query components
- control over inserts, updates, deletes, and joins

> 💡 Prefer SQLAlchemy Core when you want SQL-style control without the full ORM abstraction.

### Basic pattern

**Universal syntax**

```python
from sqlalchemy import create_engine, Table, Column, Integer, String, MetaData, select
```

**Applied syntax**

```python
engine = create_engine("sqlite:///shop.db")
metadata = MetaData()
```

## 2. Engine and connection basics

**The `create_engine()` function** creates a database connection engine. It is the entry point for SQLAlchemy Core.

**Universal syntax**

```python
engine = create_engine("dialect+driver://user:password@host/dbname")
```

**Applied syntax**

```python
engine = create_engine("sqlite:///shop.db")
connection = engine.connect()
```

### What the engine does

- manages database connections
- prepares SQL statements
- executes SQL against the target database
- supports transaction and connection pooling

> ⚠️ The engine itself does not run SQL directly in the ORM sense; it is usually used with a `Connection` or execution API.

## 3. Metadata and table definitions

**`MetaData`** stores table definitions and schema information. A **`Table`** represents a database table in Python.

**Universal syntax**

```python
metadata = MetaData()
users = Table(
    "users",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String),
)
```

**Applied syntax**

```python
from sqlalchemy import MetaData, Table, Column, Integer, String, Float

metadata = MetaData()

payment = Table(
    "payment",
    metadata,
    Column("payment_id", Integer, primary_key=True),
    Column("customer_id", Integer),
    Column("amount", Float),
    Column("payment_date", String),
)

customer = Table(
    "customer",
    metadata,
    Column("customer_id", Integer, primary_key=True),
    Column("first_name", String),
    Column("last_name", String),
)
```

### Key concepts

- `MetaData` collects tables
- `Table` maps to a real database table
- `Column` describes a field and its type
- `primary_key=True` marks the table primary key

### Table introspection

```python
print(payment)
print(payment.c.payment_id)
print(payment.columns)
```

## 4. Inserting data

**`insert()`** creates an INSERT statement. It can insert one row or many rows at once.

**Universal syntax**

```python
stmt = table.insert().values({...})
result = connection.execute(stmt)
```

**Applied syntax**

```python
from sqlalchemy import insert

stmt = payment.insert().values(
    customer_id=1,
    amount=250.00,
    payment_date="2024-01-10",
)

connection.execute(stmt)
```

### Insert many rows

**Universal syntax**

```python
stmt = table.insert()
connection.execute(stmt, [{...}, {...}])
```

**Applied syntax**

```python
stmt = payment.insert()
connection.execute(
    stmt,
    [
        {"customer_id": 1, "amount": 100.00, "payment_date": "2024-01-11"},
        {"customer_id": 2, "amount": 150.50, "payment_date": "2024-01-12"},
    ],
)
```

> 💡 Batch inserts are more efficient than many single-row inserts when processing large datasets.

## 5. Selecting data

**`select()`** builds a SELECT query. It is the core of SQLAlchemy Core read operations.

**Universal syntax**

```python
stmt = select(table1, table2).where(condition)
result = connection.execute(stmt)
```

**Applied syntax**

```python
from sqlalchemy import select

stmt = select(payment).where(payment.c.payment_id == 1)
result = connection.execute(stmt)
row = result.fetchone()
```

### Selecting specific columns

**Universal syntax**

```python
stmt = select(table.c.col1, table.c.col2)
```

**Applied syntax**

```python
stmt = select(payment.c.payment_id, payment.c.amount)
result = connection.execute(stmt)
```

### Selecting with multiple tables

```python
stmt = select(payment.c.payment_id, customer.c.first_name)
stmt = stmt.select_from(payment.join(customer, payment.c.customer_id == customer.c.customer_id))
```

## 6. Filtering, ordering, and limiting

**`where()`** filters rows. **`order_by()`** sorts results. **`limit()`** caps the number of results.

**Universal syntax**

```python
stmt = select(table).where(condition).order_by(column).limit(n)
```

**Applied syntax**

```python
stmt = (
    select(payment)
    .where(payment.c.amount > 100)
    .order_by(payment.c.amount.desc())
    .limit(5)
)
```

### Common conditions

```python
select(payment).where(payment.c.customer_id == 1)
select(payment).where(payment.c.amount >= 200)
select(payment).where(payment.c.payment_date.like("2024%"))
```

### Filtering with multiple conditions

```python
stmt = select(payment).where(
    (payment.c.customer_id == 1) & (payment.c.amount > 100)
)
```

### Logical operators

- `==` for equality
- `!=` for inequality
- `>` and `<` for comparisons
- `&` for `AND`
- `|` for `OR`
- `~` for `NOT`

> **Note:** In SQLAlchemy expressions, combine conditions with `&`, `|`, and `~`, not Python `and`, `or`, and `not`.

## 7. Joins and relationships

**Joins** combine rows from multiple tables using related columns.

**Universal syntax**

```python
stmt = select(table1, table2).select_from(table1.join(table2, table1.c.id == table2.c.id))
```

**Applied syntax**

```python
stmt = select(payment.c.payment_id, customer.c.first_name).select_from(
    payment.join(customer, payment.c.customer_id == customer.c.customer_id)
)
```

### Inner join

```python
stmt = select(payment.c.payment_id, customer.c.first_name).select_from(
    payment.join(customer, payment.c.customer_id == customer.c.customer_id)
)
```

### Left join

**Universal syntax**

```python
stmt = select(...).select_from(table1.outerjoin(table2, condition))
```

**Applied syntax**

```python
stmt = select(customer.c.customer_id, payment.c.amount).select_from(
    customer.outerjoin(payment, customer.c.customer_id == payment.c.customer_id)
)
```

### Join aliasing

```python
from sqlalchemy import aliased

c1 = aliased(customer)
c2 = aliased(customer)
```

## 8. Aggregations and SQL functions

**SQLAlchemy Core includes SQL functions for aggregates and calculations.**

**Universal syntax**

```python
from sqlalchemy import func

stmt = select(func.count(table.c.id))
```

**Applied syntax**

```python
from sqlalchemy import func

stmt = select(func.count(payment.c.payment_id).label("total_payments"))
result = connection.execute(stmt)
```

### Common aggregate functions

```python
select(func.sum(payment.c.amount))
select(func.avg(payment.c.amount))
select(func.max(payment.c.amount))
select(func.min(payment.c.amount))
```

### Grouping results

**Universal syntax**

```python
stmt = select(column, func.count(...)).group_by(column)
```

**Applied syntax**

```python
stmt = select(
    payment.c.customer_id,
    func.sum(payment.c.amount).label("total_amount"),
).group_by(payment.c.customer_id)
```

## 9. Set operations

**`union()`, `union_all()`, `intersect()`, and `except_()`** combine result sets from multiple SELECT statements.

### UNION vs UNION ALL

| Operation | Duplicates | Use case |
| --- | --- | --- |
| `union()` | Removed | When you need unique rows |
| `union_all()` | Kept | When duplicates are acceptable or impossible |
| `intersect()` | Common rows only | When rows appear in both queries |
| `except_()` | Rows in first not second | When subtracting result sets |

**Universal syntax**

```python
stmt = query1.union(query2)
stmt = query1.union_all(query2)
```

**Applied syntax**

```python
from sqlalchemy import union, union_all

q1 = select(payment.c.customer_id, payment.c.amount).where(payment.c.amount > 100)
q2 = select(payment.c.customer_id, payment.c.amount).where(payment.c.amount < 50)

result = connection.execute(union(q1, q2))
result = connection.execute(union_all(q1, q2))
```

> 💡 Prefer `union_all()` when duplicate rows are impossible or duplicates are not important, because it is usually faster.

## 10. Updating and deleting

### Update

**Universal syntax**

```python
stmt = table.update().where(condition).values({...})
connection.execute(stmt)
```

**Applied syntax**

```python
from sqlalchemy import update

stmt = (
    payment.update()
    .where(payment.c.payment_id == 1)
    .values(amount=275.00)
)

connection.execute(stmt)
```

### Delete

**Universal syntax**

```python
stmt = table.delete().where(condition)
connection.execute(stmt)
```

**Applied syntax**

```python
from sqlalchemy import delete

stmt = delete(payment).where(payment.c.payment_id == 1)
connection.execute(stmt)
```

## 11. Transactions and execution patterns

**Transactions** ensure atomic database operations. SQLAlchemy allows explicit transaction control.

**Universal syntax**

```python
with engine.begin() as connection:
    connection.execute(stmt1)
    connection.execute(stmt2)
```

**Applied syntax**

```python
with engine.begin() as connection:
    connection.execute(payment.insert().values(customer_id=3, amount=300.00, payment_date="2024-01-25"))
    connection.execute(
        payment.update()
        .where(payment.c.payment_id == 3)
        .values(amount=325.00)
    )
```

### Common execution patterns

```python
result = connection.execute(stmt)
rows = result.fetchall()
for row in rows:
    print(row)
```

> ⚠️ A transaction is committed only when the context exits successfully. If an exception occurs, the transaction is rolled back.

## 12. Core vs ORM

| Feature | SQLAlchemy Core | SQLAlchemy ORM |
| --- | --- | --- |
| Primary style | SQL expression language | mapped Python classes |
| Data model | `Table` and `Column` | `Declarative` models / mapped classes |
| Best for | explicit SQL and schema control | application-level object modeling |
| Query style | `select(table)` | `session.query(Model)` or `session.execute(select(Model))` |
| Relationship handling | manual join expressions | object-level relationships |

**Core** is lower-level and more SQL-centric. **ORM** adds object mapping and higher-level relationship behavior.

## 13. Quick Reference

| Concept | Universal syntax | Applied syntax |
| --- | --- | --- |
| Engine | `create_engine("dialect+driver://...")` | `create_engine("sqlite:///shop.db")` |
| Table | `Table("name", metadata, Column(...))` | `payment = Table("payment", metadata, Column("payment_id", Integer, primary_key=True))` |
| Insert | `table.insert().values({...})` | `payment.insert().values(customer_id=1, amount=250.00)` |
| Select | `select(table).where(condition)` | `select(payment).where(payment.c.payment_id == 1)` |
| Select columns | `select(table.c.col1, table.c.col2)` | `select(payment.c.payment_id, payment.c.amount)` |
| Filter | `.where(condition)` | `.where(payment.c.amount > 100)` |
| Order | `.order_by(column)` | `.order_by(payment.c.amount.desc())` |
| Limit | `.limit(n)` | `.limit(5)` |
| Join | `.join(table, condition)` | `payment.join(customer, payment.c.customer_id == customer.c.customer_id)` |
| Aggregate | `func.count(table.c.id)` | `func.sum(payment.c.amount)` |
| Update | `table.update().where(condition).values({...})` | `payment.update().where(payment.c.payment_id == 1).values(amount=275.00)` |
| Delete | `table.delete().where(condition)` | `delete(payment).where(payment.c.payment_id == 1)` |
| Union | `query1.union(query2)` | `union(q1, q2)` |
| Union All | `query1.union_all(query2)` | `union_all(q1, q2)` |

This README is intended as a clean reference for SQLAlchemy Core fundamentals with practical examples using a consistent `payment`, `customer`, and `order`-style schema.
