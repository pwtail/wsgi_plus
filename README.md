# WSGI+: enhancing WSGI with suspend/resume

### Application shouldn’t do anything that takes an undefined amount of time

An excerpt from gunicorn [docs](https://docs.gunicorn.org/en/stable/design.html?highlight=timeout#choosing-a-worker-type) says:

>The default synchronous workers assume that your application is resource-bound in terms of CPU and network bandwidth. Generally this means that your application shouldn’t do anything that takes an undefined amount of time. An example of something that takes an undefined amount of time is a request to the internet. At some point the external network will fail in such a way that clients will pile up on your servers. So, in this sense, any web application which makes outgoing requests to APIs will benefit from an asynchronous worker.

This RFC addresses exactly this issue: it allows for WSGI apps to not be bound by a strict timeot while still remaining blocking apps - mostly. This is done by suspending and resuming the application.

### WSGI spec

```python
def application(environ, start_response):
    start_response('200 OK', [('Content-type', 'text/plain')])
    yield b'Hi!\n'
```

The WSGI spec is simple and widely known. It provides a `start_response` callback to the app while the app returns an iterator over the response body.

A thing to note: *app can be a generator*. That means, it calls `start_response` during the first iteration. This feature is not documented in the spec, but every WSGI server supports it.

Now, here is the idea - since the app can be a generator.

- generator yields a special value, indicating it wants to be suspended
- we switch to processing other requests
- when the generator is ready, we continue iterating on it

### Implementation: yielding a Future

I think, the best special value is a `Future`. When application yields a future, we stop iterating on it. Then, when future has completed without exception, we continue the iteration.

```python
def application(environ, start_response):
    if 'needs suspending':
        yield Future()
    start_response('200 OK', [('Content-type', 'text/plain')])
    yield b'Hi!\n'
```

Simple, isn't it? And backwards-compatible too!

### Proof of concept

I've made a [proof of concept](https://github.com/pwtail/gunicorn/pull/1/files#diff-9818e6c0e3d6054dc383f77ce881ba79f8090a904fb3abd9892306f096e58319) for gunicorn, also providing an [app](https://github.com/pwtail/gunicorn/blob/wsgi-plus/examples/wsgi_plus.py) to test it.

The implementation is straightforward: submit generator to a thread pool, wait for a future, then add a callback on that future that submits it to the thread pool again.


### The goals and non-goals

One frequent usecase that is addressed here, is an application making http requests. Generally, you can solve this by increasing the timeout and the number of threads. However, if your application is some kind of proxy and  makes an http request every single time, then you are left with no choice other than wrapping it into an async app. This RFC gives you another choice.

An obvious way of making a web request while the app is suspended is using a dedicated async thread. However, this part is left up to application.

The non-goal is further extending of the WSGI spec. "Do one thing and do it well"

### Discussion

A disscussion [thread](https://github.com/pwtail/wsgi_plus/discussions/1) was created.
