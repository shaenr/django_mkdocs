# Integrading a password-protected MkDocs in Django

It's about time to that big `README.md` file from your project in something that supports a nice-looking markdown-driven documentaion, such as [MkDocs](http://www.mkdocs.org/)

But you have the following requirements:

* You want to serve it as part of your Django project. This means - being self-contained.
* And also, **you want it to be password-protected**, using your existing users in the system.

In this article, we are going to do exactly that.

## What we want to achieve?

We want to open our project at `/docs`, be redirect to a login page, and after that, see the documentation, rendered at `/docs`.

## The setup

First, we are going to setup our django project and create `docs` app.

```bash
django-admin startproject django_mkdocs
cd django_mkdocs
python manage.py startapp docs
```

And since we are going to serve the documentation as a static content from our docs app:

```bash
mkdir docs/static
```

Then, we need to install `MkDocs`:

```bash
pip install mkdocs
```

and start a new `MkDocs` project:

```bash
mkdocs new mkdocs
```

This will create a new documentation project in `mkdocs` folder. **This is where we are going to store our documentation markdown files.**

We need to do some moving around, since we want to end up with `mkdocs.yml` at the same directory level as `manage.py`:

```bash
mv mkdocs/docs/index.md mkdocs/
mv mkdocs/mkdocs.yml .
rm -r mkdocs/docs
```

We need to end up with the following dir structure:

```bash
.
├── django_mkdocs
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── docs
│   ├── admin.py
│   ├── apps.py
│   ├── __init__.py
│   ├── migrations
│   │   └── __init__.py
│   ├── static
│   ├── models.py
│   ├── tests.py
│   ├── urls.py
│   └── views.py
├── manage.py
├── mkdocs
│   └── index.md
└── mkdocs.yml
```

Before we do anything else, we are going to change our `mkdocs.yml` configuration to match the structure above.

## MkDocs Configuration

We want to achieve two things:

* Say that our documentation is going to be stored in `mkdocs` folder.
* Say that our build is going to be stored in `docs/static/mkdocs_build` folder. **Django will be serving from this folder.**

Of course, those folder names can be changed to whatever you like.

We end up with the following `mkdocs.yml` file:

```
site_name: My Docs

docs_dir: 'mkdocs'
site_dir: 'docs/static/mkdocs_build'

pages:
  - Home: index.md
```

Now, if we run the test mkdocs server:

```bash
mkdocs serve
```

We can open `http://localhost:8000` and see our documentation there.

Finally, lets build our documentation:

```bash
mkdocs build
```

You can now open `docs/static/mkdocs_build` and explore it. Open `index.html` in your browser. This is a neat staic web page with our documentation.

## Making Django serve MkDocs

Now, the interesting part begins.

### Bootstraping

We want to serve our documentation from `/docs` so the first thing we are going to do is redirect `/docs` to `docs/urls.py`.

In `django_mkdocs/urls.py` change to the following:

```python
from django.conf.urls import url, include
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^docs/', include('docs.urls'))
]
```

Now, lets create `docs/urls.py` and `docs/views.py` with some default values:

```python
"""
docs/urls.py
"""
from django.conf.urls import url

from .views import serve_docs


urlpatterns = [
    url(r'^$', serve_docs),
]
```

and

```python
"""
docs/views.py
"""
from django.http import HttpResponse


def serve_docs(request):
    return HttpResponse('Docs are going to be served here')
```

Now, if we run our Django, we see the response at `http://localhost:8000/docs/`

```bash
python manage.py runserver
```

### Url configuration

Now, we want to catch every url of the format: `/docs/*` and try to find the given path inside `mkdocs_build`

Lets start with the regular expression that will match everything. We will use `.*` which means "whatever, 0, 1 or more times"

```python
"""
docs/urls.py
"""
from django.conf.urls import url


from .views import serve_docs


urlpatterns = [
    url(r'^(?P<path>.*)$', serve_docs),
]
```

Now in the view, we will receive a key-word argument called `path`:

```python
"""
docs/views.py
"""
from django.http import HttpResponse


def serve_docs(request, path):
    return HttpResponse(path)
```

If we do some testing, we will get the following values:

* `/docs/` -> empty string
* `/docs/index.html` -> `index.html`
* `/docs/about/` -> `about/`
* `/docs/about` -> `about`


### Serving the static files

Now, we are almost done. We need to get tha `path` and try to serve that file from `docs/static/mkdocs_build` directory. This is basically static serving from Django.

We will start with adding `DOCS_DIR` settings in our `settings.py` file, so we can easily concatenate file paths after that.

```
"""
django_mkdocs/settings.py
"""
# .. rest of the settings

DOCS_DIR = os.path.join(BASE_DIR, 'docs/static/mkdocs_build')
```

Since we are going to serve static files, we can take one of the two approaches:

1. Implement it ourselves.
2. Reuse Django's [static serving](https://docs.djangoproject.com/en/1.10/ref/contrib/staticfiles/)
3. Serve from a CDN / S3 / use Whitenoise.

Option 1 is good for education, option 3 is more efficient, but for this article, we will take option 2, since we can easily achieve what we want.

Since we need to provide the correct path to the desired file, we need to know the so-called **namespace** in our `docs/static` folder - `mkdocs_build/`

We will take that using `os.path.basename`:

```python
"""
django_mkdocs/settings.py
"""
# .. rest of the settings

DOCS_DIR = os.path.join(BASE_DIR, 'docs/static/mkdocs_build')
DOCS_STATIC_NAMESPACE = os.path.basename(DOCS_DIR)
```

Now, it's time for `django.contrib.staticfiles.views.serve`:

```python
"""
docs/views.py
"""
from django.conf import settings

from django.contrib.staticfiles.views import serve

def serve_docs(request, path):
    path = os.path.join(settings.DOCS_STATIC_NAMESPACE, path)

    return serve(request, path)
```

Now if we fire up our server and open `http://localhost:8000/docs/index.html` we should see the index page.

But we want to be even better - opening `http://localhost:8000/docs/` should also return the index page.


### Appending `index.html` to our path

Now, if we inspect the structure of `mkdocs_build` and add few more pages, we will see that there's always `index.html` for each page.

We can take advantage of that knowledge in our view:


```python
"""
docs/views.py
"""
import os

from django.conf import settings
from django.contrib.staticfiles.views import serve

def serve_docs(request, path):
    docs_path = os.path.join(settings.DOCS_DIR, path)

    if os.path.isdir(docs_path):
        path = os.path.join(path, 'index.html')

    path = os.path.join(settings.DOCS_STATIC_NAMESPACE, path)

    return serve(request, path)
```

Now opening `http://localhost:8000/docs/` opens the index page of the documentation. And we are done.


### Extra credit - reading `mkdocs.yml` in `settings.py`

Now, we have this `mkdocs_build` string defined both in `settings.py` and `mkdocs.yml`. We can dry things up with the following code:

```bash
pip install PyYAML
```

And change `settings.py` to look like that:

```python
"""
django_mkdocs/settings.py
"""
import yaml

# ... some settings

MKDOCS_CONFIG = os.path.join(BASE_DIR, 'mkdocs.yml')
DOCS_DIR = ''
DOCS_STATIC_NAMESPACE = ''

with open(MKDOCS_CONFIG, 'r') as f:
    DOCS_DIR = yaml.load(f, Loader=yaml.Loader)['site_dir']
    DOCS_STATIC_NAMESPACE = os.path.basename(DOCS_DIR)
```

And now, we are ready.

## Making the documentation password-protected

Now, for the final part, we can easily reuse Django's auth system and just add the neat `login_required` decorator:

```python
"""
docs/views.py
"""
import os

from django.conf import settings

from django.contrib.auth.decorators import login_required
from django.contrib.staticfiles.views import serve

@login_required
def serve_docs(request, path):
    docs_path = os.path.join(settings.DOCS_DIR, path)

    if os.path.isdir(docs_path):
        path = os.path.join(path, 'index.html')

    path = os.path.join(settings.DOCS_STATIC_NAMESPACE, path)

    return serve(request, path)
```

How you are going to handle this login is now up to you.

## Production settings

Now, if we want to push that to production, you will probably have `DEBUG = False`. **This will break our implementation**, since `django.contrib.staticfiles.views.serve` has a check about that.

If we want to have this served in production, we need to pass `insecure=True` as kwarg to `serve`:

```python
@login_required
def serve_docs(request, path):
    docs_path = os.path.join(settings.DOCS_DIR, path)

    if os.path.isdir(docs_path):
        path = os.path.join(path, 'index.html')

    path = os.path.join(settings.DOCS_STATIC_NAMESPACE, path)

    return serve(request, path, insecure=True)
```

### A security consideration

Now, if you have other static files, there's a big chance of having `collectstatic` as part of your deployment procedure.

This will also include the `mkdocs_build` folder **and everyone will have access to the documentation, using `STATIC_URL`.**

We can avoid putting our documentation in the `STATIC_ROOT` directory, by ignoring it when calling `collectstatic`:

```bash
python manage.py collectstatic -i mkdocs_build
```

## Overview

If you read the documentation about [`django.contrib.staticfiles.views.serve`](https://docs.djangoproject.com/en/1.10/ref/contrib/staticfiles/#django.contrib.staticfiles.views.serve) you will see the following warning:

> During development, if you use django.contrib.staticfiles, this will be done automatically by runserver when DEBUG is set to True (see django.contrib.staticfiles.views.serve()).

> This method is grossly inefficient and probably insecure, so it is unsuitable for production.

**Depending on your needs, this can be good enough.**

* About the **insecure** part, here is a good [StackOverflow thread](http://stackoverflow.com/questions/31097333/why-is-serving-static-files-insecure) about it.
* About the **performance** part, here is a random benchmark, done with [wrk](https://github.com/wg/wrk), on gunicorn with 2 workers:

```bash
./wrk -t2 -c10 -d30s http://localhost:8000/docs/
Running 30s test @ http://localhost:8000/docs/
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     7.13ms    2.81ms  41.73ms   85.02%
    Req/Sec   696.29    165.57     1.00k    69.22%
  35444 requests in 30.10s, 199.67MB read
  Socket errors: connect 10, read 0, write 0, timeout 0
Requests/sec:   1177.62
Transfer/sec:      6.63MB
```

But for the sake of performance, we will take a look at different approaches for solving this problem in our next articles.

For now, you can find the complete example project here.
