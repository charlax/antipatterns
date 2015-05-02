SQLAlchemy Anti-Patterns
========================

This is a list of what I consider [SQLAlchemy](http://www.sqlalchemy.org/)
anti-patterns.

Abusing lazily loaded relationships
-----------------------------------

Bad:

```python
class Customer(Base):

    @property
    def has_valid_toast(self):
        """Return True if customer has at least one valid toast."""
        return any(toast.kind == 'brioche' for toast in self.toaster.toasts)
```

This suffers from severe performance inefficiencies:

* The toaster will be loaded, as well as its toast. This involves creating and
  issuing the SQL query, waiting for the database to return, and instantiating
  all those objects.
* `has_valid_toast` does not actually care about those objects. It just returns
  a boolean.

A better way would be to issue a SQL `EXISTS` query so that the database
handles this check and only returns a boolean.

Good:

```python
class Customer(Base):

    @property
    def has_valid_toast(self):
        """Return True if customer has at least one valid toast."""
        query = (session.query(Toaster)
                 .join(Toast)
                 .with_parent(self)
                 .filter(Toast.kind == 'brioche'))
        return session.query(query.exists()).scalar()
```

This query might not always be the fastest if those relationships are small,
and eagerly loaded.

Explicit session passing
------------------------

TODO

Bad:

```python
def toaster_exists(toaster_id, session):
    ...
```

Implicit transaction handling
-----------------------------

TODO
