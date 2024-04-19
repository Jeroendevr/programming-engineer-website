---
authors:
  - jeroendevr
date: 2024-04-19
title: __super()__ considered a case for dependency injection
tags:
  - python
  - test
  - fastapi
  - sqlalchemy
---
# __super()__ considered a case for dependency injection
In following of [python super() considered harmfull](https://fuhm.net/super-harmful/) and `__super()__ considered __super()__` by Raymond Hettinger [Blog](https://rhettinger.wordpress.com/2011/05/26/super-considered-super/) [Video](https://youtu.be/EiOglTERPEo?si=6pdt8kMhLols9Wal) I came to the idea to use it for dependency injection.
# `__super()__`
If you haven't seen and used `__super()__` before, see the video from Hettinger or the [docs.python](https://docs.python.org/3.13/library/functions.html#super) . In short returns a object that delegates method calls to a parent or sibling of the same type. 
# Dependency Injection
For those unaware of the term Dependency Injection or DI for short. It is a pattern to to decouple dependencies (it can be a database, file, class or function) from a certain implementation. So it makes it easier to test or to change later on. 
# Reason to use `__super()__` 
For a small project (<150 hrs) I was working on, I wanted to achieve the following
- Write production code (obviously)
- Write Tests (Unit and Integration)
- Unit Tests should be independent from production 
- Writing tests should be easy
- No clunky adjustment to production code in order to make the testing easy [^1]
- Unittest should be fast
- Tests should be independent of production data

<!-- more -->
## Motivation of Reason

### No adjustment to production in order to make it testable
An insight I got from reading a book about testing[^2].  Is that code is a liability. I want to minimize production code. (Not in a strict sense that fewer lines are better) But the more code the more possibilities in can fail. 
### Tests should be independent of production data and production system
**Production Data**
This one speaks for itself, I think. As production can change over time you cannot make assumptions about the data and you will continuously ask yourself questions *is the test wrong or has production data changed* 

**Production system**
For the production system it is a little bit different. In this case there is a database. So I could use it to write my tests against it. But I certainly want to avoid the possibility that my tests drops a database table and my production data is gone. (Although would be good lesson to make backups. ) Or modifies existing records.
### Unittest should be fast 
When adding functionality I sometimes work test-driven and the tests lead me to bugs in the program. So I run the unittests very frequently. The amount of tests I run are something in the range of 10-100 times *per issue* for a single test of class of tests and around 1-5 times for the entire test suite again this is also *per issue*.

Adding a 5 second during test could lead to a possible increase of 5 minutes of waiting time per issue (5 secs per issue * 60 runs = 300 sec = 5 minutes). 
*Benchmark 26 tests in 3.4 seconds for the current project.* 
#### Fast ≈ local
Fast tests are almost equal to local test. You could create a online database identical to the production database but this *can* lead to longer running times for tests as a connection has to be made. In my case I had only two tests when ran it took about 2 seconds. (They could be run in parallel)
# Design choices
## Design choices 1
Have a local database for the majority of the tests. In order to keep te tests fast. This could have been a local PostgreSQL in a docker environment but a full PostgreSQL was not needed for this test setup. But this makes the testing environment unnecessarily complex. Besides that I am not proficient with docker ecosystem.
## Design choice 2
With native support in the standard library for SQLite included in the python language, I chose that one. It is fast, easily run locally. 
## Design choice 3
I was already considering a ORM but having multiple databases strengthened the choice for the ORM SQLAlchemy in this case. 
# Code-ing
Lets look at the steps taken from idea to code and a couple of refactors and considerations. 
The development stack involved was: 
- Python 
- FastAPI
- PostgreSQL
- SQLAlchemy
# Bad design
## Database connected to the class
```python
class Records:

    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

    def get_database_record(self):
        with Session(self.engine) as session:
            stmt = select(OnlyProdDB.item)
            items = session.scalar(stmt)

            return items

```
Now the implementation of the class is connected to the database. How would we reuse the methods of the `Records` class or test it? Overriding the engine perhaps?

```python
import unittest

from sqlalchemy import create_engine

from super_dependency_injection.prod import Records


class TestProd(unittest.TestCase):
    def test_get_record(self):
        test_record = Records()
        test_record.engine = create_engine("localtestengine") # Overriding engine

```
Which conforms to the design decision to conform that tests should be independent of production code. But the engine is now connected to the class which we don't want as the engine is not reusable and the Records class as a whole.
## Database defined within a method
Lets have the engine defined somewhere else and import it
```python
def engine():
    return "postgresql+psycopg://scott:tiger@localhost/test"


class Records:

    engine = create_engine(engine())

    def get_database_record(self):
        with Session(self.engine) as session:
            stmt = select(OnlyProdDB.item)
            items = session.scalar(stmt)

            return items
```
Much better and we can still override it during test, like in the previous example. 

## Database defined in the constructor / `__init__` method
The first example where dependency injection comes in. We supply the engine as a parameter during initialisation of a `Record` class.

```python
def engine():
    return "postgresql+psycopg://scott:tiger@localhost/test"


class Records:

    def __init__(self, engine) -> None:
        self.engine = create_engine(engine)

    def get_database_record(self):
        with Session(self.engine) as session:
            stmt = select(OnlyProdDB.item)
            items = session.scalar(stmt)

            return items
```
When testing we can supply our own engine
```Python
def test_engine():
    return "localtestengine"


class TestProd(unittest.TestCase):
    def test_get_record(self):
        test_record = Records(engine=test_engine)
        test_record.engine  # localtestengine
```

## Database defined through Inheritance
```Python
class Database:
    engine = create_engine("postgresql+psycopg://scott:tiger@localhost/test")


class Records(Database):

    def get_database_record(self):
        with Session(self.engine) as session:
            stmt = select(OnlyProdDB.item)
            items = session.scalar(stmt)

            return items


class Tables(Database):

    def get_database_table(self):
	    with Session(self.engine) as session:
		    pass
```
Now multiple *classes* can inherit from Database and reuse the defined behaviour in the superclass `Database`
But for testing we need to override the engine property like in the first two examples. But it does not have to and this is where Multiple Cooperative Inheritance comes in. 

## Database defined through inheritance and testing through multiple cooperative inheritance
We can keep the code in production the same as defined in the previous example and reuse the same Database for another class `Tables`.
```Python
class Database:
    engine = create_engine("postgresql+psycopg://scott:tiger@localhost/test")


class Records(Database):

    def get_database_record(self):
        with Session(self.engine) as session:
            stmt = select(OnlyProdDB.item)
            items = session.scalar(stmt)

            return items


class Tables(Database):

    def get_database_table(self):
	    with Session(self.engine) as session:
		    pass
```

But we can test it using 
```python
import unittest

from sqlalchemy import create_engine

from super_dependency_injection.prod import Database, Records


class TestDatabase(Database):
    engine = create_engine("localtestengine")


class TestRecords(Records, TestDatabase):
    pass


class TestProd(unittest.TestCase):
    def test_get_record(self):
        test_record = TestRecords()
        test_record.engine  # localtestengine

```

Defining `TestRecords` it inherited from two classes `Records` and `TestDatabase`. When calling the `enginge` property on `test_record` it dit not yield `"postgresql+psycopg://scott:tiger@localhost/test"` but `localtestengine` as it was the next in line. 

```mermaid
---
title: Method Resolution
---

flowchart BT
    Records --> Database
    TestDatabase --> Database
    TestRecord --> Records
    TestRecord --> TestDatabase

    Records -- 2. engine ---TestDatabase
    TestRecord -- 1. engine ---Records

```
when `engine` property is called on`TestRecord` first `Records` is checked, then `TestDatabase` (and then `Database` through Records`)
I found this quite a neat use of Multiple Inheritance. 
# Hindsight
After setting up the project, I had missed a design pattern available in FastAPI (seeing ArjanCodes)[^3]
the usage of `depends` was new to me. 

## Example with fastapi.Depends

```python
"""prod_v5.py"""

from fastapi import Depends, FastAPI
from sqlalchemy import String, create_engine, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column, sessionmaker

app = FastAPI()


def get_db():
    engine = create_engine("postgresql+psycopg://scott:tiger@localhost/test")
    session = sessionmaker(engine)
    database = session()
    try:
        yield database
    finally:
        database.close()


@app.get("/record")
def get_record(session: Session = Depends(get_db)):
    db_record = get_database_record(session)
    return db_record


class Base(DeclarativeBase):
    pass


class OnlyProdDB(Base):
    __tablename__ = "only_prod_db"

    id: Mapped[int] = mapped_column(primary_key=True)
    item: Mapped[str] = mapped_column(String)


def get_database_record(session):
    db_item = session.query(OnlyProdDB.item)
    return db_item
```

```python
import unittest

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from super_dependency_injection.prod_v5 import get_database_record

engine = create_engine("localtestengine")
session = sessionmaker(engine)
database = session()


class TestProd(unittest.TestCase):
    def test_get_record(self):
        test_record = get_database_record(database)
        test_record

```

This pattern is  allows for more functional style programming as it does not need the creation of classes See the previous example of`Records` inheriting from  `Database`
Another benefit I see is that querying is more neat as I don't need to set up a context manager (`with Session() as session:` ) it is less text and more readable. So I will definitely try to make use of this pattern when using **FastAPI** in another project.

**Sources**
[^1]: Good explanation of DI can be found at ArjanCodes [blog](https://www.arjancodes.com/blog/python-dependency-injection-best-practices/) 
[^2]:
	In the spirit of Vladimir Khorikov [Unit Testing Principles, Practices, and Patterns](https://www.manning.com/books/unit-testing) *Remember, all code, including test code, is a liability*.  I want to minimize code and especially production. Avoid that production fails because of an adjustment made so I can test it. 
[^3]: [ArjanCodes discussing FastAPI YouTube](https://youtu.be/XlnmN4BfCxw?si=FfNFZIYyTTdKtIDF) 
