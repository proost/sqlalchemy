.. _connections_toplevel:

====================================
Working with Engines and Connections
====================================

.. module:: sqlalchemy.engine

This section details direct usage of the :class:`_engine.Engine`,
:class:`_engine.Connection`, and related objects. Its important to note that when
using the SQLAlchemy ORM, these objects are not generally accessed; instead,
the :class:`.Session` object is used as the interface to the database.
However, for applications that are built around direct usage of textual SQL
statements and/or SQL expression constructs without involvement by the ORM's
higher level management services, the :class:`_engine.Engine` and
:class:`_engine.Connection` are king (and queen?) - read on.

Basic Usage
===========

Recall from :doc:`/core/engines` that an :class:`_engine.Engine` is created via
the :func:`_sa.create_engine` call::

    engine = create_engine('mysql://scott:tiger@localhost/test')

The typical usage of :func:`_sa.create_engine` is once per particular database
URL, held globally for the lifetime of a single application process. A single
:class:`_engine.Engine` manages many individual :term:`DBAPI` connections on behalf of
the process and is intended to be called upon in a concurrent fashion. The
:class:`_engine.Engine` is **not** synonymous to the DBAPI ``connect`` function, which
represents just one connection resource - the :class:`_engine.Engine` is most
efficient when created just once at the module level of an application, not
per-object or per-function call.

.. sidebar:: tip

    When using an :class:`_engine.Engine` with multiple Python processes, such as when
    using ``os.fork`` or Python ``multiprocessing``, it's important that the
    engine is initialized per process.  See :ref:`pooling_multiprocessing` for
    details.

The most basic function of the :class:`_engine.Engine` is to provide access to a
:class:`_engine.Connection`, which can then invoke SQL statements.   To emit
a textual statement to the database looks like::

    from sqlalchemy import text

    with engine.connect() as connection:
        result = connection.execute(text("select username from users"))
        for row in result:
            print("username:", row['username'])

Above, the :meth:`_engine.Engine.connect` method returns a :class:`_engine.Connection`
object, and by using it in a Python context manager (e.g. the ``with:``
statement) the :meth:`_engine.Connection.close` method is automatically invoked at the
end of the block.  The :class:`_engine.Connection`, is a **proxy** object for an
actual DBAPI connection. The DBAPI connection is retrieved from the connection
pool at the point at which :class:`_engine.Connection` is created.

The object returned is known as :class:`_engine.CursorResult`, which
references a DBAPI cursor and provides methods for fetching rows
similar to that of the DBAPI cursor.   The DBAPI cursor will be closed
by the :class:`_engine.CursorResult` when all of its result rows (if any) are
exhausted.  A :class:`_engine.CursorResult` that returns no rows, such as that of
an UPDATE statement (without any returned rows),
releases cursor resources immediately upon construction.

When the :class:`_engine.Connection` is closed at the end of the ``with:`` block, the
referenced DBAPI connection is :term:`released` to the connection pool.   From
the perspective of the database itself, the connection pool will not actually
"close" the connection assuming the pool has room to store this connection  for
the next use.  When the connection is returned to the pool for re-use, the
pooling mechanism issues a ``rollback()`` call on the DBAPI connection so that
any transactional state or locks are removed, and the connection is ready for
its next use.

.. deprecated:: 2.0 The :class:`_engine.CursorResult` object is replaced in SQLAlchemy
   2.0 with a newly refined object known as :class:`_future.Result`.

Our example above illustrated the execution of a textual SQL string, which
should be invoked by using the :func:`_expression.text` construct to indicate that
we'd like to use textual SQL.  The :meth:`_engine.Connection.execute` method can of
course accommodate more than that, including the variety of SQL expression
constructs described in :ref:`sqlexpression_toplevel`.


Using Transactions
==================

.. note::

  This section describes how to use transactions when working directly
  with :class:`_engine.Engine` and :class:`_engine.Connection` objects. When using the
  SQLAlchemy ORM, the public API for transaction control is via the
  :class:`.Session` object, which makes usage of the :class:`.Transaction`
  object internally. See :ref:`unitofwork_transaction` for further
  information.

The :class:`~sqlalchemy.engine.Connection` object provides a :meth:`_engine.Connection.begin`
method which returns a :class:`.Transaction` object.  Like the :class:`_engine.Connection`
itself, this object is usually used within a Python ``with:`` block so
that its scope is managed::

    with engine.connect() as connection:
        with connection.begin():
            r1 = connection.execute(table1.select())
            connection.execute(table1.insert(), {"col1": 7, "col2": "this is some data"})

The above block can be stated more simply by using the :meth:`_engine.Engine.begin`
method of :class:`_engine.Engine`::

    # runs a transaction
    with engine.begin() as connection:
        r1 = connection.execute(table1.select())
        connection.execute(table1.insert(), {"col1": 7, "col2": "this is some data"})

The block managed by each ``.begin()`` method has the behavior such that
the transaction is committed when the block completes.   If an exception is
raised, the transaction is instead rolled back, and the exception propagated
outwards.

The underlying object used to represent the transaction is the
:class:`.Transaction` object.  This object is returned by the
:meth:`_engine.Connection.begin` method and includes the methods
:meth:`.Transaction.commit` and :meth:`.Transaction.rollback`.   The context
manager calling form, which invokes these methods automatically, is recommended
as a best practice.

.. _connections_nested_transactions:

Nesting of Transaction Blocks
-----------------------------

.. note:: The "transaction nesting" feature of SQLAlchemy is a legacy feature
   that will be deprecated in an upcoming release.  New usage paradigms will
   eliminate the need for it to be present.

The :class:`.Transaction` object also handles "nested" behavior by keeping
track of the outermost begin/commit pair. In this example, two functions both
issue a transaction on a :class:`_engine.Connection`, but only the outermost
:class:`.Transaction` object actually takes effect when it is committed.

.. sourcecode:: python+sql

    # method_a starts a transaction and calls method_b
    def method_a(connection):
        with connection.begin():  # open a transaction
            method_b(connection)

    # method_b also starts a transaction
    def method_b(connection):
        with connection.begin(): # open a transaction - this runs in the
                                 # context of method_a's transaction
            connection.execute(text("insert into mytable values ('bat', 'lala')"))
            connection.execute(mytable.insert(), {"col1": "bat", "col2": "lala"})

    # open a Connection and call method_a
    with engine.connect() as conn:
        method_a(conn)

Above, ``method_a`` is called first, which calls ``connection.begin()``. Then
it calls ``method_b``. When ``method_b`` calls ``connection.begin()``, it just
increments a counter that is decremented when it calls ``commit()``. If either
``method_a`` or ``method_b`` calls ``rollback()``, the whole transaction is
rolled back. The transaction is not committed until ``method_a`` calls the
``commit()`` method. This "nesting" behavior allows the creation of functions
which "guarantee" that a transaction will be used if one was not already
available, but will automatically participate in an enclosing transaction if
one exists.

.. index::
   single: thread safety; transactions

.. _autocommit:

Library Level (e.g. emulated) Autocommit
==========================================

.. deprecated:: 1.4  The "autocommit" feature of SQLAlchemy Core is deprecated
   and will not be present in version 2.0 of SQLAlchemy.   DBAPI-level
   AUTOCOMMIT is now widely available which offers superior performance
   and occurs transparently.  See :ref:`migration_20_autocommit` for background.

.. note:: This section discusses the feature within SQLAlchemy that automatically
   invokes the ``.commit()`` method on a DBAPI connection, however this is against
   a DBAPI connection that **is itself transactional**.  For true AUTOCOMMIT,
   see the next section :ref:`dbapi_autocommit`.

The previous transaction example illustrates how to use :class:`.Transaction`
so that several executions can take part in the same transaction. What happens
when we issue an INSERT, UPDATE or DELETE call without using
:class:`.Transaction`?  While some DBAPI
implementations provide various special "non-transactional" modes, the core
behavior of DBAPI per PEP-0249 is that a *transaction is always in progress*,
providing only ``rollback()`` and ``commit()`` methods but no ``begin()``.
SQLAlchemy assumes this is the case for any given DBAPI.

Given this requirement, SQLAlchemy implements its own "autocommit" feature which
works completely consistently across all backends. This is achieved by
detecting statements which represent data-changing operations, i.e. INSERT,
UPDATE, DELETE, as well as data definition language (DDL) statements such as
CREATE TABLE, ALTER TABLE, and then issuing a COMMIT automatically if no
transaction is in progress. The detection is based on the presence of the
``autocommit=True`` execution option on the statement.   If the statement
is a text-only statement and the flag is not set, a regular expression is used
to detect INSERT, UPDATE, DELETE, as well as a variety of other commands
for a particular backend::

    conn = engine.connect()
    conn.execute(text("INSERT INTO users VALUES (1, 'john')"))  # autocommits

The "autocommit" feature is only in effect when no :class:`.Transaction` has
otherwise been declared.   This means the feature is not generally used with
the ORM, as the :class:`.Session` object by default always maintains an
ongoing :class:`.Transaction`.

Full control of the "autocommit" behavior is available using the generative
:meth:`_engine.Connection.execution_options` method provided on :class:`_engine.Connection`
and :class:`_engine.Engine`, using the "autocommit" flag which will
turn on or off the autocommit for the selected scope. For example, a
:func:`_expression.text` construct representing a stored procedure that commits might use
it so that a SELECT statement will issue a COMMIT::

    with engine.connect().execution_options(autocommit=True) as conn:
        conn.execute(text("SELECT my_mutating_procedure()"))

.. _dbapi_autocommit:

Setting Transaction Isolation Levels including DBAPI Autocommit
=================================================================

Most DBAPIs support the concept of configurable transaction :term:`isolation` levels.
These are traditionally the four levels "READ UNCOMMITTED", "READ COMMITTED",
"REPEATABLE READ" and "SERIALIZABLE".  These are usually applied to a
DBAPI connection before it begins a new transaction, noting that most
DBAPIs will begin this transaction implicitly when SQL statements are first
emitted.

DBAPIs that support isolation levels also usually support the concept of true
"autocommit", which means that the DBAPI connection itself will be placed into
a non-transactional autocommit mode.   This usually means that the typical
DBAPI behavior of emitting "BEGIN" to the database automatically no longer
occurs, but it may also include other directives.   When using this mode,
**the DBAPI does not use a transaction under any circumstances**.  SQLAlchemy
methods like ``.begin()``, ``.commit()`` and ``.rollback()`` pass silently
and have no effect.

Instead, each statement invoked upon the connection will commit any changes
automatically; it sometimes also means that the connection itself will use
fewer server-side database resources. For this reason and others, "autocommit"
mode is often desirable for non-tranasctional applications that need to read
individual tables or rows in isolation of a true ACID transaction.

SQLAlchemy dialects can support these isolation levels as well as autocommit to
as great a degree as possible.   The levels are set via family of
"execution_options" parameters and methods that are throughout the Core, such
as the :meth:`_engine.Connection.execution_options` method.   The parameter is
known as :paramref:`_engine.Connection.execution_options.isolation_level` and
the values are strings which are typically a subset of the following names::

    # possible values for Connection.execution_options(isolation_level="<value>")

    "AUTOCOMMIT"
    "READ COMMITTED"
    "READ UNCOMMITTED"
    "REPEATABLE READ"
    "SERIALIZABLE"

Not every DBAPI supports every value; if an unsupported value is used for a
certain backend, an error is raised.

For example, to force REPEATABLE READ on a specific connection::

  with engine.connect().execution_options(isolation_level="REPEATBLE READ") as connection:
      connection.execute(<statement>)

The :paramref:`_engine.Connection.execution_options.isolation_level` option
may also be set engine wide, as is often preferable.   It can be set either
within :func:`_sa.create_engine` directly via the :paramref:`_sa.create_engine.execution_options`
parameter::


    from sqlalchemy import create_engine

    eng = create_engine(
        "postgresql://scott:tiger@localhost/test",
        isolation_level='REPEATABLE READ'
    )

Or for an application that chooses between multiple levels, as may be the case
for the use of "AUTOCOMMIT" to switch between "transactional" and "read-only"
engines, the :meth:`_engine.Engine.execution_options` method will provide a shallow
copy of the :class:`_engine.Engine` that will apply the given isolation
level to all connections::


    from sqlalchemy import create_engine

    eng = create_engine("postgresql://scott:tiger@localhost/test")

    autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")


Above, both ``eng`` and ``autocommit_engine`` share the same dialect
and connection pool.  However the "AUTOCOMMIT" mode will be set upon connections
when they are acquired from the ``autocommit_engine``.

The isolation level setting, regardless of which one it is, is unconditionally
reverted when a connection is returned to the connection pool.


.. note:: The :paramref:`_engine.Connection.execution_options.isolation_level`
   parameter necessarily does not apply to statement level options, such as
   that of :meth:`_sql.Executable.execution_options`.  This because the option
   must be set on a DBAPI connection on a per-transaction basis.

.. seealso::

      :ref:`SQLite Transaction Isolation <sqlite_isolation_level>`

      :ref:`PostgreSQL Transaction Isolation <postgresql_isolation_level>`

      :ref:`MySQL Transaction Isolation <mysql_isolation_level>`

      :ref:`SQL Server Transaction Isolation <mssql_isolation_level>`

      :ref:`session_transaction_isolation` - for the ORM

.. _dbengine_implicit:


Connectionless Execution, Implicit Execution
============================================

.. deprecated:: 2.0 The features of "connectionless" and "implicit" execution
   in SQLAlchemy are deprecated and will be removed in version 2.0.  See
   :ref:`migration_20_implicit_execution` for background.

Recall from the first section we mentioned executing with and without explicit
usage of :class:`_engine.Connection`. "Connectionless" execution
refers to the usage of the ``execute()`` method on an object
which is not a :class:`_engine.Connection`.  This was illustrated using the
:meth:`_engine.Engine.execute` method of :class:`_engine.Engine`::

    result = engine.execute(text("select username from users"))
    for row in result:
        print("username:", row['username'])

In addition to "connectionless" execution, it is also possible
to use the :meth:`~.Executable.execute` method of
any :class:`.Executable` construct, which is a marker for SQL expression objects
that support execution.   The SQL expression object itself references an
:class:`_engine.Engine` or :class:`_engine.Connection` known as the **bind**, which it uses
in order to provide so-called "implicit" execution services.

Given a table as below::

    from sqlalchemy import MetaData, Table, Column, Integer

    meta = MetaData()
    users_table = Table('users', meta,
        Column('id', Integer, primary_key=True),
        Column('name', String(50))
    )

Explicit execution delivers the SQL text or constructed SQL expression to the
:meth:`_engine.Connection.execute` method of :class:`~sqlalchemy.engine.Connection`:

.. sourcecode:: python+sql

    engine = create_engine('sqlite:///file.db')
    with engine.connect() as connection:
        result = connection.execute(users_table.select())
        for row in result:
            # ....

Explicit, connectionless execution delivers the expression to the
:meth:`_engine.Engine.execute` method of :class:`~sqlalchemy.engine.Engine`:

.. sourcecode:: python+sql

    engine = create_engine('sqlite:///file.db')
    result = engine.execute(users_table.select())
    for row in result:
        # ....
    result.close()

Implicit execution is also connectionless, and makes usage of the :meth:`~.Executable.execute` method
on the expression itself.   This method is provided as part of the
:class:`.Executable` class, which refers to a SQL statement that is sufficient
for being invoked against the database.    The method makes usage of
the assumption that either an
:class:`~sqlalchemy.engine.Engine` or
:class:`~sqlalchemy.engine.Connection` has been **bound** to the expression
object.   By "bound" we mean that the special attribute :attr:`_schema.MetaData.bind`
has been used to associate a series of
:class:`_schema.Table` objects and all SQL constructs derived from them with a specific
engine::

    engine = create_engine('sqlite:///file.db')
    meta.bind = engine
    result = users_table.select().execute()
    for row in result:
        # ....
    result.close()

Above, we associate an :class:`_engine.Engine` with a :class:`_schema.MetaData` object using
the special attribute :attr:`_schema.MetaData.bind`.  The :func:`_expression.select` construct produced
from the :class:`_schema.Table` object has a method :meth:`~.Executable.execute`, which will
search for an :class:`_engine.Engine` that's "bound" to the :class:`_schema.Table`.

Overall, the usage of "bound metadata" has three general effects:

* SQL statement objects gain an :meth:`.Executable.execute` method which automatically
  locates a "bind" with which to execute themselves.
* The ORM :class:`.Session` object supports using "bound metadata" in order
  to establish which :class:`_engine.Engine` should be used to invoke SQL statements
  on behalf of a particular mapped class, though the :class:`.Session`
  also features its own explicit system of establishing complex :class:`_engine.Engine`/
  mapped class configurations.
* The :meth:`_schema.MetaData.create_all`, :meth:`_schema.MetaData.drop_all`, :meth:`_schema.Table.create`,
  :meth:`_schema.Table.drop`, and "autoload" features all make usage of the bound
  :class:`_engine.Engine` automatically without the need to pass it explicitly.

.. note::

    The concepts of "bound metadata" and "implicit execution" are not emphasized in modern SQLAlchemy.
    While they offer some convenience, they are no longer required by any API and
    are never necessary.

    In applications where multiple :class:`_engine.Engine` objects are present, each one logically associated
    with a certain set of tables (i.e. *vertical sharding*), the "bound metadata" technique can be used
    so that individual :class:`_schema.Table` can refer to the appropriate :class:`_engine.Engine` automatically;
    in particular this is supported within the ORM via the :class:`.Session` object
    as a means to associate :class:`_schema.Table` objects with an appropriate :class:`_engine.Engine`,
    as an alternative to using the bind arguments accepted directly by the :class:`.Session`.

    However, the "implicit execution" technique is not at all appropriate for use with the
    ORM, as it bypasses the transactional context maintained by the :class:`.Session`.

    Overall, in the *vast majority* of cases, "bound metadata" and "implicit execution"
    are **not useful**.   While "bound metadata" has a marginal level of usefulness with regards to
    ORM configuration, "implicit execution" is a very old usage pattern that in most
    cases is more confusing than it is helpful, and its usage is discouraged.
    Both patterns seem to encourage the overuse of expedient "short cuts" in application design
    which lead to problems later on.

    Modern SQLAlchemy usage, especially the ORM, places a heavy stress on working within the context
    of a transaction at all times; the "implicit execution" concept makes the job of
    associating statement execution with a particular transaction much more difficult.
    The :meth:`.Executable.execute` method on a particular SQL statement
    usually implies that the execution is not part of any particular transaction, which is
    usually not the desired effect.

In both "connectionless" examples, the
:class:`~sqlalchemy.engine.Connection` is created behind the scenes; the
:class:`~sqlalchemy.engine.CursorResult` returned by the ``execute()``
call references the :class:`~sqlalchemy.engine.Connection` used to issue
the SQL statement. When the :class:`_engine.CursorResult` is closed, the underlying
:class:`_engine.Connection` is closed for us, resulting in the
DBAPI connection being returned to the pool with transactional resources removed.

.. _schema_translating:

Translation of Schema Names
===========================

To support multi-tenancy applications that distribute common sets of tables
into multiple schemas, the
:paramref:`.Connection.execution_options.schema_translate_map`
execution option may be used to repurpose a set of :class:`_schema.Table` objects
to render under different schema names without any changes.

Given a table::

    user_table = Table(
        'user', metadata,
        Column('id', Integer, primary_key=True),
        Column('name', String(50))
    )

The "schema" of this :class:`_schema.Table` as defined by the
:paramref:`_schema.Table.schema` attribute is ``None``.  The
:paramref:`.Connection.execution_options.schema_translate_map` can specify
that all :class:`_schema.Table` objects with a schema of ``None`` would instead
render the schema as ``user_schema_one``::

    connection = engine.connect().execution_options(
        schema_translate_map={None: "user_schema_one"})

    result = connection.execute(user_table.select())

The above code will invoke SQL on the database of the form::

    SELECT user_schema_one.user.id, user_schema_one.user.name FROM
    user_schema_one.user

That is, the schema name is substituted with our translated name.  The
map can specify any number of target->destination schemas::

    connection = engine.connect().execution_options(
        schema_translate_map={
            None: "user_schema_one",     # no schema name -> "user_schema_one"
            "special": "special_schema", # schema="special" becomes "special_schema"
            "public": None               # Table objects with schema="public" will render with no schema
        })

The :paramref:`.Connection.execution_options.schema_translate_map` parameter
affects all DDL and SQL constructs generated from the SQL expression language,
as derived from the :class:`_schema.Table` or :class:`.Sequence` objects.
It does **not** impact literal string SQL used via the :func:`_expression.text`
construct nor via plain strings passed to :meth:`_engine.Connection.execute`.

The feature takes effect **only** in those cases where the name of the
schema is derived directly from that of a :class:`_schema.Table` or :class:`.Sequence`;
it does not impact methods where a string schema name is passed directly.
By this pattern, it takes effect within the "can create" / "can drop" checks
performed by methods such as :meth:`_schema.MetaData.create_all` or
:meth:`_schema.MetaData.drop_all` are called, and it takes effect when
using table reflection given a :class:`_schema.Table` object.  However it does
**not** affect the operations present on the :class:`_reflection.Inspector` object,
as the schema name is passed to these methods explicitly.

.. versionadded:: 1.1

.. _sql_caching:


SQL Compilation Caching
=======================

.. versionadded:: 1.4  SQLAlchemy now has a transparent query caching system
   that substantially lowers the Python computational overhead involved in
   converting SQL statement constructs into SQL strings across both
   Core and ORM.   See the introduction at :ref:`change_4639`.

SQLAlchemy includes a comprehensive caching system for the SQL compiler as well
as its ORM variants.   This caching system is transparent within the
:class:`.Engine` and provides that the SQL compilation process for a given Core
or ORM SQL statement, as well as related computations which assemble
result-fetching mechanics for that statement, will only occur once for that
statement object and all others with the identical
structure, for the duration that the particular structure remains within the
engine's "compiled cache". By "statement objects that have the identical
structure", this generally corresponds to a SQL statement that is
constructed within a function and is built each time that function runs::

    def run_my_statement(connection, parameter):
        stmt = select(table)
        stmt = stmt.where(table.c.col == parameter)
        stmt = stmt.order_by(table.c.id)
        return connection.execute(stmt)

The above statement will generate SQL resembling
``SELECT id, col FROM table WHERE col = :col ORDER BY id``, noting that
while the value of ``parameter`` is a plain Python object such as a string
or an integer, the string SQL form of the statement does not include this
value as it uses bound parameters.  Subsequent invocations of the above
``run_my_statement()`` function will use a cached compilation construct
within the scope of the ``connection.execute()`` call for enhanced performance.

.. note:: it is important to note that the SQL compilation cache is caching
   the **SQL string that is passed to the database only**, and **not the data**
   returned by a query.   It is in no way a data cache and does not
   impact the results returned for a particular SQL statement nor does it
   imply any memory use linked to fetching of result rows.

While SQLAlchemy has had a rudimentary statement cache since the early 1.x
series, and additionally has featured the "Baked Query" extension for the ORM,
both of these systems required a high degree of special API use in order for
the cache to be effective.  The new cache as of 1.4 is instead completely
automatic and requires no change in programming style to be effective.

The cache is automatically used without any configurational changes and no
special steps are needed in order to enable it. The following sections
detail the configuration and advanced usage patterns for the cache.


Configuration
-------------

The cache itself is a dictionary-like object called an ``LRUCache``, which is
an internal SQLAlchemy dictionary subclass that tracks the usage of particular
keys and features a periodic "pruning" step which removes the least recently
used items when the size of the cache reaches a certain threshold.  The size
of this cache defaults to 500 and may be configured using the
:paramref:`_sa.create_engine.query_cache_size` parameter::

    engine = create_engine("postgresql://scott:tiger@localhost/test", query_cache_size=1200)

The size of the cache can grow to be a factor of 150% of the size given, before
it's pruned back down to the target size.  A cache of size 1200 above can therefore
grow to be 1800 elements in size at which point it will be pruned to 1200.

The sizing of the cache is based on a single entry per unique SQL statement rendered,
per engine.   SQL statements generated from both the Core and the ORM are
treated equally.  DDL statements will usually not be cached.  In order to determine
what the cache is doing, engine logging will include details about the
cache's behavior, described in the next section.


Estimating Cache Performance Using Logging
------------------------------------------

The above cache size of 1200 is actually fairly large.   For small applications,
a size of 100 is likely sufficient.  To estimate the optimal size of the cache,
assuming enough memory is present on the target host, the size of the cache
should be based on the number of unique SQL strings that may be rendered for the
target engine in use.    The most expedient way to see this is to use
SQL echoing, which is most directly enabled by using the
:paramref:`_sa.create_engine.echo` flag, or by using Python logging; see the
section :ref:`dbengine_logging` for background on logging configuration.

As an example, we will examine the logging produced by the following program::

  from sqlalchemy import Column
  from sqlalchemy import create_engine
  from sqlalchemy import ForeignKey
  from sqlalchemy import Integer
  from sqlalchemy import String
  from sqlalchemy.ext.declarative import declarative_base
  from sqlalchemy.orm import relationship
  from sqlalchemy.orm import Session

  Base = declarative_base()


  class A(Base):
      __tablename__ = "a"

      id = Column(Integer, primary_key=True)
      data = Column(String)
      bs = relationship("B")


  class B(Base):
      __tablename__ = "b"
      id = Column(Integer, primary_key=True)
      a_id = Column(ForeignKey("a.id"))
      data = Column(String)


  e = create_engine("sqlite://", echo=True)
  Base.metadata.create_all(e)

  s = Session(e)

  s.add_all(
      [A(bs=[B(), B(), B()]), A(bs=[B(), B(), B()]), A(bs=[B(), B(), B()])]
  )
  s.commit()

  for a_rec in s.query(A):
      print(a_rec.bs)

When run, each SQL statement that's logged will include a bracketed
cache statistics badge to the left of the parameters passed.   The four
types of message we may see are summarized as follows:

* ``[raw sql]`` - the driver or the end-user emitted raw SQL using
  :meth:`.Connection.exec_driver_sql` - caching does not apply

* ``[no key]`` - the statement object is a DDL statement that is not cached, or
  the statement object contains uncacheable elements such as user-defined
  constructs or arbitrarily large VALUES clauses.

* ``[generated in Xs]`` - the statement was a **cache miss** and had to be
  compiled, then stored in the cache.  it took X seconds to produce the
  compiled construct.  The number X will be in the small fractional seconds.

* ``[cached since Xs ago]`` - the statement was a **cache hit** and did not
  have to be recompiled.  The statement has been stored in the cache since
  X seconds ago.  The number X will be proportional to how long the application
  has been running and how long the statement has been cached, so for example
  would be 86400 for a 24 hour period.

Each badge is described in more detail below.

The first statements we see for the above program will be the SQLite dialect
checking for the existence of the "a" and "b" tables::

  INFO sqlalchemy.engine.Engine PRAGMA temp.table_info("a")
  INFO sqlalchemy.engine.Engine [raw sql] ()
  INFO sqlalchemy.engine.Engine PRAGMA main.table_info("b")
  INFO sqlalchemy.engine.Engine [raw sql] ()

For the above two SQLite PRAGMA statements, the badge reads ``[raw sql]``,
which indicates the driver is sending a Python string directly to the
database using :meth:`.Connection.exec_driver_sql`.  Caching does not apply
to such statements because they already exist in string form, and there
is nothing known about what kinds of result rows will be returned since
SQLAlchemy does not parse SQL strings ahead of time.

The next statements we see are the CREATE TABLE statements::

  INFO sqlalchemy.engine.Engine
  CREATE TABLE a (
    id INTEGER NOT NULL,
    data VARCHAR,
    PRIMARY KEY (id)
  )

  INFO sqlalchemy.engine.Engine [no key 0.00007s] ()
  INFO sqlalchemy.engine.Engine
  CREATE TABLE b (
    id INTEGER NOT NULL,
    a_id INTEGER,
    data VARCHAR,
    PRIMARY KEY (id),
    FOREIGN KEY(a_id) REFERENCES a (id)
  )

  INFO sqlalchemy.engine.Engine [no key 0.00006s] ()

For each of these statements, the badge reads ``[no key 0.00006s]``.  This
indicates that these two particular statements, caching did not occur because
the DDL-oriented :class:`_schema.CreateTable` construct did not produce a
cache key.  DDL constructs generally do not participate in caching because
they are not typically subject to being repeated a second time and DDL
is also a database configurational step where performance is not as critical.

The ``[no key]`` badge is important for one other reason, as it can be produced
for SQL statements that are cacheable except for some particular sub-construct
that is not currently cacheable.   Examples of this include custom user-defined
SQL elements that don't define caching parameters, as well as some constructs
that generate arbitrarily long and non-reproducible SQL strings, the main
examples being the :class:`.Values` construct as well as when using "multivalued
inserts" with the :meth:`.Insert.values` method.

So far our cache is still empty.  The next statements will be cached however,
a segment looks like::


  INFO sqlalchemy.engine.Engine INSERT INTO a (data) VALUES (?)
  INFO sqlalchemy.engine.Engine [generated in 0.00011s] (None,)
  INFO sqlalchemy.engine.Engine INSERT INTO a (data) VALUES (?)
  INFO sqlalchemy.engine.Engine [cached since 0.0003533s ago] (None,)
  INFO sqlalchemy.engine.Engine INSERT INTO a (data) VALUES (?)
  INFO sqlalchemy.engine.Engine [cached since 0.0005326s ago] (None,)
  INFO sqlalchemy.engine.Engine INSERT INTO b (a_id, data) VALUES (?, ?)
  INFO sqlalchemy.engine.Engine [generated in 0.00010s] (1, None)
  INFO sqlalchemy.engine.Engine INSERT INTO b (a_id, data) VALUES (?, ?)
  INFO sqlalchemy.engine.Engine [cached since 0.0003232s ago] (1, None)
  INFO sqlalchemy.engine.Engine INSERT INTO b (a_id, data) VALUES (?, ?)
  INFO sqlalchemy.engine.Engine [cached since 0.0004887s ago] (1, None)

Above, we see essentially two unique SQL strings; ``"INSERT INTO a (data) VALUES (?)"``
and ``"INSERT INTO b (a_id, data) VALUES (?, ?)"``.  Since SQLAlchemy uses
bound parameters for all literal values, even though these statements are
repeated many times for different objects, because the parameters are separate,
the actual SQL string stays the same.

.. note:: the above two statements are generated by the ORM unit of work
   process, and in fact will be caching these in a separate cache that is
   local to each mapper.  However the mechanics and terminology are the same.
   The section :ref:`engine_compiled_cache` below will describe how user-facing
   code can also use an alternate caching container on a per-statement basis.

The caching badge we see for the first occurrence of each of these two
statements is ``[generated in 0.00011s]``. This indicates that the statement
was **not in the cache, was compiled into a String in .00011s and was then
cached**.   When we see the ``[generated]`` badge, we know that this means
there was a **cache miss**.  This is to be expected for the first occurrence of
a particular statement.  However, if lots of new ``[generated]`` badges are
observed for a long-running application that is generally using the same series
of SQL statements over and over, this may be a sign that the
:paramref:`_sa.create_engine.query_cache_size` parameter is too small.  When a
statement that was cached is then evicted from the cache due to the LRU
cache pruning lesser used items, it will display the ``[generated]`` badge
when it is next used.

The caching badge that we then see for the subsequent occurrences of each of
these two statements looks like ``[cached since 0.0003533s ago]``.  This
indicates that the statement **was found in the cache, and was originally
placed into the cache .0003533 seconds ago**.   It is important to note that
while the ``[generated]`` and ``[cached since]`` badges refer to a number of
seconds, they mean different things; in the case of ``[generated]``, the number
is a rough timing of how long it took to compile the statement, and will be an
extremely small amount of time.   In the case of ``[cached since]``, this is
the total time that a statement has been present in the cache.  For an
application that's been running for six hours, this number may read ``[cached
since 21600 seconds ago]``, and that's a good thing.    Seeing high numbers for
"cached since" is an indication that these statements have not been subject to
cache misses for a long time.  Statements that frequently have a low number of
"cached since" even if the application has been running a long time may
indicate these statements are too frequently subject to cache misses, and that
the
:paramref:`_sa.create_engine.query_cache_size` may need to be increased.

Our example program then performs some SELECTs where we can see the same
pattern of "generated" then "cached", for the SELECT of the "a" table as well
as for subsequent lazy loads of the "b" table::

  INFO sqlalchemy.engine.Engine SELECT a.id AS a_id, a.data AS a_data
  FROM a
  INFO sqlalchemy.engine.Engine [generated in 0.00009s] ()
  INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
  FROM b
  WHERE ? = b.a_id
  INFO sqlalchemy.engine.Engine [generated in 0.00010s] (1,)
  INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
  FROM b
  WHERE ? = b.a_id
  INFO sqlalchemy.engine.Engine [cached since 0.0005922s ago] (2,)
  INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
  FROM b
  WHERE ? = b.a_id

From our above program, a full run shows a total of four distinct SQL strings
being cached.   Which indicates a cache size of **four** would be sufficient.   This is
obviously an extremely small size, and the default size of 500 is fine to be left
at its default.

How much memory does the cache use?
-----------------------------------

The previous section detailed some techniques to check if the
:paramref:`_sa.create_engine.query_cache_size` needs to be bigger.   How do we know
if the cache is not too large?   The reason we may want to set
:paramref:`_sa.create_engine.query_cache_size` to not be higher than a certain
number would be because we have an application that may make use of a very large
number of different statements, such as an application that is building queries
on the fly from a search UX, and we don't want our host to run out of memory
if for example, a hundred thousand different queries were run in the past 24 hours
and they were all cached.

It is extremely difficult to measure how much memory is occupied by Python
data structures, however using a process to measure growth in memory via ``top`` as a
successive series of 250 new statements are added to the cache suggest a
moderate Core statement takes up about 12K while a small ORM statement takes about
20K, including result-fetching structures which for the ORM will be much greater.


.. _engine_compiled_cache:

Disabling or using an alternate dictionary to cache some (or all) statements
-----------------------------------------------------------------------------

The internal cache used is known as ``LRUCache``, but this is mostly just
a dictionary.  Any dictionary may be used as a cache for any series of
statements by using the :paramref:`.Connection.execution_options.compiled_cache`
option as an execution option.  Execution options may be set on a statement,
on an :class:`_engine.Engine` or :class:`_engine.Connection`, as well as
when using the ORM :meth:`_orm.Session.execute` method for SQLAlchemy-2.0
style invocations.   For example, to run a series of SQL statements and have
them cached in a particular dictionary::

  my_cache = {}
  with engine.connect().execution_options(compiled_cache=my_cache) as conn:
      conn.execute(table.select())

The SQLAlchemy ORM uses the above technique to hold onto per-mapper caches
within the unit of work "flush" process that are separate from the default
cache configured on the :class:`_engine.Engine`, as well as for some
relationship loader queries.

The cache can also be disabled with this argument by sending a value of
``None``::

  # disable caching for this connection
  with engine.connect().execution_options(compiled_cache=None) as conn:
      conn.execute(table.select())

.. _engine_lambda_caching:

Using Lambdas to add significant speed gains to statement production
--------------------------------------------------------------------

.. deepalchemy:: This technique is generally non-essential except in very performance
   intensive scenarios, and intended for experienced Python programmers.
   While fairly straightforward, it involves metaprogramming concepts that are
   not appropriate for novice Python developers.  The lambda approach can be
   applied to at a later time to existing code with a minimal amount of effort.

The caching system has in its roots the SQLAlchemy :ref:`"baked query"
<baked_toplevel>` extension, which made novel use of Python lambdas in order to
produce SQL statements that were intrinsically cacheable, while at the same
time decreasing not just the overhead involved to compile the statement into
SQL, but also the overhead in constructing the statement object from a Python
perspective. The new caching in SQLAlchemy by default does not substantially
optimize the construction of SQL constructs.  This refers to the Python
overhead taken up to construct the statement object itself before it is
compiled or executed, such as the :class:`_sql.Select` object used in the
example below::

    def run_my_statement(connection, parameter):
        stmt = select(table)
        stmt = stmt.where(table.c.col == parameter)
        stmt = stmt.order_by(table.c.id)

        return connection.execute(stmt)

Above, in order to construct ``stmt``, we see three Python functions or methods
``select()``, ``.where()`` and ``.order_by()`` being invoked directly, and
additionally there is a Python method invoked when we construct ``table.c.col
== 'foo'``, as the expression language overrides the ``__eq__()`` method to
produce a SQL construct.   Within each of these calls is a series of argument
checking and internal construction logic that makes use of many more Python
function calls.   With intense production of thousands of statement objects,
these function calls can add up.   Using the recipe for profiling at
:ref:`faq_code_profiling`, the above Python code within the scope of the
``select()`` call down to the ``.order_by()`` call uses 73 Python function
calls to produce.

Additionally, statement caching requires that a cache key be generated against
the above statement, which must be composed of all elements within the
statement that uniquely identify the SQL that it would produce.  Measuring
this process for the above statement takes another 40 Python function calls.

In order to ensure the full performance gains of the prior "baked query"
extension are still available, the "lambda:" system used by baked queries has
been adapted into a more capable and easier to use system as an intrinsic part
of the SQLAlchemy Core expression language (which by extension then includes
ORM queries, which as of SQLAlchemy 1.4 using 2.0-style APIs may also be
invoked directly from SQLAlchemy Core expression objects).   We can
adapt our statement above to be built using "lambdas" by making use of the
:func:`_sql.lambda_stmt` element.  Using this approach, we indicate that the
:func:`_sql.select` should be returned by a lambda.  We can then add new
criteria to the statement by composing further lambdas onto the object in a
similar manner as how "baked queries" worked::

  from sqlalchemy import lambda_stmt

  def run_my_statement(connection, parameter):
      stmt = lambda_stmt(lambda: select(table))
      stmt += lambda s: s.where(table.c.col == parameter)
      stmt += lambda s: s.order_by(table.c.id)

      return connection.execute(stmt)

  result = run_my_statement(some_connection, "some parameter")

The above code produces a :class:`.StatementLambdaElement`, which behaves like a
Core SQL construct but defers the construction of the statement in most
cases until it is needed by the compiler.  If the statement is already cached,
the lambdas will not be called.

The cache key is based on the **Python source code location of each lambda
itself**, which in the Python interpreter is essentially the ``__code__``
element of the Python function. This means that the lambda approach should only
be used inside of a function where the lambdas themselves will be the **same
lambdas each time, from a Python source code perspective**.

The execution process for the above lambda will **extract literal parameters**
from the statement each time, without needing to actually run the lambdas. In
the above example, each time the variable ``parameter`` is used within the
lambda to generate the WHERE clause of the statement, while the actual lambda
present will not actually be run, the value of ``parameter`` will be tracked
and the current value of the variable will be used within the statement
parameters at execution time.  This is a feature that was not possible with the
"baked query" extension and involves the use of up-front analysis of the
incoming ``__code__`` object to determine how parameters can be extracted from
future lambdas against that same code object.

More simply, this means it's safe for the lambda statement
to use arbitrary literal parameters, which don't modify the structure
of the statement, on each invocation::

  def run_my_statement(connection, parameter):
      stmt = lambda_stmt(lambda: select(table))
      stmt += lambda s: s.where(table.c.col == parameter)
      stmt += lambda s: s.order_by(table.c.id)

      return connection.execute(stmt)

However, it's not safe for an individual lambda so modify the SQL structure
of the statement across calls::

  # incorrect example
  def run_my_statement(connection, parameter, add_criteria=False):
      stmt = lambda_stmt(lambda: select(table))

      # will not be cached correctly as add_criteria changes
      stmt += lambda s: s.where(
          and_(add_criteria, table.c.col == parameter)
          if add_criteria
          else s.where(table.c.col == parameter)
      )

      stmt += lambda s: s.order_by(table.c.id)

      return connection.execute(stmt)

The lambda statements indicated above will invoke all of the lambdas the first
time they are constructed; subsequent to that, the lambdas will not be invoked.
On these subsequent runs, a lambda construct will use far fewer Python function
calls in order to construct the un-cached object as well as to generate the
cache key after the first call.  The above statement using lambdas takes only
41 Python function calls to generate the whole structure as well as to produce
the cache key, including the extraction of the bound parameters. This is
compared to a total of about 115 Python function calls for the non-lambda
version.

For a series of examples of "lambda" caching with performance comparisons,
see the "short_selects" test suite within the :ref:`examples_performance`
performance example.

.. _engine_disposal:

Engine Disposal
===============

The :class:`_engine.Engine` refers to a connection pool, which means under normal
circumstances, there are open database connections present while the
:class:`_engine.Engine` object is still resident in memory.   When an :class:`_engine.Engine`
is garbage collected, its connection pool is no longer referred to by
that :class:`_engine.Engine`, and assuming none of its connections are still checked
out, the pool and its connections will also be garbage collected, which has the
effect of closing out the actual database connections as well.   But otherwise,
the :class:`_engine.Engine` will hold onto open database connections assuming
it uses the normally default pool implementation of :class:`.QueuePool`.

The :class:`_engine.Engine` is intended to normally be a permanent
fixture established up-front and maintained throughout the lifespan of an
application.  It is **not** intended to be created and disposed on a
per-connection basis; it is instead a registry that maintains both a pool
of connections as well as configurational information about the database
and DBAPI in use, as well as some degree of internal caching of per-database
resources.

However, there are many cases where it is desirable that all connection resources
referred to by the :class:`_engine.Engine` be completely closed out.  It's
generally not a good idea to rely on Python garbage collection for this
to occur for these cases; instead, the :class:`_engine.Engine` can be explicitly disposed using
the :meth:`_engine.Engine.dispose` method.   This disposes of the engine's
underlying connection pool and replaces it with a new one that's empty.
Provided that the :class:`_engine.Engine`
is discarded at this point and no longer used, all **checked-in** connections
which it refers to will also be fully closed.

Valid use cases for calling :meth:`_engine.Engine.dispose` include:

* When a program wants to release any remaining checked-in connections
  held by the connection pool and expects to no longer be connected
  to that database at all for any future operations.

* When a program uses multiprocessing or ``fork()``, and an
  :class:`_engine.Engine` object is copied to the child process,
  :meth:`_engine.Engine.dispose` should be called so that the engine creates
  brand new database connections local to that fork.   Database connections
  generally do **not** travel across process boundaries.

* Within test suites or multitenancy scenarios where many
  ad-hoc, short-lived :class:`_engine.Engine` objects may be created and disposed.


Connections that are **checked out** are **not** discarded when the
engine is disposed or garbage collected, as these connections are still
strongly referenced elsewhere by the application.
However, after :meth:`_engine.Engine.dispose` is called, those
connections are no longer associated with that :class:`_engine.Engine`; when they
are closed, they will be returned to their now-orphaned connection pool
which will ultimately be garbage collected, once all connections which refer
to it are also no longer referenced anywhere.
Since this process is not easy to control, it is strongly recommended that
:meth:`_engine.Engine.dispose` is called only after all checked out connections
are checked in or otherwise de-associated from their pool.

An alternative for applications that are negatively impacted by the
:class:`_engine.Engine` object's use of connection pooling is to disable pooling
entirely.  This typically incurs only a modest performance impact upon the
use of new connections, and means that when a connection is checked in,
it is entirely closed out and is not held in memory.  See :ref:`pool_switching`
for guidelines on how to disable pooling.

.. _dbapi_connections:

Working with Driver SQL and Raw DBAPI Connections
=================================================

The introduction on using :meth:`_engine.Connection.execute` made use of the
:func:`_expression.text` construct in order to illustrate how textual SQL statements
may be invoked.  When working with SQLAlchemy, textual SQL is actually more
of the exception rather than the norm, as the Core expression language
and the ORM both abstract away the textual representation of SQL.  However, the
:func:`_expression.text` construct itself also provides some abstraction of textual
SQL in that it normalizes how bound parameters are passed, as well as that
it supports datatyping behavior for parameters and result set rows.

Invoking SQL strings directly to the driver
--------------------------------------------

For the use case where one wants to invoke textual SQL directly passed to the
underlying driver (known as the :term:`DBAPI`) without any intervention
from the :func:`_expression.text` construct, the :meth:`_engine.Connection.exec_driver_sql`
method may be used::

    with engine.connect() as conn:
        conn.exec_driver_sql("SET param='bar'")


.. versionadded:: 1.4  Added the :meth:`_engine.Connection.exec_driver_sql` method.

Working with the DBAPI cursor directly
--------------------------------------

There are some cases where SQLAlchemy does not provide a genericized way
at accessing some :term:`DBAPI` functions, such as calling stored procedures as well
as dealing with multiple result sets.  In these cases, it's just as expedient
to deal with the raw DBAPI connection directly.

The most common way to access the raw DBAPI connection is to get it
from an already present :class:`_engine.Connection` object directly.  It is
present using the :attr:`_engine.Connection.connection` attribute::

    connection = engine.connect()
    dbapi_conn = connection.connection

The DBAPI connection here is actually a "proxied" in terms of the
originating connection pool, however this is an implementation detail
that in most cases can be ignored.    As this DBAPI connection is still
contained within the scope of an owning :class:`_engine.Connection` object, it is
best to make use of the :class:`_engine.Connection` object for most features such
as transaction control as well as calling the :meth:`_engine.Connection.close`
method; if these operations are performed on the DBAPI connection directly,
the owning :class:`_engine.Connection` will not be aware of these changes in state.

To overcome the limitations imposed by the DBAPI connection that is
maintained by an owning :class:`_engine.Connection`, a DBAPI connection is also
available without the need to procure a
:class:`_engine.Connection` first, using the :meth:`_engine.Engine.raw_connection` method
of :class:`_engine.Engine`::

    dbapi_conn = engine.raw_connection()

This DBAPI connection is again a "proxied" form as was the case before.
The purpose of this proxying is now apparent, as when we call the ``.close()``
method of this connection, the DBAPI connection is typically not actually
closed, but instead :term:`released` back to the
engine's connection pool::

    dbapi_conn.close()

While SQLAlchemy may in the future add built-in patterns for more DBAPI
use cases, there are diminishing returns as these cases tend to be rarely
needed and they also vary highly dependent on the type of DBAPI in use,
so in any case the direct DBAPI calling pattern is always there for those
cases where it is needed.

Some recipes for DBAPI connection use follow.

.. _stored_procedures:

Calling Stored Procedures
-------------------------

For stored procedures with special syntactical or parameter concerns,
DBAPI-level `callproc <http://legacy.python.org/dev/peps/pep-0249/#callproc>`_
may be used::

    connection = engine.raw_connection()
    try:
        cursor = connection.cursor()
        cursor.callproc("my_procedure", ['x', 'y', 'z'])
        results = list(cursor.fetchall())
        cursor.close()
        connection.commit()
    finally:
        connection.close()

Multiple Result Sets
--------------------

Multiple result set support is available from a raw DBAPI cursor using the
`nextset <http://legacy.python.org/dev/peps/pep-0249/#nextset>`_ method::

    connection = engine.raw_connection()
    try:
        cursor = connection.cursor()
        cursor.execute("select * from table1; select * from table2")
        results_one = cursor.fetchall()
        cursor.nextset()
        results_two = cursor.fetchall()
        cursor.close()
    finally:
        connection.close()



Registering New Dialects
========================

The :func:`_sa.create_engine` function call locates the given dialect
using setuptools entrypoints.   These entry points can be established
for third party dialects within the setup.py script.  For example,
to create a new dialect "foodialect://", the steps are as follows:

1. Create a package called ``foodialect``.
2. The package should have a module containing the dialect class,
   which is typically a subclass of :class:`sqlalchemy.engine.default.DefaultDialect`.
   In this example let's say it's called ``FooDialect`` and its module is accessed
   via ``foodialect.dialect``.
3. The entry point can be established in setup.py as follows::

      entry_points="""
      [sqlalchemy.dialects]
      foodialect = foodialect.dialect:FooDialect
      """

If the dialect is providing support for a particular DBAPI on top of
an existing SQLAlchemy-supported database, the name can be given
including a database-qualification.  For example, if ``FooDialect``
were in fact a MySQL dialect, the entry point could be established like this::

      entry_points="""
      [sqlalchemy.dialects]
      mysql.foodialect = foodialect.dialect:FooDialect
      """

The above entrypoint would then be accessed as ``create_engine("mysql+foodialect://")``.

Registering Dialects In-Process
-------------------------------

SQLAlchemy also allows a dialect to be registered within the current process, bypassing
the need for separate installation.   Use the ``register()`` function as follows::

    from sqlalchemy.dialects import registry
    registry.register("mysql.foodialect", "myapp.dialect", "MyMySQLDialect")

The above will respond to ``create_engine("mysql+foodialect://")`` and load the
``MyMySQLDialect`` class from the ``myapp.dialect`` module.


Connection / Engine API
=======================

.. autoclass:: BaseCursorResult
    :members:

.. autoclass:: ChunkedIteratorResult
    :members:

.. autoclass:: Connection
   :members:

.. autoclass:: Connectable
   :members:

.. autoclass:: CreateEnginePlugin
   :members:

.. autoclass:: Engine
   :members:

.. autoclass:: ExceptionContext
   :members:

.. autoclass:: FrozenResult
    :members:

.. autoclass:: IteratorResult
    :members:

.. autoclass:: LegacyRow
    :members:

.. autoclass:: MergedResult
    :members:

.. autoclass:: NestedTransaction
    :members:

.. autoclass:: Result
    :members:
    :inherited-members:
    :exclude-members: memoized_attribute, memoized_instancemethod


.. autoclass:: CursorResult
    :members:
    :inherited-members:
    :exclude-members: memoized_attribute, memoized_instancemethod

.. autoclass:: LegacyCursorResult
    :members:

.. autoclass:: Row
    :members:
    :private-members: _fields, _mapping

.. autoclass:: RowMapping
    :members:

.. autoclass:: Transaction
    :members:

.. autoclass:: TwoPhaseTransaction
    :members:

