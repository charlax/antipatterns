Python Antipatterns
===================

Redundant type checking
-----------------------

Bad:

```python
def toast(bread):
    if bread is None:
        raise TypeError('Need bread to toast.')
    if bread.is_toastable:
        toaster.toast(bread)
```

In this case, checking against `None` is totally useless because in the next
line, `bread.is_toastable` would raise `AttributeError: 'NoneType' object has
no attribute 'is_toastable'`. This is not a general rule, but in this case
I would definitely argue that adding the type checks hurts readability and adds
very little value to the function.

Good:

```python
def toast(bread):
    if bread.is_toastable:
        toaster.toast(bread)
```

Restricting version in setup.py dependencies
--------------------------------------------

Read this article first: [setup.py vs.
requirements.txt](https://caremad.io/2013/07/setup-vs-requirement/)

The main point is that `setup.py` should not specify explicit version
requirements (good: `flask`, bad: `flask==1.1.1`).

In a few words, if library A requires `flask==1.1.1` and library B requires
`flask==1.1.2`, then you'll have a conflict and won't be able to use them both
in application Z. Yet in 99.999% of the cases, you don't need a specific
version of flask, so A should just require `flask` in `setup.py` (no version
specified), B should just require `flask` in `setup.py` (same), and Z will be
happy using A and B with whatever version of flask it wants (this specific
version will be in `requirements.txt`, usually apps don't really need to
maintain dependencies in both `setup.py` and `requirements.txt`, only
`requirements.txt` is usually used). Then the developers of A can put whatever
version of flask they're currently developing against in A's
`requirements.txt`.

Ruby has a pretty similar dichotomy with [Gemspec and
gemfile](http://yehudakatz.com/2010/12/16/clarifying-the-roles-of-the-gemspec-and-gemfile/).

Unwieldy if... else instead of dict
-----------------------------------

Bad:

```python
import operator as op


def get_operator(value):
    """Return operator function based on string.

    e.g. ``get_operator('+')`` returns a function which takes two arguments
    and return the sum of them.
    """
    if value == '+':
        return op.add
    elif value == '-':
        return op.sub
    elif value == '*':
        return op.mul
    elif value == '/':
        return op.div
    else:
        raise ValueError('Unknown operator %s' % value)
```

Note: the operator module is a standard library modules that defines base
operator functions. For instance `operator.add(1, 1) == 2`.

This huge switch-like expression will soon become really difficult to read and
maintain. A more pythonic way is to use a dict to store the mapping.

Another reason is that to get 100% line and branch coverage, you will have to
create as many tests as you have mappings. Yet you could consider that the
value to operator mapping is part of the codebase's configuration, not its
behavior, and thus shouldn't be tested.

Good:

```python
import operator as op

OPERATORS = {
    '+': op.add,
    '-': op.sub,
    '*': op.mul,
    '/': op.div,
}


def get_operator(value):
    """Return operator function based on string."""
    operator = OPERATORS.get(value)
    if operator:
        return operator
    else:
        raise ValueError('Unknown operator %s' % value)
```

Overreliance on kwargs
----------------------

TODO

Overreliance on list/dict comprehensions
----------------------------------------

TODO

Mutable default arguments
-------------------------

TODO

Unnecessarily re-raising exceptions
-----------------------------------

Bad:

```python
def toast(bread):
    try:
        put_in_toaster(bread)
    except:  # or except Exception as e
        raise ToastException('Could not toast bread')
```

There's two problems with this implementation:

* We're unconditionally catching all exceptions without being restrictive.
  Let's say that `put_in_toaster` is undefined. Instead of giving proper
  feedback to the developer, we will reraise `ToastException`.
* We're hiding what's happening in `put_in_toaster`. Let's say there's
  a programmer error in `put_in_toaster` (for instance, an undefined variable).
  Instead of getting an immediate feedback, this will be reraised as
  `ToastException`, and the developer will waste hours trying to find where the
  exception was raised.

Here's an example `put_in_toaster` implementation. In this case, the typo would
be hidden by the bare `try... except`:

```python
def put_in_toaster(bread):
    breadd.color = 'light_brown'  # Note the typo that would be hidden
```

The following full example:

```python
from collections import namedtuple

Bread = namedtuple('Bread', 'color')

class ToastException(Exception):
    pass

def toast(bread):
    try:
        put_in_toaster(bread)
    except:
        raise ToastException('Could not toast bread')


def put_in_toaster(bread):
    brad.color = 'light_brown'  # Note the typo


toast(Bread('yellow'))
```

Will raise this cryptic and impossible to debug error:

```
Traceback (most recent call last):
  File "python-examples/reraise_exceptions.py", line 19, in <module>
    toast(Bread('yellow'))
  File "python-examples/reraise_exceptions.py", line 12, in toast
    raise ToastException('Could not toast bread')
__main__.ToastException: Could not toast bread
```

This simple example will evidently be even more painful in a large codebase.

This is a better pattern because we're very selective about the exception we
catch:

```python
def toast(bread):
    try:
        put_in_toaster(bread)
    except InvalidBreadType as e:
        raise ToastException('Cannot toast this bread type')
```

This is another good pattern. Because we're re-raising without a specific
exception, the full traceback will be kept:

```python
def toast(bread):
    try:
        put_in_toaster(bread)
    except:
        print 'Got exception while trying to toast'
        raise  # note the absence of specific exception
```

Here's what would happen:

```
Got exception while trying to toast
Traceback (most recent call last):
  File "reraise_exceptions_good.py", line 20, in <module>
    toast(Bread('yellow'))
  File "reraise_exceptions_good.py", line 10, in toast
    put_in_toaster(bread)
  File "reraise_exceptions_good.py", line 17, in put_in_toaster
    brad.color = 'light_brown'  # Note the typo
NameError: global name 'brad' is not defined
```

Using `is` to compare objects
-----------------------------

TODO

[Why you should almost never use “is” in
Python](http://blog.lerner.co.il/why-you-should-almost-never-use-is-in-python/)

Reference
---------

* [Pythonic Pitfalls](http://nafiulis.me/potential-pythonic-pitfalls.html)
* [Python Patterns](https://github.com/faif/python-patterns)
* [The Little Book of Python
  Anti-Patterns](http://docs.quantifiedcode.com/python-anti-patterns/)
