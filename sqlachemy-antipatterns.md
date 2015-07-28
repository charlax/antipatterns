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

Loading the full object when checking for object existence
----------------------------------------------------------

Bad:

```python
def toaster_exists(toaster_id):
    return bool(session.query(Toaster).filter_by(id=toaster_id).first())
```

This is inefficient because it:

* Queries all the columns from the database (including any eagerly loaded joins)
* Instantiates and maps all data on the Toaster model

The database query would look something like this. You can see that all columns
are selected to be loaded by the ORM.

```sql
SELECT toasters.id AS toasters_id, toasters.name AS toasters_name,
toasters.color AS toasters_color
FROM toasters
WHERE toasters.id = 1
 LIMIT 1 OFFSET 0
```

And then it just checks if the result is truthy.

Here's a better way to do it:

```python
def toaster_exists(toaster_id):
    query = session.query(Toaster).filter_by(id=toaster_id)
    return session.query(query.exists()).scalar()
```

In this case, we just ask the database about whether a record exists with this
id. This is obviously much more efficient.

```sql
SELECT EXISTS (SELECT 1
FROM toasters
WHERE toasters.id = 1) AS anon_1
```
