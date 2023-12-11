# aiohub

[![Test & Checks](https://github.com/4383/aiohub/actions/workflows/main.yml/badge.svg)](https://github.com/4383/aiohub/actions/workflows/main.yml)
![PyPI](https://img.shields.io/pypi/v/aiohub.svg)
![PyPI - Python Version](https://img.shields.io/pypi/pyversions/aiohub.svg)
![PyPI - Status](https://img.shields.io/pypi/status/aiohub.svg)
[![Downloads](https://pepy.tech/badge/aiohub)](https://pepy.tech/project/aiohub)
[![Downloads](https://pepy.tech/badge/aiohub/month)](https://pepy.tech/project/aiohub/month)

Aiohub is a library that aim to make possible to run eventlet and asyncio
in the same process.

The goal of aiohub is to allow migrating from eventlet to asyncio in a gradual
manner.

We think that it should be possible to:

* Run eventlet and asyncio in the same thread.
* Allow asyncio and eventlet to interact: eventlet code can use asyncio-based
  libraries, asyncio-based code can get results out of eventlet.
* Migrate from eventlet to asyncio in smooth way.

Aiohub is designed to answer these 3 criterias.

## Install

```
$ pip install aiohub
```

## Usage

### Part 1: Implementing asyncio/eventlet interoperability

There are three different parts involved in integrating eventlet and asyncio
for purposes.

#### 1. Create a hub that runs on asyncio

Like many networking frameworks, eventlet has pluggable event loops, in this
case called a "hub". Typically hubs wrap system APIs like `select()` and
`epoll()`, but there also used to be a hub that ran on Twisted.

Aiohub aim to create a hub that runs on the top of the asyncio event loop.

By using this hub, eventlet and asyncio code can run in the same process and
the same thread, but they would still have difficulties talking to each other.
This latter requirement requires additional work, as covered by the next two
items.

#### 2. Calling `async def` functions from eventlet

The goal is to allow something like this:

```python
import aiohttp
from aiohub import future_to_greenlet

async def get_url_body(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

def eventlet_code():
    green_thread = future_to_greenlet(get_url_body("https://example.com"))
    return green_thread.wait()
```

The code would presumably be similar to
https://github.com/gfmio/asyncio-gevent/blob/main/asyncio_gevent/future_to_greenlet.py

#### 3. Calling eventlet code from asyncio

The goal is to allow something like this:

```python
from urllib.request import urlopen
from eventlet import spawn
from aiohub import greenlet_to_future

def get_url_body(url):
    # Looks blocking, but actually isn't
    return urlopen(url).read()

# This would likely be common pattern, so could be implemented as decorator...
async def asyncio_code():
    greenlet = eventlet.spawn(get_url_body, "https://example.com")
    future = greenlet_to_future(greenlet)
    return await future
```

The code would presumably be similar to
https://github.com/gfmio/asyncio-gevent/blob/main/asyncio_gevent/future_to_greenlet.py

### Part 2: How a port would work on a technical level

#### Porting a library

1. Usage of eventlet-based APIs would be replaced with usage of asyncio APIs.
   For example, `urllib` or `requests` might be replaced with
   [`aiohttp`](https://docs.aiohttp.org/en/stable/).
   The interoperability above can be used to make sure this continues to work
   with eventlet-based APIs.
2. Over time, APIs would need be migrated to be `async` function, but in the
   intermediate time frame a standard `def` can still be used, again using the
   interoperability layer above.
3. Eventually all "blocking" APIs have been removed, at which point everything
   can be switched to `async def` and `await`, including external API, and the
   library will no longer depend on eventlet.

#### Porting an application

An application would need to install the asyncio hub before kicking off
eventlet. Beyond that porting would be the same as a library.

Once all libraries are purely asyncio-based, eventlet usage can be removed and
an asyncio loop run instead.

## Short story of aiohub

The Openstack community searched a solution to migrate their code base from
eventlet to asyncio. This decision was motivated by:
- a low maintenance activity on eventlet leading to unsupported version of
  Python in eventlet.
- the willingness to migrate to a modern async backend
- the willingness to remain close to the Python stdlib

However, Openstack heavily relied on eventlet and it was not possible to
migrate all the Openstack in one shot, an intermediary steps was mandatory.
Aiohub was born.

Aiohub would allow migrating from eventlet to asyncio in a gradual manner both
within and across projects:

1. Within an OpenStack library, code could be a mixture of asyncio and
   eventlet code. This means migration doesn't have to be done in one stop,
   neither in libraries nor in the applications that depend on them.
2. Even when an OpenStack library fully migrates to asyncio, it will still be
   usable by anything that is still running on eventlet.

### Prior art

* Gevent has a similar model to eventlet.
  There exists an integration between gevent and asyncio that follows model
  proposed below: https://pypi.org/project/asyncio-gevent/
* Twisted can run on top of the asyncio event loop.
  Separately, it includes utilities for mapping its `Deferred` objects
  (similar to a JavaScript Promise) to the async/await model introduced in
  newer versions in Python 3, and in the opposite direction it added support
  for turning async/await functions into `Deferred`s.
  In an eventlet context, `GreenThread` would need a similar former of
  integration to Twisted's `Deferred`.
