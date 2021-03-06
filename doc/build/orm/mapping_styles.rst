=================
Types of Mappings
=================

Modern SQLAlchemy features two distinct styles of mapper configuration.
The "Classical" style is SQLAlchemy's original mapping API, whereas
"Declarative" is the richer and more succinct system that builds on top
of "Classical".   Both styles may be used interchangeably, as the end
result of each is exactly the same - a user-defined class mapped by the
:func:`.mapper` function onto a selectable unit, typically a :class:`_schema.Table`.

Declarative Mapping
===================

The *Declarative Mapping* is the typical way that
mappings are constructed in modern SQLAlchemy.
Making use of the :ref:`declarative_toplevel`
system, the components of the user-defined class as well as the
:class:`_schema.Table` metadata to which the class is mapped are defined
at once::

    from sqlalchemy.ext.declarative import declarative_base
    from sqlalchemy import Column, Integer, String, ForeignKey

    Base = declarative_base()

    class User(Base):
        __tablename__ = 'user'

        id = Column(Integer, primary_key=True)
        name = Column(String)
        fullname = Column(String)
        nickname = Column(String)

Above, a basic single-table mapping with four columns.   Additional
attributes, such as relationships to other mapped classes, are also
declared inline within the class definition::

    class User(Base):
        __tablename__ = 'user'

        id = Column(Integer, primary_key=True)
        name = Column(String)
        fullname = Column(String)
        nickname = Column(String)

        addresses = relationship("Address", backref="user", order_by="Address.id")

    class Address(Base):
        __tablename__ = 'address'

        id = Column(Integer, primary_key=True)
        user_id = Column(ForeignKey('user.id'))
        email_address = Column(String)

The declarative mapping system is introduced in the
:ref:`ormtutorial_toplevel`.  For additional details on how this system
works, see :ref:`declarative_toplevel`.

.. _classical_mapping:

Classical Mappings
==================

A *Classical Mapping* refers to the configuration of a mapped class using the
:func:`.mapper` function, without using the Declarative system.  This is
SQLAlchemy's original class mapping API, and is still the base mapping
system provided by the ORM.

In "classical" form, the table metadata is created separately with the
:class:`_schema.Table` construct, then associated with the ``User`` class via
the :func:`.mapper` function::

    from sqlalchemy import Table, MetaData, Column, Integer, String, ForeignKey
    from sqlalchemy.orm import mapper

    metadata = MetaData()

    user = Table('user', metadata,
                Column('id', Integer, primary_key=True),
                Column('name', String(50)),
                Column('fullname', String(50)),
                Column('nickname', String(12))
            )

    class User(object):
        def __init__(self, name, fullname, nickname):
            self.name = name
            self.fullname = fullname
            self.nickname = nickname

    mapper(User, user)

Information about mapped attributes, such as relationships to other classes, are provided
via the ``properties`` dictionary.  The example below illustrates a second :class:`_schema.Table`
object, mapped to a class called ``Address``, then linked to ``User`` via :func:`_orm.relationship`::

    address = Table('address', metadata,
                Column('id', Integer, primary_key=True),
                Column('user_id', Integer, ForeignKey('user.id')),
                Column('email_address', String(50))
                )

    mapper(User, user, properties={
        'addresses' : relationship(Address, backref='user', order_by=address.c.id)
    })

    mapper(Address, address)

When using classical mappings, classes must be provided directly without the benefit
of the "string lookup" system provided by Declarative.  SQL expressions are typically
specified in terms of the :class:`_schema.Table` objects, i.e. ``address.c.id`` above
for the ``Address`` relationship, and not ``Address.id``, as ``Address`` may not
yet be linked to table metadata, nor can we specify a string here.

Some examples in the documentation still use the classical approach, but note that
the classical as well as Declarative approaches are **fully interchangeable**.  Both
systems ultimately create the same configuration, consisting of a :class:`_schema.Table`,
user-defined class, linked together with a :func:`.mapper`.  When we talk about
"the behavior of :func:`.mapper`", this includes when using the Declarative system
as well - it's still used, just behind the scenes.

.. _mapping_dataclasses:

Mapping dataclasses and attrs
-----------------------------

The dataclasses_ module, added in Python 3.7, provides a ``dataclass`` class
decorator to automatically generate boilerplate definitions of ``__init__()``,
``__eq__()``, ``__repr()__``, etc. methods. Another very popular library that does
the same, and much more, is attrs_. Classes defined using either of these can
be mapped with the following caveats.

.. versionadded:: 1.4 Added support for direct mapping of Python dataclasses.

The declarative "base" can't be used directly; a mapping function such as
:func:`_declarative.instrument_declarative` or :func:`_orm.mapper` may be
used.

The ``dataclass`` decorator adds class attributes corresponding to simple default values.
This is done mostly as documentation, these attributes are not necessary for the function
of any of the generated methods. Mapping replaces these class attributes with property
descriptors.

Mapping of frozen ``dataclass`` and ``attrs`` classes is not possible, because the
machinery used to enforce immutability interferes with loading.

Example using classical mapping::

    from __future__ import annotations
    from dataclasses import dataclass, field
    from typing import List

    from sqlalchemy import Column, ForeignKey, Integer, MetaData, String, Table
    from sqlalchemy.orm import mapper, relationship

    @dataclass
    class User:
        id: int = field(init=False)
        name: str = None
        fullname: str = None
        nickname: str = None
        addresses: List[Address] = field(default_factory=list)

    @dataclass
    class Address:
        id: int = field(init=False)
        user_id: int = field(init=False)
        email_address: str = None

    metadata = MetaData()

    user = Table(
        'user',
        metadata,
        Column('id', Integer, primary_key=True),
        Column('name', String(50)),
        Column('fullname', String(50)),
        Column('nickname', String(12)),
    )

    address = Table(
        'address',
        metadata,
        Column('id', Integer, primary_key=True),
        Column('user_id', Integer, ForeignKey('user.id')),
        Column('email_address', String(50)),
    )

    mapper(User, user, properties={
        'addresses': relationship(Address, backref='user', order_by=address.c.id),
    })

    mapper(Address, address)

Note that ``User.id``, ``Address.id``, and ``Address.user_id`` are defined as ``field(init=False)``.
This means that parameters for these won't be added to ``__init__()`` methods, but
:class:`.Session` will still be able to set them after getting their values during flush
from autoincrement or other default value generator. You can also give them a
``None`` default value instead if you want to be able to specify their values in the constructor.

.. _dataclasses: https://docs.python.org/3/library/dataclasses.html
.. _attrs: https://www.attrs.org/en/stable/

Runtime Introspection of Mappings, Objects
==========================================

The :class:`_orm.Mapper` object is available from any mapped class, regardless
of method, using the :ref:`core_inspection_toplevel` system.  Using the
:func:`_sa.inspect` function, one can acquire the :class:`_orm.Mapper` from a
mapped class::

    >>> from sqlalchemy import inspect
    >>> insp = inspect(User)

Detailed information is available including :attr:`_orm.Mapper.columns`::

    >>> insp.columns
    <sqlalchemy.util._collections.OrderedProperties object at 0x102f407f8>

This is a namespace that can be viewed in a list format or
via individual names::

    >>> list(insp.columns)
    [Column('id', Integer(), table=<user>, primary_key=True, nullable=False), Column('name', String(length=50), table=<user>), Column('fullname', String(length=50), table=<user>), Column('nickname', String(length=50), table=<user>)]
    >>> insp.columns.name
    Column('name', String(length=50), table=<user>)

Other namespaces include :attr:`_orm.Mapper.all_orm_descriptors`, which includes all mapped
attributes as well as hybrids, association proxies::

    >>> insp.all_orm_descriptors
    <sqlalchemy.util._collections.ImmutableProperties object at 0x1040e2c68>
    >>> insp.all_orm_descriptors.keys()
    ['fullname', 'nickname', 'name', 'id']

As well as :attr:`_orm.Mapper.column_attrs`::

    >>> list(insp.column_attrs)
    [<ColumnProperty at 0x10403fde0; id>, <ColumnProperty at 0x10403fce8; name>, <ColumnProperty at 0x1040e9050; fullname>, <ColumnProperty at 0x1040e9148; nickname>]
    >>> insp.column_attrs.name
    <ColumnProperty at 0x10403fce8; name>
    >>> insp.column_attrs.name.expression
    Column('name', String(length=50), table=<user>)

.. seealso::

    :ref:`core_inspection_toplevel`

    :class:`_orm.Mapper`

    :class:`.InstanceState`
