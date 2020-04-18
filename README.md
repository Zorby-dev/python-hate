A growing list of things I dislike about Python.

There are workarounds for some of them (often half-broken and usually unintuitive)
and others may even be considered virtues by some people.

# Generic

## Zero static checking

The most critical problem of Python is that it does not perform any static checks
(not even missing variable definitions).
Lack of static checking increases debugging time and makes refactoring
more time-consuming than needed.
This becomes particularly obvious when you run your app on a huge amount of data overnight,
just to detect missing initialization in some rarely called function in the morning.

There is [Pylint](https://www.pylint.org) but it is a linter (i.e. style checker)
rather than a real static analyzer so it is unable, by design, to detect
many serious errors in programs which require dataflow analysis.
For example if fails on basic stuff like
* invalid string formatting (fixed [here](https://github.com/PyCQA/pylint/pull/2465))
* iterating over unsorted dicts (reported [here](https://github.com/PyCQA/pylint/issues/2467) with draft patch, rejected because maintainers consider it unimportant (no particular reasons provided))
* dead list computations (e.g. using `sorted(lst)` instead of `lst.sort()`)
* modifying list while iterating over it (reported [here](https://github.com/PyCQA/pylint/issues/2471) with draft patch, rejected because maintainers consider it unimportant (no particular reasons provided))
* etc.

## Global interpreter lock

[GIL](https://wiki.python.org/moin/GlobalInterpreterLock) precludes high-performant multithreading
which is suprising at the age of multicores.

## No type annotations

Lack of type annotations forces people to use hungarian notation
in complex programs (hello 90-s!).

# Language

## Negation operator syntax issue

This is not a valid syntax:
```
x == not y
```

## Hiding type errors via helpful conversions

It's very easy to write `len(lst1) == lst2` instead of `len(lst1) == len(lst2)`.
Python will helpfully make it harder to find this error
by converting first variant to `[len(lst1)] * len(lst2) == lst2`
(instead of aborting with a type fail).

## Limited lambdas

For unclear reason lambda functions only support expressions
so anything more complicated requires a local named function.

## Problematic operator precedence

The `is` and `is not` operators have the same precedence as comparisons
so this code
```
op.post_modification is None != full_op.post_modification is None
```
would rather unexpectedly evalute as
```
((op.post_modification is None) != full_op.post_modification) is None
```

## The useless self

Explicitly writing out `self` in all method declarations and calls
should not be needed. Apart from being an unnecessary boilerplate,
this enables another class of bugs:
```
class A:
  def first(x, *y):
    return x

a = A
print(a.first(1,2,3))  # Will print a, not 1
```

## Inconsistency of set literals

Sets can be initialized via syntax sugar:
```
>>> x = {1, 2, 3}
>>> type(x)
<class 'set'>
```
but it breaks for empty sets:
```
>>> x = {}
>>> type(x)
<class 'dict'>
```

## Syntax checking

Syntax error reporting in Python is extremely primitive.
In most cases you simply get `SyntaxError: invalid syntax`.

## Inadvertent sharing

It's too easy to inadvertently share references:
```
a = b = []
```
or
```
def foo(x=[]):  # foo() will return [1], [1, 1], [1, 1, 1], etc.
  x.append(1)
  return x
```
or even
```
def foo(obj, lst=[]):
  obj.lst = lst
foo(obj)
obj.lst.append(1)  # Hoorah, this modifies default value of foo
```

## Functions always return

Default return value from function (when `return` is omitted) is `None`.
This makes it impossible to declare subroutines which are not supposed
to return anything (and verify this at runtime).

## Automatic field insertion

Assigning a non-existent object field adds it instead of throwing an exception:
```
class A:
  def __init__(self):
    self.x = 0
...
a = A()
a.y = 1  # OK
```

This complicates refactoring because forgetting to update an outdated field name
deep inside your (or your colleague's) program will silently work,
breaking your program much later.

This can be overcome with `__slots__` but when have you seen them
used last time?

## "Is" operator does not work for primitive types

No comments:
```
> 2+2 is 4
True
> 999+1 is 1000
False
```

This happens because [only sufficiently small integer objects are reused](https://stackoverflow.com/a/306353/2170527):
```
# Two different instances of number "1000"
>>> id(999+1)
140068481622512
>>> id(1000)
140068481624112

# Single instance of number "4"
>>> id(2+2)
10968896
>>> id(4)
10968896
```

## Inconsistent index checks

Invalid indexing throws an exception
but invalid slicing does not:
```
>>> a=list(range(4))
>>> a[4]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
IndexError: list index out of range
>>> a[4:10]
[]
```

# Standard Libraries

## List generators fail check for emptiness

Thanks to active use of generators in Python 3 it became easier
to misuse standard APIs:
```
if filter(lambda x: x == 0, [1,2]):
  print("Yes")  # Prints "Yes"!
```
Surpisingly enough this does not apply to `range` (i.e. `bool(range(0))` returns `True`).

## Argparse does not support negated flags

`argparse` does not provide automatic support for `--no-XXX` flags.

## Getopt does not allow parameters after positional arguments

```
>>> opts, args = getopt.getopt(['A', '-o', 'B'], 'o:')
>>> opts
[]
>>> args
['A', '-o', 'B']
>>> opts, args = getopt.getopt(['-o', 'B', 'A'], 'o:')
>>> opts
[('-o', 'B')]
>>> args
['A']
```

## Split and join disagree on argument order

`split` and `join` accept list and separator in different order:
```
sep.join(lst)
lst.split(sep)
```

# Name Resolution

## No way to localize a name

There is no way to localize variable in a scope smaller than a function.
This often hurts when renaming variables during code refactoring.
Forgetting to rename variable name in a single place, causes interpreter
to pick up an unrelated name from unrelated block 50 lines above or
from previous loop iteration.

This is especially inconvenient for one-off variables (e.g. loop counters):
```
for i in range(100):
  ...

# 100 lines later

for j in range(200):
  a[i] = 0  # Yikes, forgot to rename!
```

One could remove a name from scope via `del i` but this is considered too
verbose so noone uses it.

## Everything is exported

Similarly to above, there is no control over visibility (can't hide class methods,
can't hide module functions). You are left with a _convention_ to precede
private functions with `_` and hope for the best.

## Assignment makes variable local

Python scoping rules require that assigning a variable automatically declares it local.
This causes inconsistencies and weird limitations in practice. E.g. variable would
be considered local even if assignment follows first use:
```
global xxx
xxx = 0

def foo():
  a = xxx  # Throws UnboundLocalError
  xxx = 2
```
and even if assignment is never ever executed:
```
def foo():
  a = xxx  # Still aborts...
  if False:
    xxx = 2
```
This is particularly puzzling in long functions when someone accidentally
adds a local variable which matches name of a global variable used in other part
of the same function:
```
def value():
  ...

def foo():
  ...  # A lot of code
  value(1)  # Surprise! UnboundLocalError
  ...  # Yet more code
  for value in data:
    ...
```

Once you've lost some time debugging the issue, you can be overcome
for global variables by declaring their names as `global`
before first use:
```
def foo():
  global xxx
  a = xxx
  xxx = 2
```
But there are no magic keywords forvariables from non-global outer scopes so they
are essentially unwritable from nested scopes i.e. closures:
```
def foo():
  xxx = 1
  def bar():
    xxx = 2  # No way to modify xxx...
```

The only available "solution" is to wrap the variable into a fake 1-element array
(whaat?!):
```
def foo():
  xxx = [1]
  def bar():
    xxx[0] = 2
```

## Relative imports are unusable

Relative imports (`from .xxx.yyy import mymod`) have many weird limitations
e.g. they will not allow you to import module from parent folder and
they will seize work in main script
```
ModuleNotFoundError: No module named '__main__.xxx'; '__main__' is not a package
```

A workaround is to use extremely ugly `sys.path` hackery:
```
import sys
import os.path
sys.path.append(os.path.join(os.path.dirname(__file__), 'xxx', 'yyy'))
```

Search for "python relative imports" on stackoverflow to see some really clumsy Python code
(e.g. [here](https://stackoverflow.com/questions/279237/import-a-module-from-a-relative-path)
or [here](https://stackoverflow.com/questions/1918539/can-anyone-explain-pythons-relative-imports)).
Also see [When are circular imports fatal?](https://datagrok.org/python/circularimports/)
for more weird limitations of relative imports with respect to circular dependencies.

# Performance

## Automatic optimization is hard

It's very hard to automatically optimize Python code
because there are far too many ways
in which program may change execution environment e.g.
```
  for re in regexes:
    ...
```
Existing optimizers (e.g. pypy) have to rely on idioms and heuristics.

# Infrastructure

## Different conventions on OSes

Windows and Linux use different naming convention for Python executables
(`python` on Windows, `python2`/`python3` on Linux).

## Debugger is slow

Python debugging is super-slow (few orders of magnitude slower than
interpretation).

## No static analyzer

Already mentioned in [Zero static checking](#zero-static-checking).

## Unable to break on pass statement

Python debugger will [ignore breakpoints set on pass statements](https://stackoverflow.com/a/47626134/2170527).
Thus poor-man's conditional breakpoints like
```
if x > 0:
  pass
```
will silently fail to work, leaving a false impression that condition is always false.
