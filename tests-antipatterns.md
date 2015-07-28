Test antipatterns
=================

Testing implementation
----------------------

TODO

Testing configuration
---------------------

TODO

Testing multiple things
-----------------------

TODO

Repeating integration tests for minor variations
------------------------------------------------

TODO

Over-reliance on centralized fixtures
-------------------------------------

TODO

Over-reliance on replaying external requests
--------------------------------------------

TODO

Inefficient query testing
-------------------------

Bad:

```python
def test_find():
    install_fixture('toaster_1')  # color is red
    install_fixture('toaster_2')  # color is blue
    install_fixture('toaster_3')  # color is blue
    install_fixture('toaster_4')  # color is green

    results = find(color='blue')
    assert len(results) == 2
    for r in results:
        assert r.color == 'blue'
```

This is inefficient because we're installing four fixtures, even though we
could probably get away with only one. In most cases this would achieve the
same thing at a much lower cost:

```python
def test_find():
    install_fixture('toaster_2')  # color is blue
    results = find(color='blue')
    # Note the implicit assertion that the list is not empty because we're
    # accessing its first item.
    assert results[0].color == 'blue'
```

The general rule here is that tests should require the minimum amount of code
to verify the behavior.

One would also recommend to not do this kind of integration testing for queries
going to the database, but sometimes it's a good tradeoff.


Assertions in loop
------------------

Bad:

```python
def test_find():
    results = find(color='blue')
    for r in results:
        assert r.color == 'blue'
```

This is bad because in most languages, if the list is empty then the test will
pass, which is probably not what we want to test.

Good:

```python
def test_find():
    results = find(color='blue')
    assert len(results) == 5
    for r in results:
        assert r.color == 'blue'
```

This is also a matter of tradeoff, but in most cases I'd try to make sure the
list contains only one item. If we're checking more than one item, that hints
at our test trying to do too many things.
