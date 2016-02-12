<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Test antipatterns](#test-antipatterns)
  - [Testing implementation](#testing-implementation)
  - [Testing configuration](#testing-configuration)
  - [Testing multiple things](#testing-multiple-things)
  - [Repeating integration tests for minor variations](#repeating-integration-tests-for-minor-variations)
  - [Over-reliance on centralized fixtures](#over-reliance-on-centralized-fixtures)
  - [Over-reliance on replaying external requests](#over-reliance-on-replaying-external-requests)
  - [Inefficient query testing](#inefficient-query-testing)
  - [Assertions in loop](#assertions-in-loop)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

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

Bad:

```python
# fixtures.py
toaster = Toaster(color='black')
toaster_with_color_blue = Toaster(color='blue')
toaster_with_color_red = Toaster(color='red')


# test.py
from fixtures import toaster, toaster_with_color_blue


def test_stuff():
    toaster_with_color_blue.toast('brioche')
```

The problem with centralized fixtures is that they tend to grow exponentially.
Every single test case will have some specific fixtures requirements, and every
single permutation will have its own fixture. This will make the fixture file
really difficult to maintain.

The other problem is that it will become very easy for two unrelated tests to
use the same fixture. Now let's say one of those test's requirement changes,
and you have to change the fixture as well. In that case, you might break the
other test, which will slow you down and defeats the purpose of keeping test an
efficient part of the developer flow.

Lastly, this separate the setup and running part of the tests. It makes it more
difficult for a new engineer to understand what is specific about this test's
setup without having to open the ``fixtures`` file.

Here's a more explicit way to do this. Most fixtures libraries allow you to
override default parameters, so that you can make clear what setup is specific
to each test.

```python
def test_stuff():
    toaster_with_color_blue = Toaster(color='blue')
    toaster_with_color_blue.toast('brioche')
```

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
