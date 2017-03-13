# django_mkdocs

Example project setup from the blog post about integrading mkdocs with Django.

**How to run:**

* Requires python version `3.4` or `3.5`
* `pip install -r requirements.txt` in a proper venv
* `mkdocs build`
* `python manage.py migrate`
* `python manage.py createsuperuser`
* `python manage.py runserver`
* Open `http://localhost:8000/docs/`
* Login with the created superuser.
* Profit!
