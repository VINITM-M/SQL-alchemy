# SQLAlchemy Core — Notes

## Table of Contents

- [1. What is SQLAlchemy Core?](#1-what-is-sqlalchemy-core)
- [2. Engine and connection basics](#2-engine-and-connection-basics)
- [3. Metadata and table definitions](#3-metadata-and-table-definitions)
- [4. Declarative Base and ORM Classes](#4-declarative-base-and-orm-classes)
- [5. Sessions and Session Management](#5-sessions-and-session-management)
- [6. Inserting data](#6-inserting-data)
- [7. Selecting data](#7-selecting-data)
- [8. Filtering, ordering, and limiting](#8-filtering-ordering-and-limiting)
- [9. Joins and relationships](#9-joins-and-relationships)
- [10. Aggregations and SQL functions](#10-aggregations-and-sql-functions)
- [11. Set operations](#11-set-operations)
- [12. Updating and deleting](#12-updating-and-deleting)
- [13. Transactions and execution patterns](#13-transactions-and-execution-patterns)
- [14. Core vs ORM](#14-core-vs-orm)
- [15. Quick Reference](#15-quick-reference)

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

### Engine analogy

Think of the engine like a restaurant:

```
engine = "the restaurant itself (address, kitchen, everything)"
         ↓
One connection to your database. You make it once and keep it forever.
```

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

## 4. Declarative Base and ORM Classes

**`declarative_base()`** creates a base class that all your table classes inherit from. It's the magic glue that tells SQLAlchemy to turn your Python classes into database tables.

### Why You Need declarative_base()

Without `declarative_base()`, your Python class is just a regular class. With it, the class becomes a database table automatically.

| Without Base | With Base |
| --- | --- |
| `Student` is just a Python class | `Student` becomes a database table |
| `id = Column(...)` is just a variable | `id` becomes a real database column with a type |
| `__tablename__` means nothing | `__tablename__` becomes the actual table name in the database |

### Universal syntax

```python
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class TableName(Base):
    __tablename__ = 'table_name'
    id = Column(Integer, primary_key=True)
    column_name = Column(String)
```

### Applied syntax

```python
from sqlalchemy import create_engine, Column, Integer, String, Float
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Student(Base):
    __tablename__ = 'students'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)

class Fee(Base):
    __tablename__ = 'fees'
    id = Column(Integer, primary_key=True)
    student_id = Column(Integer)
    amount = Column(Float)
    payment_date = Column(String)

class Payment(Base):
    __tablename__ = 'payment'
    payment_id = Column(Integer, primary_key=True)
    customer_id = Column(Integer)
    amount = Column(Float)
    payment_date = Column(String)

class Customer(Base):
    __tablename__ = 'customer'
    customer_id = Column(Integer, primary_key=True)
    first_name = Column(String)
    last_name = Column(String)
```

### Creating tables in the database

```python
# Create all tables defined in Base
Base.metadata.create_all(engine)

# This will create the actual tables in your database
```

> **One-Line Summary**: `declarative_base()` = "From now on, any class that inherits from me will automatically become a database table."

## 5. Sessions and Session Management

**A `Session`** is your working object that manages communication between your Python code and the database. Think of it as a waiter taking your order to the kitchen.

### Understanding Engine vs Session vs SessionMaker

Think of it like going to a restaurant:

| Code | Restaurant analogy | What it actually does |
| --- | --- | --- |
| `engine = create_engine(...)` | The restaurant itself (address, kitchen, everything) | One connection to your database. You make it once and keep it forever. |
| `Session = sessionmaker(bind=engine)` | The waiter's training (how to take orders, how to serve) | A template for creating sessions. You make it once. |
| `session = Session()` | A specific waiter sitting at YOUR table right now | Your actual working object. You make a new one every time someone makes a request. |

### Simple analogy

```
engine   = "the database"         (one, never changes)
Session  = "how to talk to it"    (one, never changes)
session  = "I am talking to it NOW" (new one every request)
```

### Universal syntax

```python
from sqlalchemy.orm import sessionmaker

Session = sessionmaker(bind=engine)
session = Session()
```

### Applied syntax

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base

engine = create_engine("sqlite:///shop.db")
Base = declarative_base()

# Step 1: Create the SessionMaker (do this ONCE)
Session = sessionmaker(bind=engine)

# Step 2: Create a session instance (do this for each request/operation)
session = Session()

# Now you can use the session to interact with the database
```

### Using a session with context manager (recommended)

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine("sqlite:///shop.db")
Session = sessionmaker(bind=engine)

# Recommended: Use context manager for automatic cleanup
with Session() as session:
    # Your database operations here
    pass
# Session automatically closes when exiting the block
```

### Session lifecycle

```python
# Create engine once
engine = create_engine("sqlite:///shop.db")

# Create SessionMaker once (at app startup)
Session = sessionmaker(bind=engine)

# In your application code:
# Create a new session for each request
session = Session()

try:
    # Do your database work
    # session.add(obj)
    # session.commit()
    pass
finally:
    # Always close the session
    session.close()
```

> ⚠️ Always close your sessions when done. Use context managers (`with`) for automatic cleanup.

## 6. Inserting data

**`insert()`** creates an INSERT statement. It can insert one row or many rows at once.

**Universal syntax**

```python
stmt = table.insert().values({...})
result = connection.execute(stmt)
```

**Applied syntax with Core**

```python
from sqlalchemy import insert

stmt = payment.insert().values(
    customer_id=1,
    amount=250.00,
    payment_date="2024-01-10",
)

connection.execute(stmt)
```

**Applied syntax with ORM and Session**

```python
from sqlalchemy.orm import sessionmaker

Session = sessionmaker(bind=engine)
session = Session()

# Create an object
new_payment = Payment(
    customer_id=1,
    amount=250.00,
    payment_date="2024-01-10"
)

# Add it to the session
session.add(new_payment)

# Commit to save it to the database
session.commit()

# Close when done
session.close()
```

### Insert many rows

**Universal syntax (Core)**

```python
stmt = table.insert()
connection.execute(stmt, [{...}, {...}])
```

**Applied syntax (Core)**

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

**Applied syntax with ORM and Session**

```python
Session = sessionmaker(bind=engine)
session = Session()

# Create multiple objects
payments = [
    Payment(customer_id=1, amount=100.00, payment_date="2024-01-11"),
    Payment(customer_id=2, amount=150.50, payment_date="2024-01-12"),
]

# Add all at once
session.add_all(payments)

# Commit to save
session.commit()
session.close()
```

> 💡 Batch inserts are more efficient than many single-row inserts when processing large datasets.

## 7. Selecting data

**`select()`** builds a SELECT query. It is the core of SQLAlchemy Core read operations.

**Universal syntax**

```python
stmt = select(table1, table2).where(condition)
result = connection.execute(stmt)
```

**Applied syntax (Core)**

```python
from sqlalchemy import select

stmt = select(payment).where(payment.c.payment_id == 1)
result = connection.execute(stmt)
row = result.fetchone()
```

**Applied syntax with ORM and Session**

```python
Session = sessionmaker(bind=engine)
session = Session()

# Query all payments
all_payments = session.query(Payment).all()

# Query with condition
payment = session.query(Payment).filter(Payment.payment_id == 1).first()

# Close when done
session.close()
```

### Selecting specific columns

**Universal syntax**

```python
stmt = select(table.c.col1, table.c.col2)
```

**Applied syntax (Core)**

```python
stmt = select(payment.c.payment_id, payment.c.amount)
result = connection.execute(stmt)
```

**Applied syntax with ORM and Session**

```python
# Get only specific columns
result = session.query(Payment.payment_id, Payment.amount).all()
```

### Selecting with multiple tables

```python
stmt = select(payment.c.payment_id, customer.c.first_name)
stmt = stmt.select_from(payment.join(customer, payment.c.customer_id == customer.c.customer_id))
```

## 8. Filtering, ordering, and limiting

**`where()`** filters rows. **`order_by()`** sorts results. **`limit()`** caps the number of results.

**Universal syntax**

```python
stmt = select(table).where(condition).order_by(column).limit(n)
```

**Applied syntax (Core)**

```python
stmt = (
    select(payment)
    .where(payment.c.amount > 100)
    .order_by(payment.c.amount.desc())
    .limit(5)
)
```

**Applied syntax with ORM and Session**

```python
Session = sessionmaker(bind=engine)
session = Session()

result = (
    session.query(Payment)
    .filter(Payment.amount > 100)
    .order_by(Payment.amount.desc())
    .limit(5)
    .all()
)

session.close()
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

## 9. Joins and relationships

**Joins** combine rows from multiple tables using related columns.

**Universal syntax**

```python
stmt = select(table1, table2).select_from(table1.join(table2, table1.c.id == table2.c.id))
```

**Applied syntax (Core)**

```python
stmt = select(payment.c.payment_id, customer.c.first_name).select_from(
    payment.join(customer, payment.c.customer_id == customer.c.customer_id)
)
```

**Applied syntax with ORM and Session**

```python
Session = sessionmaker(bind=engine)
session = Session()

# Query with join
result = (
    session.query(Payment, Customer)
    .join(Customer, Payment.customer_id == Customer.customer_id)
    .all()
)

session.close()
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

## 10. Aggregations and SQL functions

**SQLAlchemy Core includes SQL functions for aggregates and calculations.**

**Universal syntax**

```python
from sqlalchemy import func

stmt = select(func.count(table.c.id))
```

**Applied syntax (Core)**

```python
from sqlalchemy import func

stmt = select(func.count(payment.c.payment_id).label("total_payments"))
result = connection.execute(stmt)
```

**Applied syntax with ORM and Session**

```python
from sqlalchemy import func

result = session.query(func.count(Payment.payment_id)).scalar()
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

**Applied syntax (Core)**

```python
stmt = select(
    payment.c.customer_id,
    func.sum(payment.c.amount).label("total_amount"),
).group_by(payment.c.customer_id)
```

**Applied syntax with ORM and Session**

```python
result = (
    session.query(
        Payment.customer_id,
        func.sum(Payment.amount).label("total_amount")
    )
    .group_by(Payment.customer_id)
    .all()
)
```

## 11. Set operations

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

## 12. Updating and deleting

### Update

**Universal syntax**

```python
stmt = table.update().where(condition).values({...})
connection.execute(stmt)
```

**Applied syntax (Core)**

```python
from sqlalchemy import update

stmt = (
    payment.update()
    .where(payment.c.payment_id == 1)
    .values(amount=275.00)
)

connection.execute(stmt)
```

**Applied syntax with ORM and Session**

```python
Session = sessionmaker(bind=engine)
session = Session()

payment = session.query(Payment).filter(Payment.payment_id == 1).first()
payment.amount = 275.00
session.commit()
session.close()
```

### Delete

**Universal syntax**

```python
stmt = table.delete().where(condition)
connection.execute(stmt)
```

**Applied syntax (Core)**

```python
from sqlalchemy import delete

stmt = delete(payment).where(payment.c.payment_id == 1)
connection.execute(stmt)
```

**Applied syntax with ORM and Session**

```python
Session = sessionmaker(bind=engine)
session = Session()

payment = session.query(Payment).filter(Payment.payment_id == 1).first()
session.delete(payment)
session.commit()
session.close()
```

## 13. Transactions and execution patterns

**Transactions** ensure atomic database operations. SQLAlchemy allows explicit transaction control.

**Universal syntax (Core)**

```python
with engine.begin() as connection:
    connection.execute(stmt1)
    connection.execute(stmt2)
```

**Applied syntax (Core)**

```python
with engine.begin() as connection:
    connection.execute(payment.insert().values(customer_id=3, amount=300.00, payment_date="2024-01-25"))
    connection.execute(
        payment.update()
        .where(payment.c.payment_id == 3)
        .values(amount=325.00)
    )
```

**Applied syntax with ORM and Session**

```python
Session = sessionmaker(bind=engine)
session = Session()

try:
    new_payment = Payment(customer_id=3, amount=300.00, payment_date="2024-01-25")
    session.add(new_payment)
    session.flush()  # Get the ID without committing
    
    updated = session.query(Payment).filter(Payment.payment_id == new_payment.payment_id).first()
    updated.amount = 325.00
    
    session.commit()
except Exception as e:
    session.rollback()
    print(f"Transaction failed: {e}")
finally:
    session.close()
```

### Common execution patterns

```python
result = connection.execute(stmt)
rows = result.fetchall()
for row in rows:
    print(row)
```

> ⚠️ A transaction is committed only when the context exits successfully. If an exception occurs, the transaction is rolled back.

## 14. Core vs ORM

| Feature | SQLAlchemy Core | SQLAlchemy ORM |
| --- | --- | --- |
| Primary style | SQL expression language | mapped Python classes |
| Data model | `Table` and `Column` | `Declarative` models / mapped classes |
| Best for | explicit SQL and schema control | application-level object modeling |
| Query style | `select(table)` | `session.query(Model)` or `session.execute(select(Model))` |
| Relationship handling | manual join expressions | object-level relationships |
| Session needed | No (use Connection) | Yes |

**Core** is lower-level and more SQL-centric. **ORM** adds object mapping and higher-level relationship behavior.

## 15. Quick Reference

| Concept | Universal syntax | Applied syntax |
| --- | --- | --- |
| Engine | `create_engine("dialect+driver://...")` | `create_engine("sqlite:///shop.db")` |
| Base class | `Base = declarative_base()` | `from sqlalchemy.ext.declarative import declarative_base` |
| SessionMaker | `Session = sessionmaker(bind=engine)` | Created once at app startup |
| Session | `session = Session()` | New session per request |
| Table | `Table("name", metadata, Column(...))` | `payment = Table("payment", metadata, Column("payment_id", Integer, primary_key=True))` |
| ORM Class | `class Model(Base): __tablename__ = 'table'` | `class Payment(Base): __tablename__ = 'payment'` |
| Insert | `table.insert().values({...})` | `payment.insert().values(customer_id=1, amount=250.00)` |
| Insert ORM | `session.add(obj)` | `session.add(Payment(...))` |
| Select | `select(table).where(condition)` | `select(payment).where(payment.c.payment_id == 1)` |
| Select ORM | `session.query(Model)` | `session.query(Payment).filter(Payment.payment_id == 1).first()` |
| Select columns | `select(table.c.col1, table.c.col2)` | `select(payment.c.payment_id, payment.c.amount)` |
| Filter | `.where(condition)` | `.where(payment.c.amount > 100)` |
| Order | `.order_by(column)` | `.order_by(payment.c.amount.desc())` |
| Limit | `.limit(n)` | `.limit(5)` |
| Join | `.join(table, condition)` | `payment.join(customer, payment.c.customer_id == customer.c.customer_id)` |
| Aggregate | `func.count(table.c.id)` | `func.sum(payment.c.amount)` |
| Update | `table.update().where(condition).values({...})` | `payment.update().where(payment.c.payment_id == 1).values(amount=275.00)` |
| Update ORM | `obj.column = value` | `payment.amount = 275.00` |
| Delete | `table.delete().where(condition)` | `delete(payment).where(payment.c.payment_id == 1)` |
| Delete ORM | `session.delete(obj)` | `session.delete(payment)` |
| Union | `query1.union(query2)` | `union(q1, q2)` |
| Union All | `query1.union_all(query2)` | `union_all(q1, q2)` |

This README is intended as a clean reference for SQLAlchemy Core fundamentals with practical examples using a consistent `payment`, `customer`, and `order`-style schema. It now includes both Core and ORM approaches for easy comparison.
