PynamoDB Tutorial
===================

PynamoDB is attempt to be a Pythonic interface to DynamoDB that supports all of DynamoDB's
powerful features in *both* Python 3, and Python 2. This includes support for unicode and
binary attributes.

But why stop there? PynamoDB also supports:

* Sets for Binary, Number, and Unicode attributes
* Automatic pagination for bulk operations
* Global secondary indexes
* Local secondary indexes
* Complex queries

Why PynamoDB?
^^^^^^^^^^^^^

It all started when I needed to use Global Secondary Indexes, a new and powerful feature of
DynamoDB. I quickly realized that my go to library, `dynamodb-mapper <http://dynamodb-mapper.readthedocs.org/en/latest/>`__, didn't support them.
In fact, it won't be supporting them anytime soon because dynamodb-mapper relies on another
library, `boto.dynamodb <http://docs.pythonboto.org/en/latest/migrations/dynamodb_v1_to_v2.html>`__,
which itself won't support them. In fact, boto doesn't support
Python 3 either. If you want to know more, `I blogged about it <http://jlafon.io/pynamodb.html>`__.

Installation
^^^^^^^^^^^^

::

    $ pip install pynamodb

.. note::

    PynamoDB is still under development. More advanced features are available with the development version
    of PynamoDB. You can install it directly from GitHub with pip: `pip install git+https://github.com/jlafon/PynamoDB#egg=pynamodb`

Getting Started
^^^^^^^^^^^^^^^

PynamoDB provides three API levels, a ``Connection``, a ``TableConnection``, and a ``Model``.
Each API is built on top of the previous, and adds higher level features. Each API level is
fully featured, and can be used directly. Before you begin, you should already have an
`Amazon Web Services account <http://aws.amazon.com/>`__, and have your
`AWS credentials configured your boto <http://boto.readthedocs.org/en/latest/boto_config_tut.html>`__.

Defining a Model
----------------

The most powerful feature of PynamoDB is the ``Model`` API. You start using it by defining a model
class that inherits from ``pynamodb.models.Model``. Then, you add attributes to the model that
inherit from ``pynamodb.attributes.Attribute``. The most common attributes have already been defined for you.

Here is an example, using the same table structure as shown in `Amazon's DynamoDB Thread example <http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SampleTablesAndData.html>`__.

.. code-block:: python

    from pynamodb.models import Model
    from pynamodb.attributes import (
        UnicodeAttribute, NumberAttribute, UnicodeSetAttribute, UTCDateTimeAttribute
    )


    class Thread(Model):
        table_name = 'Thread'
        forum_name = UnicodeAttribute(hash_key=True)
        subject = UnicodeAttribute(range_key=True)
        views = NumberAttribute(default=0)
        replies = NumberAttribute(default=0)
        answered = NumberAttribute(default=0)
        tags = UnicodeSetAttribute()
        last_post_datetime = UTCDateTimeAttribute()

The ``table_name`` class attribute is required, and tells the model which DynamoDB table to use. The ``forum_name`` attribute
is specified as the hash key for this table with the ``hash_key`` argument. Specifying that an attribute is a range key is done
with the ``range_key`` attribute. You can specify a default value for any field, and `default` can even be a function.

PynamoDB comes with several built in attribute types for convenience, which include the following:

* :py:class:`UnicodeAttribute <pynamodb.attributes.UnicodeAttribute>`
* :py:class:`UnicodeSetAttribute <pynamodb.attributes.UnicodeSetAttribute>`
* :py:class:`NumberAttribute <pynamodb.attributes.NumberAttribute>`
* :py:class:`NumberSetAttribute <pynamodb.attributes.NumberSetAttribute>`
* :py:class:`BinaryAttribute <pynamodb.attributes.BinaryAttribute>`
* :py:class:`BinarySetAttribute <pynamodb.attributes.BinarySetAttribute>`
* :py:class:`UTCDateTimeAttribute <pynamodb.attributes.UTCDateTimeAttribute>`
* :py:class:`BooleanAttribute <pynamodb.attributes.BooleanAttribute>`
* :py:class:`JSONAttribute <pynamodb.attributes.JSONAttribute>`

All of these built in attributes handle serializing and deserializng themselves, in both Python 2 and Python 3.

Creating the table
------------------

If your table doesn't already exist, you will have to create it. This can be done with easily:

.. code-block:: python

    >>> if not Thread.exists():
            Thread.create_table(read_capacity_units=1, write_capacity_units=1, wait=True)

The ``wait`` argument tells PynamoDB to wait until the table is ready for use before returning.


Using the Model
^^^^^^^^^^^^^^^

Now that you've defined a model (referring to the example above), you can start interacting with
your DynamoDB table. You can create a new `Thread` item by calling the `Thread` constructor.

Creating Items
--------------
.. code-block:: python

    >>> thread_item = Thread('forum_name', 'forum_subject')

The first two arguments are automatically assigned to the item's hash and range keys. You can
specify attributes during construction as well:

.. code-block:: python

    >>> thread_item = Thread('forum_name', 'forum_subject', replies=10)

The item won't be added to your DynamoDB table until you call save:

.. code-block:: python

    >>> thread_item.save()

If you want to retrieve an item that already exists in your table, you can do that with `get`:

.. code-block:: python

    >>> thread_item = Thread.get('forum_name', 'forum_subject')

If the item doesn't exist, `None` will be returned.

Updating Items
--------------

You can update an item with the latest data from your table:

.. code-block:: python

    >>> thread_item.refresh()

Updates to table items are supported too, even atomic updates. Here is an example of
atomically updating the view count of an item:

.. code-block:: python

    >>> thread_item.update_item('views', 1, action='add')

Batch Operations
^^^^^^^^^^^^^^^^

Batch operations are supported using context managers, and iterators. The DynamoDB API has limits for each batch operation
that it supports, but PynamoDB removes the need implement your own grouping or pagination. Instead, it handles
pagination for you automatically.

Batch Writes
-------------

Here is an example using a context manager for a bulk write operation:

.. code-block:: python

    with Thread.batch_write() as batch:
        items = [TestModel('forum-{0}'.format(x), 'thread-{0}'.format(x)) for x in range(1000)]
        for item in items:
            batch.save(item)

Batch Gets
-------------

Here is an example using an iterator for retrieving items in bulk:

.. code-block:: python

    item_keys = [('forum-{0}'.format(x), 'thread-{0}'.format(x)) for x in range(1000)]
    for item in Thread.batch_get(item_keys):
        print(item)

Query Filters
-------------

You can query items from your table using a simple syntax, similar to other Python ORMs:

.. code-block:: python

    for item in Thread.query('ForumName', thread__begins_with='mygreatprefix'):
        print("Query returned item {0}".format(item))

Query filters are translated from an ORM like syntax to DynamoDB API calls, and use
the following syntax: `attribute__operator=value`, where `attribute` is the name of an attribute
and `operator` can be one of the following:

 * eq
 * le
 * lt
 * ge
 * gt
 * begins_with
 * between

Scan Filters
------------

Scan filters have the same syntax as Query filters, but support different operations (a consequence of the
DynamoDB API - not PynamoDB). The supported operators are:

 * eq
 * ne
 * le
 * lt
 * gt
 * not_null
 * null
 * contains
 * not_contains
 * begins_with
 * between

You can even specify multiple filters:

.. code-block:: python

    >>> for item in Thread.scan(forum__begins_with='Prefix', views__gt=10):
            print(item)