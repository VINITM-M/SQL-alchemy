# Working with Multiple Tables in SQLAlchemy

## Overview

SQLAlchemy is an open-source library that provides a set of tools for working with relational databases. It offers a high-level Object-Relational Mapping (ORM) interface as well as a lower-level SQL Expression Language (Core) interface.

When working with a relational database, it's common to have multiple tables that are related to each other. For example, we might have a "students" table and a "fees" table, where each student has multiple fees associated with them. In SQLAlchemy Core, we can define multiple tables and create relationships between them.

## Table of Contents

- [1. Defining Multiple Tables](#1-defining-multiple-tables)
- [2. Creating Relationships](#2-creating-relationships)
- [3. Inserting Data](#3-inserting-data)
- [4. Querying Related Data](#4-querying-related-data)
- [5. Joining Multiple Tables](#5-joining-multiple-tables)
- [6. Multiple-Table Delete Operations](#6-multiple-table-delete-operations)
- [7. Advanced Query Patterns](#7-advanced-query-patterns)

---

## 1. Defining Multiple Tables

### Basic Table Definition

```python
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base, relationship, sessionmaker

# Create engine
engine = create_engine('sqlite:///database.db', echo=True)

# Create base class
Base = declarative_base()

# Define Student table
class Student(Base):
    __tablename__ = 'students'
    
    id = Column(Integer, primary_key=True)
    name = Column(String)
    age = Column(Integer)
    
    # Define relationship
    fees = relationship("Fee", back_populates="student")

# Define Fee table
class Fee(Base):
    __tablename__ = 'fees'
    
    id = Column(Integer, primary_key=True)
    amount = Column(Integer)
    student_id = Column(Integer, ForeignKey('students.id'))
    
    # Define relationship
    student = relationship("Student", back_populates="fees")

# Create all tables
Base.metadata.create_all(engine)
```

### Key Concepts

| Concept | Description |
| --- | --- |
| `__tablename__` | Maps the Python class to a database table |
| `Column` | Defines table columns and their data types |
| `primary_key=True` | Marks the primary key column |
| `ForeignKey()` | Creates a foreign key constraint to another table |
| `relationship()` | Establishes a relationship between two tables |

---

## 2. Creating Relationships

### One-to-Many Relationship

**One Student → Many Fees**

```python
class Student(Base):
    __tablename__ = 'students'
    
    id = Column(Integer, primary_key=True)
    name = Column(String)
    age = Column(Integer)
    
    # One student has many fees
    fees = relationship("Fee", back_populates="student")

class Fee(Base):
    __tablename__ = 'fees'
    
    id = Column(Integer, primary_key=True)
    amount = Column(Integer)
    student_id = Column(Integer, ForeignKey('students.id'))
    
    # Many fees belong to one student
    student = relationship("Student", back_populates="fees")
```

### Understanding `back_populates`

`back_populates` creates a bidirectional relationship:

```python
# Access student from fee
fee = session.query(Fee).first()
print(fee.student.name)  # ← Access student through fee

# Access fees from student
student = session.query(Student).first()
print(student.fees)  # ← Access fees through student
```

---

### Understaing this below syntax

Base.metadata.create_all(engine)

base = declarative() 
Base → Contains all model/table definitions registered through declarative_base().
metadata → Stores the schema information (table names, columns, primary keys, foreign keys, constraints, relationships metadata).
create_all() → Generates and executes SQL CREATE TABLE statements for all tables present in the metadata.
engine → Database connection through which the SQL is executed.

In one sentence:

Base.metadata.create_all(engine) tells SQLAlchemy to read all model definitions stored in Base.metadata and create the corresponding tables in the connected database using engine if they do not already exist.


## 3. Inserting Data

### Insert Single Student with Multiple Fees

```python
from sqlalchemy.orm import sessionmaker

# Create session
Session = sessionmaker(bind=engine)
session = Session()

# Create student
s1 = Student(name='John Doe', age=20)

# Add fees to student
s1.fees = [
    Fee(amount=100),
    Fee(amount=200)
]

# Commit to database
session.add(s1)
session.commit()

print("Student and fees added successfully!")
```

### Insert Multiple Students with Multiple Fees

```python
Session = sessionmaker(bind=engine)
session = Session()

# Create students
students = [
    Student(name='John Doe', age=20),
    Student(name='Jane Smith', age=22),
    Student(name='Bob Brown', age=25),
    Student(name='Alice Jones', age=23)
]

# Create fees
fees = [
    Fee(amount=5000),
    Fee(amount=6000),
    Fee(amount=4500),
    Fee(amount=5500)
]

# Associate fees with students
for i in range(len(students)):
    students[i].fees.append(fees[i])
    session.add(students[i])

# Commit all at once
session.commit()

print("All students and fees added!")
```

---

## 4. Querying Related Data

### Query Student and Access Related Fees

```python
Session = sessionmaker(bind=engine)
session = Session()

# Query by student name
student = session.query(Student).filter_by(name='John Doe').first()

print(f"Student: {student.name}, Age: {student.age}")
print("Fees:")
for fee in student.fees:
    print(f"  - Amount: {fee.amount}")
```

### Query with joinedload (Eager Loading)

Prevents N+1 query problem by loading related objects in one query:

```python
from sqlalchemy.orm import joinedload

# Query all students and their fees in one query
students = session.query(Student).options(joinedload(Student.fees)).all()

for student in students:
    print(f'{student.name} ({student.age}):')
    for fee in student.fees:
        print(f'  - Amount: {fee.amount}')
```

**Why use joinedload?**
- Without it: 1 query for students + N queries for each student's fees = N+1 queries
- With it: Only 1 query that joins both tables

---

## 5. Joining Multiple Tables

### Inner Join

```python
from sqlalchemy.orm import aliased

# Create aliases for clean code
student = aliased(Student)
fee = aliased(Fee)

# Join students and fees
results = (
    session.query(student, fee)
    .join(fee, student.id == fee.student_id)
    .order_by(student.name)
    .all()
)

# Print results
for s, f in results:
    print(f'{s.name} ({s.age}): Amount = {f.amount}')
```

### Left Outer Join

```python
from sqlalchemy.orm import aliased

student = aliased(Student)
fee = aliased(Fee)

# Left join - includes students even if they have no fees
results = (
    session.query(student, fee)
    .join(fee, student.id == fee.student_id, isouter=True)
    .order_by(student.name)
    .all()
)

for s, f in results:
    if f:
        print(f'{s.name}: Amount = {f.amount}')
    else:
        print(f'{s.name}: No fees')
```

---

## 6. Multiple-Table Delete Operations

### Delete with Condition on Related Table

```python
from sqlalchemy import delete

# Delete books where publisher_name is "Springer"
# Using multiple-table join condition

# Define tables
# delete_stmt = delete(BOOKS).where(
#     BOOKS.c.book_id == book_publisher.c.publisher_id
# ).where(
#     book_publisher.c.publisher_name == 'Springer'
# )

# For ORM (using classes):
students_to_delete = session.query(Student).filter(
    Student.name == 'John Doe'
).all()

for student in students_to_delete:
    session.delete(student)

session.commit()
print("Records deleted!")
```

### SQLAlchemy Core Delete Example

```python
from sqlalchemy import delete, Table, MetaData

# Using SQLAlchemy Core
metadata = MetaData()

# Assume tables are defined
books = Table('books', metadata, autoload_with=engine)
book_publisher = Table('book_publisher', metadata, autoload_with=engine)

# Delete query using Core
delete_stmt = (
    delete(books)
    .where(books.c.book_id == book_publisher.c.publisher_id)
    .where(book_publisher.c.publisher_name == 'Springer')
)

# Execute the delete
engine.execute(delete_stmt)

# Fetch remaining records
from sqlalchemy import text
sql = text("SELECT * FROM books")
result = engine.execute(sql).fetchall()

for record in result:
    print(record)
```

---

## 7. Advanced Query Patterns

### Query with Filter and Order By

```python
# Get all fees for students aged 20-25, ordered by amount
results = (
    session.query(Student.name, Fee.amount)
    .join(Fee)
    .filter(Student.age >= 20, Student.age <= 25)
    .order_by(Fee.amount.desc())
    .all()
)

for name, amount in results:
    print(f'{name}: {amount}')
```

### Aggregate Queries

```python
from sqlalchemy import func

# Get total fees per student
results = (
    session.query(Student.name, func.sum(Fee.amount).label('total'))
    .join(Fee)
    .group_by(Student.id)
    .all()
)

for name, total in results:
    print(f'{name}: Total Fees = {total}')
```

### Count Fees per Student

```python
from sqlalchemy import func

# Count fees per student
results = (
    session.query(Student.name, func.count(Fee.id).label('fee_count'))
    .join(Fee)
    .group_by(Student.id)
    .all()
)

for name, count in results:
    print(f'{name}: {count} fees')
```

---

## Complete Working Example

```python
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base, relationship, sessionmaker, joinedload
from sqlalchemy import func

# Setup
engine = create_engine('sqlite:///example.db', echo=False)
Base = declarative_base()

# Define Models
class Student(Base):
    __tablename__ = 'students'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    age = Column(Integer)
    fees = relationship('Fee', back_populates='student')

class Fee(Base):
    __tablename__ = 'fees'
    id = Column(Integer, primary_key=True)
    amount = Column(Integer)
    student_id = Column(Integer, ForeignKey('students.id'))
    student = relationship('Student', back_populates='fees')

# Create tables
Base.metadata.create_all(engine)

# Create session
Session = sessionmaker(bind=engine)
session = Session()

# Insert data
students = [
    Student(name='John Doe', age=20),
    Student(name='Jane Smith', age=22),
    Student(name='Bob Brown', age=25)
]

fees = [
    Fee(amount=5000),
    Fee(amount=6000),
    Fee(amount=4500)
]

for i in range(len(students)):
    students[i].fees.append(fees[i])
    session.add(students[i])

session.commit()

# Query 1: Get student and all fees
print("=== Query 1: Student with Fees ===")
student = session.query(Student).filter_by(name='John Doe').first()
print(f'{student.name}: Fees = {[f.amount for f in student.fees]}')

# Query 2: Join query
print("\n=== Query 2: Joined Query ===")
results = session.query(Student.name, Fee.amount).join(Fee).all()
for name, amount in results:
    print(f'{name}: {amount}')

# Query 3: Aggregate
print("\n=== Query 3: Total Fees per Student ===")
results = (
    session.query(Student.name, func.sum(Fee.amount).label('total'))
    .join(Fee)
    .group_by(Student.id)
    .all()
)
for name, total in results:
    print(f'{name}: {total}')

# Query 4: Using joinedload
print("\n=== Query 4: Eager Load (joinedload) ===")
students = session.query(Student).options(joinedload(Student.fees)).all()
for s in students:
    print(f'{s.name}: {len(s.fees)} fees')

session.close()
```

---

## Best Practices

| Practice | Description |
| --- | --- |
| **Use relationships** | Define relationships bidirectionally with `back_populates` |
| **Use joinedload** | Avoid N+1 queries by eager loading related objects |
| **Use context manager** | Always close sessions with `with` statements or `.close()` |
| **Use transactions** | Commit/rollback multiple operations atomically |
| **Use aliases** | Use `aliased()` for complex queries with same table joins |
| **Index foreign keys** | Add indexes on foreign key columns for better performance |

---

## Summary

- **Define tables** using SQLAlchemy ORM classes
- **Create relationships** using `relationship()` and `ForeignKey()`
- **Insert data** using sessions and `session.add()` / `session.commit()`
- **Query related data** using `.join()` and relationships
- **Optimize queries** using `joinedload()` to avoid N+1 problems
- **Delete related data** using filters on related tables
