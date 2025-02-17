Using the Engine
^^^^^^^^^^^^^^^^

The :class:`~bloop.engine.Engine` is the main way you'll interact with DynamoDB (and DynamoDBStreams).
Once you've :ref:`defined some models <define-models>`, you're ready to start
:func:`loading <bloop.engine.Engine.load>`, :func:`saving <bloop.engine.Engine.save>` and
:func:`querying <bloop.engine.Engine.query>`.

.. attention::

    This section uses the same ``User`` model from the previous section.  If you've haven't already done so,
    :ref:`go back <define-models>` and set that up.


.. _user-engine-config:

===============
 Configuration
===============

Engines expose a small number of configuration options.  On ``__init__``, there are three optional kwargs:

* ``dynamodb``, a DynamoDB client defaulting to ``boto3.client("dynamodb")``
* ``dynamodbstreams``, a DynamoDBStreams client defaulting to ``boto3.client("dynamodbstreams")``
* ``table_name_template``, a format string containing "{table_name}" or a function that takes a model and returns a
  table name for the engine.

You will rarely need to modify the first two, except when you are constructing multiple engines (eg. cross-region
replication) or connecting to DynamoDBLocal.  For examples of both, see :ref:`Bloop Patterns <patterns-local>`.

Most of the time, you will use ``table_name_template`` to inject configuration into your model/table bindings.  For
example, the following will prefix every table name with ``"dev-"`` for local development:

.. code-block:: python

    engine = Engine(table_name_template="dev-{table_name}")

Meanwhile, the following function will suffix the table name with a random int:

.. code-block:: python

    def with_nonce(model):
        return f"{model.Meta.table_name}-{random.randint(0, 10)}"

    engine = Engine(table_name_template=with_nonce)


======
 Bind
======

As noted in the previous section, every model must first be bound to a backing table with
:func:`Engine.bind <bloop.engine.Engine.bind>` before we can interact with instances in DynamoDB.

.. note::

    Starting with 1.1.0, the ``skip_table_setup`` parameter is available to bypass the create/verify calls
    to DynamoDB.  This is not recommended except in situations where models are bound frequently, ie. a high-volume
    Lambda function.  See `Issue #83`_.

.. _Issue #83: https://github.com/numberoverzero/bloop/issues/83

When an engine binds a model, it also binds all non-abstract subclasses.  This means you can bind all models in one
call, centralizing any error handling or table correction.  For example, you may have specialized models for users,
notifications, and impressions.  Each of these can be grouped with an abstract base, and then all specialized models
created at once:

.. code-block:: python

    class BaseUser(BaseModel):
        class Meta:
            abstract = True

    class BaseNotification(BaseModel):
        class Meta:
            abstract = True

    ...

    class Admin(BaseUser):
        ...

    class Moderator(BaseUser):
        ...

    class PriorityNotification(BaseNotification):
        ...

    class EmailNotification(BaseNotification):
        ...


    try:
        engine.bind(BaseUser)
    except TableMismatch:
        print("Failed to bind all user models")

    try:
        engine.bind(BaseNotification)
    except TableMismatch:
        print("Failed to bind all notification models")

Now you can import a single base (:class:`~bloop.models.BaseModel` or a subclass) from your ``models.py`` module
and automatically bind any dynamic models created from that base.

.. _user-engine-save:

======
 Save
======

:func:`Save <bloop.engine.Engine.save>` is performed with `UpdateItem`_ since absolute overwrites (such as PutItem)
are rarely desired in a distributed, optimistic concurrency system.  This is the central decision that enables a
table to back multiple models.  A partial save allows a model to update an item in the table without accidentally
clearing the columns that model doesn't know about.

Saving an item or items is very simple:

.. code-block:: pycon

    >>> from datetime import datetime, timezone
    >>> now = datetime.now(timezone.utc)
    >>> user = User(...)
    >>> engine.save(user)
    >>> tweet = Tweet(...)
    >>> user.last_activity = now
    >>> engine.save(user, tweet)

-----------------
 Save Conditions
-----------------

You can perform optimistic saves with a ``condition``.  If a condition is not met when DynamoDB tries to apply the
update, the update fails and bloop immediately raises :exc:`~bloop.exceptions.ConstraintViolation`.  Conditions are
specified on columns using the standard ``<, >=, ==, ...`` operators, as well as
``begins_with, between, contains, in_``.  Conditions can be chained together and combined with bitwise operators
``&, |, ~``:

.. code-block:: pycon

    >>> user = User(username="numberoverzero")
    >>> username_available = User.username.is_(None)
    >>> engine.save(user, condition=username_available)
    # Success
    >>> engine.save(user, condition=username_available)
    Traceback (most recent call last):
      ...
    ConstraintViolation: The condition was not met.

-------------
 Atomic Save
-------------

.. warning::

    The ``atomic=`` keyword is deprecated in 2.4 and will be removed in 3.0.
    For an equivalent pattern see :ref:`patterns-snapshot` and `Issue #138`_.

    .. _Issue #138: https://github.com/numberoverzero/bloop/issues/138

A common use for conditions is performing atomic updates.  Often you only want to apply some changes if the local
state matches the state in DynamoDB; if it has changed, you may need to reload the object before applying
local changes.  Save provides a shorthand for this, ``atomic=True``.  By default saves are not atomic.


.. note::

    The ``atomic`` keyword is applied **per-object** and not **for all objects being saved at once**.
    For full-blown transactions see the User Guide's :ref:`user-transactions`.

If you create a new User and perform an atomic save, it will fail if there was any previous state for that hash/range
key (since the expected state before the save was non-existent).  If you fetch an object from a query which doesn't
project all columns, only the columns that are projected will be part of the atomic condition (not loading a column
doesn't say whether we should expect it to have a value or not).

.. seealso::

    Atomic conditions can be tricky, and there are subtle edge cases.  See the :ref:`Atomic Conditions
    <user-conditions-atomic>` section of the User Guide for detailed examples of generated atomic conditions.

If you provide a ``condition`` and ``atomic`` is True, the atomic condition will be ANDed with the condition to
form a single ConditionExpression.

.. code-block:: pycon

    >>> is_verified = User.verified.is_(True)
    >>> no_profile = User.profile.is_(None)
    >>> engine.save(
    ...     user,
    ...     condition=(is_verified & no_profile),
    ...     atomic=True)

.. _UpdateItem: http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_UpdateItem.html

---------------
 Return Values
---------------

You can optionally specify ``sync="old"`` or ``sync="new"`` to update the saved objects with the last seen or most
recent values when the save completes.  This saves a read unit and is strongly consistent, and can be useful to eg.
read the last value before you overwrote an attr or fetch attributes you didn't modify:

.. code-block:: pycon

    >>> user = User(username="n0", email="user@n0.dev")
    >>> engine.save(user, sync="new")
    >>> if not user.verified:
    ...     helpers.send_verification_reminder(user.email, since=user.created_on)


In a highly concurrent environment the ``sync="old"`` option is very useful to capture the last value a field held
before overwriting; then you can safely clean up any cascading references.  For example, if you store an
s3 object key that points to the latest revision of some document you might model it as follows:

.. code-block:: pycon

    class Document(BaseModel):
        name = Column(String, hash_key=True)
        location = Column(String)

The following could cause dangling objects if two updates occur at the same time:

.. code-block:: python

    def wrong_update(name, new_location):
        doc = Document(name=name)
        engine.load(doc)
        if doc.location != new_location:
            delete_s3_object(doc.location)
        doc.location = new_location
        engine.save(doc)

Instead, you should read the previous values when you perform the write, and then clean up the location:

.. code-block:: python

    def correct_update(name, new_location):
        doc = Document(name=name, location=new_location)
        engine.save(doc, sync="old")
        if doc.location != new_location:
            delete_s3_object(doc.location)

---------
 Actions
---------

Most changes you make to modeled objects fall into two update categories: ``SET`` and ``REMOVE``.  Any time a value
serializes as ``None`` or you call ``del myobj.some_attr`` it will likely be a remove, while ``myobj.attr = value``
will be a set.  (This is up to the column's type, so you can override this behavior to use your own sentinel values).

.. warning::
    As mentioned in `Issue #136`_ and the `DynamoDb Developer Guide`_, an atomic counter is not appropriate
    unless you can tolerate overcounting or undercounting.  AWS explicitly discourages using ``add`` or ``delete``
    in general.

    .. _Issue #136: https://github.com/numberoverzero/bloop/issues/136
    .. _DynamoDb Developer Guide: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/WorkingWithItems.html#WorkingWithItems.AtomicCounters

Dynamo exposes two additional update types: ``ADD`` and ``DELETE``.  These allow you to specify relative changes
without knowing the current value stored in Dynamo.  One of the most common examples is a website view count: for a
popular website the optimistic concurrency model will cause a lot of write contention and cap your throughput since
each change requires a read, modify, save.  If there's a conflict you'll need to do all three again, for each writer.

Instead of reading the value and using a conditional save, you can instead wrap the offset in a
:func:`bloop.actions.add` and tell bloop to apply the desired change.  Compare the two following:

.. code-block:: python

    # Option 1) conditional write, wrap in retries
    website = Website("google.com")
    engine.load(website)
    website.views += 1
        # raises ConstraintViolation most of the time due to write contention
    engine.save(website, condition=Website.views==(website.views-1))


    # Option 2) add instead of set
    website = Website("google.com")
    website.views = bloop.actions.add(1)
        # no contention
    engine.save(website)

When combined with return values above, we can add 1 and see the new value all in one call:

.. code-block:: python

    website = Website("google.com")
    website.views = bloop.actions.add(1)
    engine.save(website, sync=True)
    print(f"views after save: {website.views}")

Note that :func:`bloop.actions.set` and :func:`bloop.actions.remove` are assumed if you don't set a column
to an explicit action:

.. code-block:: python

    # both equivalent
    website.views = 21
    website.views = bloop.actions.set(21)

    # all equivalent
    website.views = None
    del website.views
    website.views = bloop.actions.remove(None)

Finally, the :func:`bloop.actions.add` action only supports Number and Set data types.
In addition, add can only be used on top-level attributes, not nested attributes.

Meanwhile :func:`bloop.actions.delete` only supports the Set data type.
It can also only be used on top-level attributes.

.. _user-engine-delete:

========
 Delete
========

:func:`Delete <bloop.engine.Engine.delete>` has the same signature as :func:`~bloop.engine.Engine.save`.  Both
operations are mutations on an object that may or may not exist, and simply map to two different APIs (Delete calls
`DeleteItem`_).  You can delete multiple objects at once; specify a ``condition``; use the ``atomic=True``
shorthand to only delete objects unchanged since you last loaded them from DynamoDB; and use ``sync="old"`` to update
local objects with their last values before deletion.

.. code-block:: pycon

    >>> from datetime import datetime, timedelta, timezone
    >>> engine.delete(user, tweet)
    >>> engine.delete(tps_report, atomic=True)
    >>> now = datetime.now(timezone.utc)
    >>> cutoff = now - timedelta(years=2)
    >>> engine.delete(
    ...     account,
    ...     condition=Account.last_login < cutoff)


.. code-block:: pycon

    >>> banned_account = Account(id="user@n0.dev")
    >>> engine.delete(banned_account, sync="old")
    >>> last_email = banned_account.email
    >>> helpers.notify_acct_change(last_email, reason="spamming")

.. _DeleteItem: http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_DeleteItem.html

.. _user-engine-load:

======
 Load
======

Unlike most existing DynamoDB object mappers, Bloop does not create new instances when loading objects.
This improves performance and makes atomic tracking much easier, and allows you to use thick or thin models by
minimizing how many times the constructor is invoked for effectively the same object (same hash/range keys).

Like :func:`~bloop.engine.Engine.save` and :func:`~bloop.engine.Engine.delete` above,
:func:`Engine.load <bloop.engine.Engine.load>` takes a variable number of objects to load from DynamoDB:

.. code-block:: pycon

    >>> user = User(id="some-id")
    >>> tweet = Tweet(user="some-id", id="some-tweet")
    >>> engine.load(user, tweet)

If ``consistent`` is True, then `strongly consistent reads`__ will be used:

.. code-block:: pycon

    >>> objs = user, tweet
    >>> engine.load(*objs, consistent=True)

If any objects aren't loaded, Bloop raises :exc:`~bloop.exceptions.MissingObjects`:

.. code-block:: pycon

    >>> user = User(username="not-real")
    >>> engine.load(user)
    Traceback (most recent call last):
      ...
    MissingObjects: Failed to load some objects.

You can access :attr:`MissingObjects.objects <bloop.exceptions.MissingObjects.objects>` to see which objects failed
to load.

__ http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html

.. _user-query:

=======
 Query
=======

This section defines a new model to demonstrate the various filtering and conditions available:

.. code-block:: python

    class Account(BaseModel):
        name = Column(String, hash_key=True)
        number = Column(Integer, range_key=True)
        created_on = Column(DateTime)
        balance = Column(Number)
        level = Column(Integer)

        by_level = GlobalSecondaryIndex(
            projection="all", hash_key=level)

        by_balance = LocalSecondaryIndex(
            projection=["created_on"], range_key="balance")

    engine = Engine()
    engine.bind(Account)

-----
 All
-----

Bloop's query and scan iterators are lazy, fetching only as many pages as needed to advance when you call ``next()``.
If you want to eagerly load all results, you can use :func:`all() <bloop.search.QueryIterator.all>` to load all
results into a single list.  Note that calling ``all()`` will reset the query, and will return an empty list if there
are no results.

.. code-block:: pycon

    >>> q = engine.query(Account,
    ...     key=Account.name == "numberoverzero")
    >>> q.all()
    [Account(name='numberoverzero', number=21623]
    >>> q.exhausted
    True
    >>> q.all()
    [Account(name='numberoverzero', number=21623]


-------
 First
-------

Often, you'll only need a single result from the query; with the correct sorting and indexes, the first result can
be used to get a maximum or minimum.  Use :func:`first() <bloop.search.QueryIterator.first>` to get the first result,
if it exists.  If there are no results, raises :exc:`~bloop.exceptions.ConstraintViolation`.

.. code-block:: pycon

    >>> q = engine.query(Account,
    ...     key=Account.name == "numberoverzero")
    >>> q.first()
    Account(name='numberoverzero', number=21623)

-----
 One
-----

Similar to :func:`~bloop.search.QueryIterator.first`, you can get the unique result of a query with
:func:`one() <bloop.search.QueryIterator.one>`.  If there are no results, or more than one result, raises
:exc:`~bloop.exceptions.ConstraintViolation`.

.. code-block:: pycon

    >>> q = engine.query(Account,
    ...     key=Account.name == "numberoverzero")
    >>> q.one()
    Traceback (most recent call last):
        ...
    ConstraintViolation: Query found more than one result.

-------
 Count
-------

To get a count of items that match some query use the ``"count"`` projection.

.. code-block:: pycon

    >>> q = engine.query(
    ...         Account.by_email,
    ...         key=Account.email == "foo@bar.com",
    ...         projection="count")
    >>> q.count
    256

Both ``count`` and ``scanned`` are calculated only when the query is executed, so you must call
:func:`QueryIterator.reset` to see changes take effect.

.. code-block:: pycon

    >>> new = Account(...)
    >>> engine.save(new)
    >>> q.count
    256
    >>> q.reset()
    >>> q.count
    257

.. _user-query-key:

----------------
 Key Conditions
----------------

Queries can be performed against a Model or an Index.  You must specify at least a hash key equality condition; a
range key condition is optional.

.. code-block:: pycon

    >>> owned_by_stacy = Account.name == "Stacy"
    >>> q = engine.query(Account, key=owned_by_stacy)
    >>> for account in q:
    ...     print(account)
    ...

Here, the query uses the Index's range_key to narrow the range of accounts to find:

.. code-block:: pycon

    >>> owned_by_stacy = Account.name == "Stacy"
    >>> at_least_one_mil = Account.balance >= 1000000
    >>> q = engine.query(Account.by_balance,
    ...     key=owned_by_stacy & at_least_one_mil)
    >>> for account in q:
    ...     print(account.balance)

.. note::

    A query must always include an equality check ``==`` or ``is_`` against the model or index's hash key.
    If you want to include a condition on the range key, it can be one of ``==, <, <=, >, >=, between, begins_with``.

    See the `KeyConditionExpression`__ parameter of the Query operation in the Developer's Guide.

    __ http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Query.html#DDB-Query-request-KeyConditionExpression

.. _user-query-filter:

-----------
 Filtering
-----------

If you provide a ``filter`` condition, DynamoDB only returns items that match the filter.  Conditions can be on
any column -- except the hash and range key being queried -- projected into the Index.  All non-key columns are
available for queries against a model.  A filter condition can use any condition operations.
Here is the same LSI query as above, but now excluding accounts created in the last 30 days:

.. code-block:: pycon

    >>> from datetime import datetime, timedelta, timezone
    >>> now = datetime.now(timezone.utc)
    >>> recent = now - timedelta(days=30)
    >>> key_condition = owned_by_stacy & at_least_one_mil
    >>> exclude_recent = Account.created_on < recent
    >>> q = engine.query(Account.by_balance,
    ...     key=key_condition,
    ...     filter=exclude_recent)

.. warning::

    Trying to use a column that's not part of an Index's projection will raise
    :exc:`~bloop.exceptions.InvalidFilterCondition`, since the value can't be loaded.  This does not apply to queries
    against an LSI with ``strict=False``, which will consume additional reads to apply the filter.

    .. code-block:: pycon

        >>> q = engine.query(Account.by_balance,
        ...     key=key_condition,
        ...     filter=Account.level == 3)
        Traceback (most recent call last):
          ...
        InvalidFilterCondition: <Column[Account.level]> is not available for the projection.

-------------
 Projections
-------------

By default, queries return all columns projected into the index or model.  You can use the ``projection`` parameter
to control which columns are returned for each object.  This must be "all" to include everything in the index or
model's projection, or a list of columns or column model names to include.

.. code-block:: pycon

    >>> q = engine.query(Account,
    ...     key=key_condition,
    ...     projection=["email", "balance"])
    >>> account = q.first()
    >>> account.email
    'user@domain.com'
    >>> account.balance
    Decimal('3400')
    >>> account.level
    Traceback (most recent call last):
        ...
    AttributeError: ...

Because the projection did not include ``Account.level``, it was not loaded on the account object.

-----------------------
 Configuration Options
-----------------------

The remaining options are ``consistent`` and ``forward``.  When ``consistent`` is True,
`strongly consistent reads`__ are used.  By default, consistent is False.  Use ``forward`` to query ascending
or descending.  By default ``forward`` is True, or ascending.

__ http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html

.. _user-query-state:

----------------
 Iterator State
----------------

The :class:`~bloop.search.QueryIterator` exposes a number of properties to inspect its current progress:

* ``count`` -- the number of items loaded from DynamoDB so far, including buffered items.
* ``exhausted`` -- True if there are no more results
* ``scanned`` -- the number of items DynamoDB evaluated, before applying any filter condition.

To restart a query, use :func:`QueryIterator.reset() <bloop.search.QueryIterator.reset>`:

.. code-block:: pycon

    >>> query = engine.query(...)
    >>> unique = query.one()
    >>> query.exhausted
    True
    >>> query.reset()
    >>> query.exhausted
    False
    >>> same = query.one()
    >>> unique == same  # Assume we implemented __eq__
    True

-------------------
Continuation Tokens
-------------------

It is possible to record the state of an iterator and recreate that state in a separate thread or process using a
continuation token. Use the ``token`` property to retrieve a continuation token describing the current state of the
iterator. When recreating the iterator, pass the token to the
:func:`QueryIterator.move_to() <bloop.search.QueryIterator.move_to>` method to restore the previous state:

.. code-block:: pycon

    >>> query = engine.query(...)
    >>> for _ in range(10):
    ...     next(query) # read the first ten records.
    ...
    >>> token = query.token
    >>> resumed = engine.query(...)
    >>> resumed.move_to(token)
    >>> for _ in range(10):
    ...     next(query) # read the next ten records.

======
 Scan
======

Scan and :ref:`Query <user-query>` share a very similar interface.  Unlike Query, Scan does not have a key condition
and can't be performed in descending order.  Scans can be performed in parallel, however.

Using the same model from :ref:`user-query`, we can scan the model or an index:

.. code-block:: pycon

    >>> for account in engine.scan(Account):
    ...     print(account.email)
    ...
    >>> for account in engine.scan(Account.by_email):
    ...     print(account.email)

And get the first, or unique result:

.. code-block:: pycon

    >>> some_account = engine.scan(Account).first()
    >>> one_account = engine.scan(Account).one()
    Traceback (most recent call last):
        ...
    ConstraintViolation: Scan found more than one result.

Use ``filter`` and ``projection`` to exclude items and control which columns are included in results:

.. code-block:: pycon

    >>> scan = engine.scan(Account,
    ...     filter=Account.email.contains("@"),
    ...     projection=["level", "email"])

And ``consistent`` to use strongly consistent reads:

.. code-block:: pycon

    >>> scan = engine.scan(Account.by_balance, consistent=True)

----------------
 Parallel Scans
----------------

Scans can be performed `in parallel`__, using the ``parallel`` parameter.  To specify which segment you are
constructing the scan for, pass a tuple of ``(Segment, TotalSegments)``:

.. code-block:: pycon

    >>> first_segment = engine.scan(Account, parallel=(0, 2))
    >>> second_segment = engine.scan(Account, parallel=(1, 2))

You can easily construct a parallel scan with ``s`` segments by calling engine.scan in a loop:

.. code-block:: python

    def parallelize(s, engine, *args, **kwargs):
        for i in range(s):
            kwargs["parallel"] = (i, s)
            yield engine.scan(*args, **kargs)

    workers = scan_workers(n=10)
    scans = parallelize(10, engine, Account, filter=...)
    for worker, scan in zip(threads, scans):
        worker.process(scan)

__ http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/QueryAndScan.html#QueryAndScanParallelScan

==============
 Transactions
==============

.. note::

    For a detailed guide to using transactions, see the :ref:`user-transactions` section of the User Guide.

You can construct a read or write transaction by passing each mode:

.. code-block:: pycon

    >>> read_tx = engine.transaction(mode="r")
    >>> write_tx = engine.transaction(mode="w")  # defaults to write

You can also use the transaction as a context manager:

.. code-block:: pycon

    >>> with engine.transaction() as tx:
    ...     tx.save(user, condition=User.id.is_(None))
    ...     tx.delete(tweet, atomic=True)
    ...     tx.check(meta, Metadata.verified.is_(True))
    ...
    >>> # tx is committed or raises TransactionCanceled

To manually commit a transaction, call :func:`prepare() <bloop.transactions.Transaction.prepare>` and
:func:`commit() <bloop.transactions.PreparedTransaction.commit>`:

.. code-block:: pycon

    >>> tx = engine.transaction(mode="r")
    >>> tx.load(user, tweet)
    >>> prepared = tx.prepare()
    >>> prepared.commit()
    >>> prepared.commit()  # subsequent commits on a ReadTransaction re-load the objects

========
 Stream
========

.. note::

    Before you can create a stream on a model, you need to enable it in the model's :ref:`Meta <user-model-meta>`.
    For a detailed guide to using streams, head over to the :ref:`user-streams` section of the User Guide.

To start from the beginning or end of the stream, use "trim_horizon" and "latest":

.. code-block:: pycon

    >>> stream = engine.stream(User, position="trim_horizon")
    >>> stream = engine.stream(Account, "latest")

Alternatively, you can use an existing stream token to reload its previous state:

.. code-block:: pycon

    >>> same_stream = engine.stream(
    ...     Impression, previous_stream.token)

Lastly, you can use a datetime.  This is an **expensive call**, and walks the entire stream from the trim
horizon until it finds the first record in each shard after the target datetime.

.. code-block:: pycon

    >>> from datetime import datetime, timedelta, timezone
    >>> now = datetime.now(timezone.utc)
    >>> yesterday = now - timedelta(hours=12)
    >>> stream = engine.stream(User, yesterday)
