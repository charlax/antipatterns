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

Reference: [setup.py vs.
requirements.txt](https://caremad.io/2013/07/setup-vs-requirement/)
