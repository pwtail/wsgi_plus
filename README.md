# RFC: WSGI+

### Application shouldn’t do anything that takes an undefined amount of time

An excerpt from gunicorn [docs](https://docs.gunicorn.org/en/stable/design.html?highlight=timeout#choosing-a-worker-type) says:

>The default synchronous workers assume that your application is resource-bound in terms of CPU and network bandwidth. Generally this means that your application shouldn’t do anything that takes an undefined amount of time. An example of something that takes an undefined amount of time is a request to the internet. At some point the external network will fail in such a way that clients will pile up on your servers. So, in this sense, any web application which makes outgoing requests to APIs will benefit from an asynchronous worker.

This RFC addresses exactly this issue: it allows for WSGI apps to not be bound by a strict timeot while still remaining blocking apps - mostly.

### WSGI spec

```python
def application(environ, start_response):
    start_response('200 OK', [('Content-type', 'text/plain')])
    yield b'Hi!\n'
```

The WSGI spec is simple and widely known. It provides the `start_response` callback to the app while the app returns an iterator over the response body.

A thing to note however is that the app can be a generator - that means, it calls `start_response` during the first iteration. This feature is not documented in the WSGI spec, but every WSGI server supports this.

Now here is the idea: since the app already can be a generator - a lazy entity - maybe, adding the suspend/resume feature won't be that hard?

### The resume callback

It is so indeed. Here is my modified version:

```python
def application(environ, start_response, resume):
    if 'needs suspending':
        yield resume
    start_response('200 OK', [('Content-type', 'text/plain')])
    yield b'Hi!\n'

# somewhere in another thread:
resume()
```

The application takes one more argument - the `resume` callback. When the application wants to be suspended, it yields this callback. When it's ready to be iterated further, it calls `resume()`.

Obvously, an app can be suspended and resumed multiple times.

### Proof of concept

I've made a proof of concept for this feature for gunicorn, [here](https://github.com/pwtail/gunicorn/pull/1/files#diff-9818e6c0e3d6054dc383f77ce881ba79f8090a904fb3abd9892306f096e58319) is it. Also provided an [app](https://github.com/pwtail/gunicorn/blob/wsgi-plus/examples/wsgi_plus.py) to test it. It's clear it isn't hard to implement, but of course current PR is not ready to merge: for example, it lacks proper error propagation.

### The goals and non-goals

One frequent usecase that is addressed here, is an application making http requests. Generally, you can solve this by increasing the timeout and the number of threads. However, if your application is some kind of proxy and  makes an http request to a third-party service every time, then you are left with no choice other than wrapping it into an async app. This RFC solves this.

An obvious way of making a web request while the app is suspended is using a dedicated async thread. Anyway, this part is left up to application.

The non-goal is further extending of the WSGI spec. It is meant for deploying blocking Python web apps.

### Backwards compatibility

I think backwards compatibility won't be too much of a problem here. For some period of time, the user may be required to explicitly enable the new functionality (in `gunicorn.conf` for example), then it may become a default some time.

A disscussion [thread](https://github.com/pwtail/wsgi_plus/discussions/1) was created.