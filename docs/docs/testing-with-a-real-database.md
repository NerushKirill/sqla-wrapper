# Testing with a real database

"Testing with a real database, was not that supposed to be a no-no?" - you might be thinking. Many tutorials have always reccomended mocking your database so your unit-tests are fast.

More often than not, thiugh, that is bad advice. For databases like PostgreSQL or MySQLfor whatever short-term gains you might get, you lose in long-term maintainability.

Mocking your database is not only a lot of overhead but, after a while, the mocks no longer reflect the actual code and what is happening on the real database server.

We need each test to be isolated from others: you should never have to think about what other tests have put in the database. However, creating tables, loading fixtures, and dropping those tables for each test is too slow. The good news is, you don't need to do it for each test. You will not have to worry about database performance issues if you follow this setup.


## 1. Manually create an empty database for testing

First, you need to manually create a new (and empty) database for your tests to use. This could mean manually running a command like `createdb myapp-tests` or adding another database service in your development `docker-compose.yml` file.

You should use the same type of database as the one used by your application. Otherwise, your tests could be affected by implementation details of the database engines, like the default order of null values, and not being able to test what would happen in production.


## 2. Prepare the database before running the test suite

The test database exists but is empty at this point, so before we run any test, we need to create all the tables and insert the minimal fixture data necessary to run the application.

We must configure our tests to do this every time we run the tests, but only once. There is no need to run any migrations because we can create all the tables from the models definition. You can however "stamp" it with the latest revision if some test expects the `alembic_version` table to exists.

### pytest

With **pytest** this can be done with a session-scoped fixture:

```python
# conftest.py
import pytest

from myapp.models import db, alembic, load_fixture_data

@pytest.fixture(scope="session")
def dbsetup():
    assert "_test" in db.url  # better to be safe than sorry
    db.create_all()
    alembic.stamp()  # optional
    load_fixture_data()  # optional
    yield
    db.drop_all()

```

### unittest

With **unitest**, we do it with the `setUpClass` and `tearDownClass` class methods in a base class. Other test cases must inherit from this class to make it work.

```python
# base.py
import unittest

from myapp.models import db, alembic, load_fixture_data

class DBTestCase(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        assert "_test" in db.url  # better to be safe than sorry
        db.create_all()
        alembic.stamp()  # optional
        load_fixture_data()  # optional

    @classmethod
    def tearDownClass(cls):
        db.drop_all()

```


## 3. Run each test inside a transaction and rollback at the end

This part is the main trick, and **sqla-wrapper** includes a `db.test_transaction()` method that does it for you:

```python
trans = db.test_transaction()
...
trans.close()
```

By wrapping each test in a root transaction, we make sure that it will not alter the state of the database, even if we commit during the test since the transaction is rolled back on test teardown.

There is only one caveat: if you want to allow tests to also use rollbacks within them, your database must support SAVEPOINT.

!!! Note

    `test_transaction()` internally follow this recipe:

    ```python
    # 1. Start a transaction on a new connection
    connection = db.engine.connect()
    trans = connection.begin()

    # 2. Bind a new session to this connection
    session = db.Session(bind=connection)
    # and registers it as the new scoped session

    # 3. If your database supports it, start a savepoint to allow rollbacks within a test
    nested = connection.begin_nested()

    @event.listens_for(session, "after_transaction_end")
    def end_savepoint(session, transaction):
        if not nested.is_active:
            nested = connection.begin_nested()

    # 4. Run the test
    # ...

    # 5. Finally, rollback any changes and close the connection.
    session.close()
    trans.rollback()
    connection.close()
    ```

    This recipe is what [SQLAlchemy documentation](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html#session-external-transaction) recommends and even test in their own CI to ensure that it remains working as expected.


### pytest

With **pytest**, we can use a regular fixture and make it *depend* on the `dbsetup()` fixture we created before. Although this new fixture will run (automatically) before each test, the setup fixture will run only once, as we need to.

```python
# conftest.py
import pytest

from myapp.models import db, alembic, load_fixture_data

@pytest.fixture(scope="session")
def dbsetup():
    # ...

@pytest.fixture()
def dbs(dbsetup):
    trans = db.test_transaction()
    # Or, if you need to rollback in your tests
    #   = db.test_transaction(savepoint=True)
    yield
    trans.close()

```

Then, we use that database session fixture for the tests that require interacting with the database. You can safely use the "global" scoped session, because it nows point to the test session.

```python
# test_something.py
from myapp.models import db

def test_something(dbs):
    db.s.execute(some_stmt)
    db.s.commit()
    ...

```

### unittest

With **unittest** we use the `setUp()` and `tearDown()` methods

```python
# base.py
import unittest

from myapp.models import db, alembic, load_fixture_data

class DBTestCase(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        # ...

    @classmethod
    def tearDownClass(cls):
        # ...

    def setUp(self):
        self._trans = db.test_transaction()
        # Or, if you need to rollback in your tests
        #   = db.test_transaction(savepoint=True)

    def tearDown(self):
        self._trans.close()

    # ...

```

Then, we inherit from that class for the tests that require interacting with the database.

```python
# test_something.py
from myapp.models import db
from .base import DBTestCase

class TestSomething(DBTestCase):
    def test_something(self):
        db.s.execute(some_stmt)
        db.s.commit()
        # ...

```

## API

::: sqla_wrapper.TestTransaction
    :members:
