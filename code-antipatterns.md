# Antipatterns

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
* [Louis de Funès](https://en.wikipedia.org/wiki/Louis_de_Fun%C3%A8s) would never
  write a hack.

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

Hiding exceptions
-----------------

There are multiple variations of this anti-pattern:

```python
# Silence all exceptions
def toast(bread):
    try:
        toaster = Toaster()
        toaster.insert(bread)
        toaster.toast()
    except:
        pass


# Silence some exceptions
def toast(bread):
    try:
        toaster = Toaster()
        toaster.insert(bread)
        toaster.toast()
    except ValueError:
        pass
```

It depends on the context but in most cases this is a bad pattern:

* **Debugging** those silent errors will be really difficult, because they won't show up in logs and exception reporting tool such as Sentry.
* **The user experience** will randomly degrade without anybody knowing about it, including the user.
* **Identifying** those errors will be impossible. Say `do_stuff` does an HTTP request to another service, and that service starts misbehaving. There won't be any exception, any metric that will let you identify it.

An article even named this [the most diabolical Python antipattern](https://realpython.com/blog/python/the-most-diabolical-python-antipattern/).

Sometime it's tempting to think that graceful degradation is about silencing
exception. It's not.

* Graceful degradation needs to happen at the highest level of the code, so
  that the user can get a very explicit error message (e.g.  "we're having
  issues with X, please retry in a moment"). That requires knowing that there
  was an error, which you can't tell if you're silencing the exception.
* You need to know when graceful degradation happens. You also need to be
  alerted if it happens too often. This requires adding monitoring (using
  something like statsd) and logging (Python's `logger.exception` automatically
  adds the exception stacktrace to the log message for instance). Silencing an
  exception won't make the error go away: all things being equal, it's better
  for something to break hard, than for an error to be silenced.
* It is tempting to confound silencing the exception and fixing the exception. Say you're getting sporadic timeouts from a service. You might thing: let's ignore those timeouts and just do something else, like return an empty response. But this is very different from (1) actually finding the root cause for those timeouts (e.g. maybe a specific edge cases impacting certain objects) (2) doing proper graceful degradation (e.g. asking users to retry later because the request failed).

In other words, ask yourself: would it be a problem if every single action was
failing? If you're silencing the error, how would you know it's happening for
every single action?

Here's a number a better ways to do this:

**Log and create metrics**

```python
import statsd


def toast(bread):
    # Note: no exception handling here.
    toaster = Toaster()
    toaster.insert(bread)
    toaster.toast()


def main():
    try:
        toast('brioche')
    except:
        logger.exception('Could not toast bread')
        statsd.count('toast.error', 1)
```

**Very important**: adding logging and metrics won't not sufficient if it's difficult to consume them. They won't help the developer who's debugging. There needs to be automatic alerting associated to those stats. The logs needs to be surfaced somewhere, ideally next to the higher exception (e.g. let's our `main` function above is used in a web interface - the web interface could say "additionally, the following logs were generated and display the log). Otherwise the developer will still have to read the code to understand what's going on. Also - this won't work with sporadic errors. Those needs to be dealt with properly, and until then, it's better to let them go to your usual alerting tool.

Note that knowing where to catch is very important too. If you're catching
inside the `toast` function, you might be hiding things a caller would need to
know. Since this function is not returning anything, how would you make the
difference between a success and a failure? You can't. That's why you want to
let it raise, and catch only in the caller, where you have the context to know
how you'll handle the exception.

**Re-raise immediately**

```python
import statsd


def toast(bread):
    try:
        toaster = Toaster()
        toaster.insert(bread)
        toaster.toast()
    except:
        raise ToastingException('Could not toast bread %r' % bread)


def main():
    # Note: no exception handling here
    toast('brioche')
```

**Be very specific about the exception that are caught**

```python
import statsd


def toast(bread):
    try:
        toaster = Toaster()
    except ValueError:
        # Note that even though we catch, we're still logging + creating
        # a metric
        logger.exception('Could not get toaster')
        statsd.count('toaster.instantiate.error', 1)
        return

    toaster.insert(bread)
    toaster.toast()


def main():
    toast('brioche')
```

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

Unreadable response construction
--------------------------------

TODO

Bad:

```python
def get_data():
    returned = {}
    if stuff:
        returned['toaster'] = 'toaster'
    if other_stuff:
        if the_other_stuff:
            returned['toast'] = 'brioche'
    else:
        returned['toast'] = 'bread'
    return returned
```

Good:

```python
def get_data():
    returned = {
        'toaster': '',
        'toast': '',
    }
```

Undeterministic tests
---------------------

When testing function that don't behave deterministically, it can be tempting
to run them multiple time and average their results.

Bad:

```python
def function():
    if random.random() > .4:
        return True
    else:
        return False


def test_function():
    number_of_true = 0
    for _ in xrange(1000):
        returned = function()
        if returned:
            number_of_true += 1

    assert 30 < number_of_true < 50
```

There are multiple things that are wrong with this approach:

* This is a flaky test. Theoretically, this test could still fail.
* This example is simple enough, but `function` might be doing some
  computationally expensive task, which would make this test severely
  inefficient.
* The test is quite difficult to understand.

Good:

```python
@mock.patch('random.random')
def test_function(mock_random):
    mock_random.return_value = 0.7
    assert function() is True
```

This is a deterministic test that clearly tells what's going on.

Unbalanced boilerplate
----------------------

One thing to strive for in libraries is have as little boilerplate as possible,
but not less.

Not enough boilerplate: you'll spend hours trying to understand specific
behaviors that are too magical/implicit. You will need flexibility and you
won't be able to get it. Boilerplate is useful insofar as it increases
[transparency](http://www.catb.org/esr/writings/taoup/html/ch01s06.html).

Too much boilerplate: users of your library will be stuck using outdated
patterns. Users will write library to generate the boilerplate required by your
library.

I think Flask and SQLAlchemy do a very good job at keeping this under control.

Inconsistent use of verbs in functions
--------------------------------------

Bad:

```python
def get_toasters(color):
    """Get a bunch of toasters."""
    return filter(lambda toaster: toaster.color == color, TOASTERS)


def find_toaster(id_):
    """Return a single toaster."""
    toasters = filter(lambda toaster: toaster.id == id_, TOASTERS)
    assert len(toasters) == 1
    return toasters[1]


def find_toasts(color):
    """Find a bunch of toasts."""
    return filter(lambda toast: toast.color == color, TOASTS)
```

The use of verb is inconsistent in this example. `get` is used to return
a possibly empty list of toasters, and `find` is used to return a single
toaster (or raise an exception) in the second function or a possibly empty list
of toasts in the third function.

This is based on personal taste but I have the following rule:

* `get` prefixes function that return at most one object (they either return
  none or raise an exception depending on the cases)
* `find` prefixes function that return a possibly empty list (or iterable) of
  objects.

Good:

```python
def find_toasters(color):
    """Find a bunch of toasters."""
    return filter(lambda toaster: toaster.color == color, TOASTERS)


def get_toaster(id_):
    """Return a single toaster."""
    toasters = filter(lambda toaster: toaster.id == id_, TOASTERS)
    assert len(toasters) == 1
    return toasters[1]


def find_toasts(color):
    """Find a bunch of toasts."""
    return filter(lambda toast: toast.color == color, TOASTS)
```

Opaque function arguments
-------------------------

A few variants of what I consider code that is difficult to debug:

```python
def create(toaster_params):
    name = toaster_params['name']
    color = toaster_params.get('color', 'red')


class Toaster(object):

    def __init__(self, params):
        self.name = params['name']


# Probably the worst of all
def create2(*args, **kwargs):
    name = kwargs['name']
```

Why is this bad?

* It's really easy to make a mistake, especially in interpreted languages such
  as Python. For instance, if I call `create({'name': 'hello', 'ccolor':
  'blue'})`, I won't get any error, but the color won't be the one I expect.
* It increases cognitive load, as I have to understand where the object is
  coming from to introspect its content.
* It makes the job of static analyzer harder or impossible.

Granted, this pattern is sometimes required (for instance when the number of
params is too large, or when dealing with pure data).

A better way is to be explicit:

```python
def create(name, color='red'):
    pass  # ...
```

Hiding formatting
-----------------

Bad:

```python
# main.py

from utils import format_query


def get_user(user_id):
    url = get_url(user_id)
    return requests.get(url)


# utils.py


def get_url(user_id):
    return 'http://127.0.0.1/users/%s' % user_id
```

I consider this an antipattern because it hides the request formatting from the
developer, making it more complex to see what `url` look like. In this extreme
example, the formatting function is a one-liner which sounds a bit overkill for
a function.

Good:

```python
def get_user(user_id):
    url = 'http://127.0.0.1/users/%s' % user_id
    return requests.get(url)
```

## Unconstrained defensive programming

While defensive programming can be a very good technique to make the code more resilient, it can seriously backfire when misused. This is a very similar anti-pattern to carelessly silencing exceptions (see about this anti-pattern in this document).

One example is to handle an edge case as a generic case at a very low level. Consider the following example:

```python
def get_user_name(user_id):
    url = 'http://127.0.0.1/users/%s' % user_id
    response = requests.get(url)
    if response.status == 404:
        return 'unknown'
    return response.data
```

While this may look like a very good example of defensive programming (we're returning `unknown` when we can't find the user), this can have terrible repercussions, very similar to the one we have when doing an unrestricted bare `try... except`:

* A new developer might not know about this magical convention, and assume that `get_user_name` is guaranteed to return a true user name.
* The external service that we're getting user name from might start failing, and returning 404. We would silently return 'unknown' as a user name for all users, which could have terrible repercussions.

A much cleaner way is to raise an exception on 404, and let the caller decide how it wants to handle users that are not found.

