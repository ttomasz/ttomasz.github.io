---
layout: post
title: "Programming alternatives to in-band error signalling"
date: 2025-03-02 16:42:50 +0100
author: tomasz
tags: data programming
---

When writing code programmers often seem to be tempted to assign special meaning to certain values e.g. when reading temperature sensor values "9999" will mean that something went wrong and accurate reading could not be acquired. This can lead to many problems down the line and I hope I can present some alternatives.

I haven't found a better way to name this problem so I will use one that I saw once on a forum: "in-band error signalling".

# What does it mean "in-band signalling"?

[In-band signalling](https://en.wikipedia.org/wiki/In-band_signaling) is a term used in telecommunications when control information is sent over the same band/channel as data.

# Let's make up some example to illustrate the point

In our case we can imagine for the sake of example that we are writing a function that counts something. Logically when we count something the result will be zero or more, so our function will return integer with the number of things. Let's start with the function signature:
```python
def count_thingies() -> int:
    ...  # imagine some code here that does the counting and stores the count in variable called result
    return result
```

As with all things in life sometimes something can go wrong. Our example is very abstract but let's say sometimes we can't access the things we are supposed to count.

One could get an idea that with very little code changes we could handle this situation by returning e.g. -1 as the result. It's clear that -1 is not a valid value in this case so what could go wrong.

However these kinds of decisions usually are not properly documented or get lost along the way and then someone (usually other than the author) is stuck debugging weird values in the data. If you just look at the raw values it may be quite obvious that something is going on with some values but if these results get aggregated (like sum([12, 43, 2, 0, -1, -1, -1])) then it may be difficult to pick up immediately... and every time it's a bit of extra work for someone else which means hidden costs of using your software increase.

# How about Null?

Ok, so magic numbers are bad. Maybe let's use a special value for "lack of value". There are many types of those e.g. NaN (Not a number), NULL, Nil, None, missing, etc. For the sake of simplicity we'll assume they are all the same and just use `None` since our code examples are in Python.

Our function's signature would change to:
```python
from typing import Optional

def count_thingies() -> Optional[int]:
    ...
```
or if we use newer Python version:
```python
def count_thingies() -> int | None:
    ...
```

These type hints tell us that the function will return either an integer or a None. User can then decide what do they want to do with the None/empty values but those are either more likely to be picked up on some data quality checks (e.g. "more than half of the values are missing" or "we expected all rows to have values but some don't") or will be handled gracefully on aggregations (most aggregation functions will ignore empty values or throw an error so at least the user will know something is wrong).

This works well for data exchange but I wanted to mention two techniques that can be used in the code which can be useful for internal methods within a framework or in library functions.

When something goes wrong we could raise (or throw depending on given language's preferred wording) an exception or return a special result object that wraps the value in an object that contains information about success or failure. Some languages consider one approach to be better or more natural than the other. Sometimes which of these is used will depend on type of error. Maybe if DNS lookup failed raise an exception but if we got 401 HTTP Status Code return result marked as failure.

# Exceptions

One way to signal that something went wrong is to raise an exception. Exceptions are special types of action that stops the program to check how to handle it. Programmer can specify in an exception handler what should be done with the exception (retry, ignore, exit program, raise the exception further to check for another handler, etc). If there is no handler for the exception then the program is stopped completely.

In Python you can just use base Exception class or one of [the built-in exception classes](https://docs.python.org/3/library/exceptions.html) that inherit from it (like TypeError, ValueError, FileNotFound) but it's often recommended to create your own types of exceptions so it's easier for downstream users to handle them since they can react to specific exception types instead of trying to fish error type from error message.

In Python creating your own exception type is as easy as:
```python
class MyExceptionType(Exception):  # inherit from base Exception class
    """Explain what this type of exception means"""

```
then we can simply use it in our code:
```python
def count_thingies() -> int:
    # ...
    if something_wrong:
        raise MyExceptionType("Custom message")
    return result
```
and someone using our library could catch it like this:
```python
try:
    number_of_thingies = count_thingies()
except MyExceptionType as e:
    # do something here if this type of exception happens
except Exception as e:
    # in case of any other exception do something else
```

# Result types

Another way is to create a wrapper object that we'll return. This wrapper object - called [Result type](https://en.wikipedia.org/wiki/Result_type) - will contain the status (success/failure) and the value. This approach is usually used in functional programming languages but can be used in other languages like Python as well.

If you have ever used Python library "requests" you are probably familiar with their `Response` class. When we make an HTTP request and we get some response back from the server the library gives us this data in an object containing the data, HTTP headers, HTTP status code. This is a kind Result type too. We can tell if the request was a success or not and act accordingly:
```python
import requests

response = requests.get("https://httpbin.org/status/200")

if response.ok: # shortcut for: response.status_code >= 200 and response.status_code < 300
    # yay success
else:
    # oh no something went wrong
```

One can make a crude custom class with status and value fields:
```python
from typing import Any

class Result:
    def __init__(self, success: bool, value: Any) -> None:
        self.success = success
        self.value = value
```
but there is also a way to utilize types a bit more. We could create separate classes for success and failure and then check which one we got. We can even use [Generics](https://peps.python.org/pep-0636/) so we keep information about type of the value that is returned in case of success:
```python
from random import random
from typing import Generic, TypeVar

ValueType = TypeVar("ValueType")

class Success(Generic[ValueType]):
    def __init__(self, value: ValueType) -> None:
        self.value: ValueType = value

class Err:
    def __init__(self, message: str) -> None:
        self.message = message

def count_thingies() -> Success[int] | Err:
    # ...
    if random() < 0.5:
        return Err("helpful error message")
    else:
        return Success(value=42)

result = count_thingies()
if isinstance(result, Success):
    print(result.value)
else:    
    print(result.message)
```

There is an [interesting library](https://github.com/rustedpy/result) in Python that tries to implement this kind of interface [but it's unmaintained](https://github.com/rustedpy/result/issues/201) and there are some concerns that Python's typing system is not advanced enough to implement this kind of pattern with full type safety.

# Ok, so which do I use?

Well, that depends. In general I believe that one should be wary of adding additional layers of indirection and abstraction as they increase mental load when trying to understand and use some piece of code but these patterns can be very helpful when properly applied.

Most importantly... don't just return some valid number to indicate error and hope that data users will know what it means.
