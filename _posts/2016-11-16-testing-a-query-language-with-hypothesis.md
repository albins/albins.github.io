---
layout: post
title: Testing a Small Query Language in Python with Hypothesis
---
![An experimental setup with three beakers, illustration from a book](/resources/chemistry_general_and.jpg)

_This entry was intended to be cross-posted to
[the CERN Databases blog](http://db-blog.web.cern.ch/), but is currently pending review. Consider it a pre-release version_.

[Hypothesis](http://hypothesis.works/) is an implementation of
Property-based testing for Python, similar to 
[QuickCheck](https://en.wikipedia.org/wiki/QuickCheck) in Haskell/Erlang
and [test.check](https://github.com/clojure/test.check) in Clojure
(among others). Basically, it allows the programmer to formulate
invariants about their programs, and have an automated system attempt to
generate counter-examples that invalidates them.

## A Small Query Language

During my internship at CERN, I am developing a small (partial) two-way
monitoring system to propagate alerts from filers to CERN's incident
management system. In the course of developing this monitor, I decided
to invent a very minimal query/filtering language for logging events. It
maps directly against Python objects using regular expressions
(basically: "does object x have a property y matching regex z"?). The
following is its grammar (written for
[the Grako parser-generator](https://pypi.python.org/pypi/grako)):

```ebnf
start = expression ;

expression
        =
        '(' expression ')' binary_operator '(' expression ')'
        | unary_operator '(' expression ')'
        | statement
       ;

binary_operator = 'AND' | 'OR';
unary_operator = 'NOT';
statement = field ':' "'" regex "'";
field =
      /[0-9A-Za-z_]+/
      ;

regex
    = /([^'])*/
    ;
```

An example (from a test configuration file) could be
`event_type:'disk.failed'` (disk failures) or
`(source_type:'(?i)Aggregate') AND (NOT(source_name:'aggr0'))` (log
events from aggregates, but not `aggr0`).

The following invariants should hold, where `q` is any valid query:

- `NOT(NOT(q))` ≡ `q`
- `(q) AND (q)` ≡ `q`
- `(q) OR (q)` ≡ `q`
- `(q) OR (NOT(q))` is always `True`

In addition, the following properties should also hold:

- `key:'value'` matches every object containing a property `key` with
  exact value `value` for any valid values of `key` and value (that is,
  valid Python variable names for `key` and more or less any string for
  `value`)
- `key:''` matches every object that has an attribute `key` regardless
  of its value

## Generating Examples

There are several types of inputs we need to generate to test the
system. Let's break them down:

- objects with various fields
- regular expressions
- valid statements
- valid queries

Let's start from the top. As Python is a dynamic language, we can do
_crazy_ things, like dynamically generating objects from
dictionaries. The following is a fairly common hack:

``` python
class objectview(object):
    def __init__(self, d):
        self.__dict__ = d

    def __repr__(self):
        return str(self.__dict__)

    def __str__(self):
        return str(self.__dict__)
```

This allows us to instantiate an object with (almost) arbitrary fields:

``` python
cat = objectview({'colour': 'red', 'fav_food_barcode': '1941230190'})
>>> cat.colour
'red'
>>> cat.fav_food_barcode
'1941230190'
```

Given this, we can just generate valid objects using the `@composite`
decorator in Hypothesis:

``` python
@composite
def objects(draw):
    ds = draw(dictionaries(keys=valid_properties,
                           values=valid_values,
                           min_size=1))

    return objectview(ds)
```

Generating valid values is much simpler:

``` python
valid_values = text()
```

Any text string is a valid string value. Of course! Properties are a bit
trickier though:

``` python
valid_properties = (characters(max_codepoint=91,
                               whitelist_categories=["Ll", "Lu", "Nd"])
                    .filter(lambda s: not s[0].isdigit()))
```

Variable names can't start with a number, and has to be basically mostly
ASCII, so we slightly modify and filter the `characters` strategy.


Statements can be generated in much the same way, using composite strategies:

``` python
@composite
def statements(draw):
    # any valid key followed by a valid regex
    key = draw(valid_properties)
    regex = draw(regexes)

    return u"{key}:'{regex}'".format(key=key, regex=regex)
```

However, how do we produce regular expressions? Let's start with some
valid candidates:

``` python
regex_string_candidates = characters(blacklist_characters=[u'?', u'\\', u"'"])
```

Then we can generate regular expressions using Hypothesis' back-tracking
functionality through `assume()`, which causes it to discard bad
examples (in this instance `is_valid_regex()` simply tries to compile
the string as a Python regular expression, and returns False if it
fails):

``` python
@composite
def regex_strings(draw):
    maybe_regex = draw(regex_string_candidates)
    assume(is_valid_regex(maybe_regex))
    return maybe_regex
```

But we can also use recursive generation strategies to produce more complex regular expressions:

```python
regexes = recursive(regex_strings(), lambda subexps:
                    # match one or more
                    subexps.map(lambda re: u"({re})+".format(re=re)) |

                    # match zero or more
                    subexps.map(lambda re: u"({re})*".format(re=re)) |

                    # Append "match any following"
                    subexps.map(lambda re: u"{re}.*".format(re=re)) |

                    # Prepend "match any following"
                    subexps.map(lambda re: u".*{re}".format(re=re)) |

                    # Prepend start of string
                    subexps.map(lambda re: u"^{re}".format(re=re)) |

                    # Append end of string
                    subexps.map(lambda re: u"{re}$".format(re=re)) |

                    # Append escaped backslash
                    subexps.map(lambda re: u"{re}\\\\".format(re=re)) |

                    # Append escaped parenthesis
                    subexps.map(lambda re: u"{re}\(".format(re=re)) |

                    # Append dot
                    subexps.map(lambda re: u"{re}.".format(re=re)) |

                    # Match zero or one
                    subexps.map(lambda re: u"({re})?".format(re=re)) |

                    # Match five to six occurrences
                    subexps.map(lambda re: (u"({re})"
                                            .format(re=re)) + u"{5,6}") |

                    # concatenate two regexes
                    tuples(subexps, subexps).map(lambda res: u"%s%s" % res) |

                    # OR two regexes
                    tuples(subexps, subexps).map(lambda res: u"%s|%s" % res))
```

The same strategy also works for the highly recursive structure of the query language:

``` python
queries = recursive(statements(),
                    lambda subqueries:
                    subqueries.map(negated_query) |
                    tuples(subqueries, subqueries).map(ored_queries) |
                    tuples(subqueries, subqueries).map(anded_queries))
```

Read as: "a valid query is any statement, or a any valid query negated,
or two valid queries AND:ed or OR:ed".

## Making Assertions

To finally assert properties, we assert things similarly to how we would
in normal unit tests. For example, let's verify that the empty regular
expression matches anything:

```python
@given(target=objects(), key=valid_properties)
def test_query_for_empty_regex_always_matches(target, key):
    q = "{key}:''".format(key=key)
    assert query.matches_object(q, target)
```

Hypothesis immediately finds a counter-example:
```
>       assert query.matches_object(q, target)
E       assert False
E        +  where False = <function matches_object at 0x7fa76dc6f5f0>("A:''", {u'B': u''})
E        +    where <function matches_object at 0x7fa76dc6f5f0> = query.matches_object

key        = 'A'
q          = "A:''"
target     = {u'B': u''}

syncd/eql/test/test_hypothesis.py:188: AssertionError
---------- Hypothesis ---------
Falsifying example: test_query_for_empty_regex_always_matches(target={u'B': u''}, key=u'A')
```

An object which doesn't have the specified property will not match the
query, even if the query is looking for the empty string. Ok, so that's
a bad example depending on how we want to treat this edge-case. If we
really did want the empty regular expression to match even objects which
does not have their keys, this would have been a proper bug in the
implementation. However, it makes more sense to require the object to
_have_ the property checked for, and so this is a bad
counter-example. We can exclude it by adding `assume(hasattr(target,
key))` to the test, causing it to back-track on any examples where the
target object does not have the key:

```python
@given(target=objects(), key=valid_properties)
def test_query_for_empty_regex_always_matches(target, key):
    assume(hasattr(target, key))

    q = "{key}:''".format(key=key)

    assert query.matches_object(q, target)
```

And now, the test passes.

_The image is from "Chemistry: general, medical and pharmaceutical..." from 1894, [courtesy of the Internet Archive Book Images](https://www.flickr.com/photos/internetarchivebookimages/14781741274/)_
