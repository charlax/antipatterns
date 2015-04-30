Antipatterns
============

Most of those are antipatterns in the Python programming language, but some of
them might be more generic.

Strict email validation
-----------------------

It is almost impossible to strictly validate an email. Even if you were writing
or using a regex that follows
[RFC5322](http://tools.ietf.org/html/rfc5322#section-3.4), you would have false
positives when trying to validate actual emails that don't follow the RFC.

What's more, validating an email provides very weak guarantees. A stronger,
more meaningful validation would be to send an email and validate that the user
received it.

To sum up, don't waste your time trying to validate an email if you don't need
to (or just check that there's a `@` in it). If you need to, send an email with
a token and validate that the user received it.

Late returns
------------

Returning early reduces cognitive overhead, and improve readability by killing
indentation levels.

Bad:

```python
def toast(bread):
    if bread.kind != 'brioche':
        if not bread.is_stale:
            toaster.toast(bread)
```

Good:

```python
def toast(bread):
    if bread.kind == 'brioche' or bread.is_stale:
        return

    toaster.toast(bread)
```

Hacks comment
-------------

Bad:

```python
# Gigantic hack (written by Louis de Funes) 04-01-2015
toaster.restart()
```

There's multiple things wrong with this comment:

* Even if it is actually a hack, no need to say it in a comment. It lowers the
  perceived quality of a codebase and impacts developer motivation.
* Putting the author and the date is totally useless when using source control
  (`git blame`).
* This does not explain why it's temporary.
* It's impossible to easily grep for temporary fixes.

Good:

```python
# Need to restart toaster to prevent burning bread
# TODO: replace with proper fix
toaster.restart()
```

* This clearly explains the nature of the temporary fix.
* Using `TODO` is an ubiquitous pattern that allows easy grepping and plays
  nice with most text editors.
* The perceived quality of this temporary fix is much higher.

Repeating arguments in function name
------------------------------------

Bad:

```python
def get_by_color(color):
    return Toasters.filter_by(color=color)
```

Putting the argument name in both the function name and in arguments is, in
most cases and for most interpreted languages, redundant.

Good:

```python
def get(color=None):
    if color:
        return Toasters.filter_by(color=color)
```

Repeating function name in docstring
------------------------------------

Bad:

```python
def test_return_true_if_toast_is_valid():
    """Verify that we return true if toast is valid."""
    assert is_valid(Toast('brioche')) is true
```

Why is it bad?

* The docstring and function name are not DRY.
* There's no actual explanation of what valid means.

Good:

```python
def test_valid_toast():
    """Verify that 'brioche' are valid toasts."""
    assert is_valid(Toast('brioche')) is true
```

Or, another variation:

```python
def test_brioche_are_valid_toast():
    assert is_valid(Toast('brioche')) is true
```

Bare try... except...
---------------------

TODO

Raising unrelated/unspecific exception
--------------------------------------

Most languages have all predefined exceptions, including Python. It is
important to make sure that the right exception is raised from a semantic
standpoint.

Bad:

```python
def validate(toast):
    if isinstance(toast, Brioche):
        # RuntimeError is too broad
        raise RuntimeError('Invalid toast')


def validate(toast):
    if isinstance(toast, Brioche):
        # SystemError should only be used for internal interpreter errors
        raise SystemError('Invalid toast')
```

Good:

```python
def validate(toast):
    if isinstance(toast, Brioche):
        raise TypeError('Invalid toast')
```

`TypeError` is here perfectly meaningful, and clearly convey the context around
the error.
