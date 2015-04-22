Antipatterns
============

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
