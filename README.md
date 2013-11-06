mymock
======

This is a lib a wrote a few years ago, when I was just getting started with TDD. Nowdays, I don't really use it anymore, because I find that I write tests against my test doubles too :) But maybe someone else might find it useful...

Introduction
============

mymock is a quickly hacked mocking library built on top of [http://www.voidspace.org.uk/python/mock/ the python's own mocking lib], and heavily inspired on [http://rspec.info/ ruby's rspec]. The purpose of the lib is to provide an easier way to setup "expectations" and to validate them.

With Python's mock lib, one might write something like this:

```python
import unittest
import mock
import mymock

class MyRepository(object):
    def __init__(self, db):
        self.db = db

    def save(self, something):
        foo = something.get_id()
        self.db.save(something)


class RepositoryTest(unittest.TestCase):
    def test_writes_to_db(self):
        db = mock.Mock()
        something = mock.Mock()
        repo = MyRepository(db)
        repo.save(something)
        # assert that db's "save" was called with "something" as argument
        self.assertTrue(db.save.called, "db's save method wasn't called")
        calls = [m for m in db.method_calls if m[0] == "save"]
        self.assertEquals(calls[0][1][0], something, "expected db.save to be called with 'something'")

```

So, first thing we notice is that it's really bloated and doesn't express the intention of the test clearly.
Second, asserting on the arguments to calls is harder than it should be.
Mymock implements a DSL for describing what interactions are expected between objects and
automatically asserts them after the code under test was run.
The previous example using mymock becomes:

```python
class RespositoryTest2(unittest.TestCase):
    __metaclass__ = mymock.class_with_mock

    def test_writes_to_db(self):
        something = mymock.MyMock(mock_name="something")
        db = mymock.MyMock(mock_name="db")
        db.should_receive("save").with_(something)
        repo = MyRepository(db)
        repo.save(something)
```
That's it. The first three lines setup the expectations, documenting at the same time what is expected to happen.
The last two lines run the code under test. Verifications are run automatically.

Let's say the save method isn't implemented to do what's expected, the result message would be:
```python
    AssertionError: expected 'db' to receive message save
```

Yet, expectations are not the only reason to use mocks. An object might be queried during the interaction
with the code under test. Mocks can be configured to answer to those queries but it's better not to assert that those queries were made. Following several testing frameworks terminology, mymock uses the concept of "stub" to differentiate this kind of behavior definition from defining an expectation.

For example: let's say our repository, in this particular implementation of "save", needs to send the message "get_id" to
the object it's going to save. We can use a stub to define the answer to the message:

```python

class RespositoryTest3(unittest.TestCase):
    __metaclass__ = mymock.class_with_mock

    def test_writes_to_db(self):
        something = mymock.MyMock(mock_name="something")
        something.stub("get_id").and_return(1)
        db = mymock.MyMock(mock_name="db")
        db.should_receive("save").with_(something)
        repo = MyRepository(db)
        repo.save(something)

```

We don't always can get away with comparing the exact values of things. One example would be when we want to
setup an expectation that an object receives a message with another object of a particular instance:

```python
class Foo:
    pass


class Blah:
    pass


def code_under_test(m):
    m.receive_foo_object(Blah())


class ReceiveInstanceTest(unittest.TestCase):
    __metaclass__ = mymock.class_with_mock

    def test_receive_foo(self):
        m = mymock.MyMock(mock_name="expects class Foo")
        m.should_receive("receive_foo_object").with_(mymock.instanceof(Foo))
        code_under_test(m)

```

in example above, the expectation will fail until we change the return type to Foo.

But, what about checking just some of the received values?
```python
def calls_with_args(m):
    m.receive_foo_object(Foo(), "doesn't matter", 10)

class ReceiveInstanceAndOthersTest(unittest.TestCase):
    __metaclass__ = mymock.class_with_mock
    def test_receive(self):
        m = mymock.MyMock(mock_name="expects class Foo")
        m.should_receive("receive_foo_object").with_(mymock.instanceof(Foo), mymock.anything(), 10)
        calls_with_args(m)
```
in the example above, the expectation will pass whatever the second argument to receive_foo_object is.

What about special cases when we want to check more complex arguments?
The implementation of anything and instanceof is simple: they define their __eq__ method to return true
according to what's expected. So let's say we want to check that we are handed an object whose "name" attribute
should be "A Name"

```python
class ObjectWithName(object):
    def __init__(self, name):
        self.name = name
    def __eq__(self, other):
        return self.name == other.name

class Person(object):
    def __init__(self, name):
        self.name = name

def calls_with_name(m):
    m.receive_foo_object(Person("A Name"))

class ReceiveAttributeTest(unittest.TestCase):
    __metaclass__ = mymock.class_with_mock
    def test_receive(self):
        m = mymock.MyMock(mock_name="expects class Foo")
        m.should_receive("receive_foo_object").with_(ObjectWithName("A Name"))
        calls_with_name(m)

```

Returning with intelligence
===========================

Sometimes mock and stubs just need more intelligence. If we want to answer a message according to the arguments passed in it, we can use {{{return_according_to}}}:
```python
def queries_to_things(o):
    r1 = o.get_id(1)
    r2 = o.get_id(2)
    return r1, r2

class QueryTest(unittest.TestCase):
    __metaclass__ = mymock.class_with_mock
    def test_query(self):
        repo = mymock.MyMock(mock_name="repo")
        resp_map = {
            1 : 2,
            2 : 3
        }
        repo.stub("get_id").and_return_according_to(resp_map.get)
        response = queries_to_things(repo)
        assert response == (2,3)
```

In the end, if the object is too complicated, we could implement an actual class that can handle complex interactions.

They are still mocks!
=====================

Because !MyMock objects inherit from mock.Mock, you can still use all their features, such as side_effects definitions, list of calls, etc.
