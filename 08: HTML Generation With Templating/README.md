08: HTML Generation With Templating
Most web frameworks don't embed HTML in programming code. Instead, they pass data into a templating system. In this step we look at the basics of using HTML templates in Pyramid.

Background
Ouch. We have been making our own Response and filling the response body with HTML. You usually won't embed an HTML string directly in Python, but instead will use a templating language.

Pyramid doesn't mandate a particular database system, form library, and so on. It encourages replaceability. This applies equally to templating, which is fortunate: developers have strong views about template languages. As of Pyramid 1.5a2, Pyramid doesn't even bundle a template language!

It does, however, have strong ties to Jinja2, Mako, and Chameleon. In this step we see how to add pyramid_chameleon to your project, then change your views to use templating.

Objectives
Enable the pyramid_chameleon Pyramid add-on.

Generate HTML from template files.

Connect the templates as "renderers" for view code.

Change the view code to simply return data.

Steps
Let's begin by using the previous package as a starting point for a new project:

cd ..; cp -r views templating; cd templating
This step depends on pyramid_chameleon, so add it as a dependency in templating/setup.py:

from setuptools import setup

# List of dependencies installed via `pip install -e .`
# by virtue of the Setuptools `install_requires` value below.
requires = [
    'pyramid',
    'pyramid_chameleon',
    'waitress',
]

# List of dependencies installed via `pip install -e ".[dev]"`
# by virtue of the Setuptools `extras_require` value in the Python
# dictionary below.
dev_requires = [
    'pyramid_debugtoolbar',
    'pytest',
    'webtest',
]

setup(
    name='tutorial',
    install_requires=requires,
    extras_require={
        'dev': dev_requires,
    },
    entry_points={
        'paste.app_factory': [
            'main = tutorial:main'
        ],
    },
)
Now we can activate the development-mode distribution:

$VENV/bin/pip install -e .
We need to connect pyramid_chameleon as a renderer by making a call in the setup of templating/tutorial/__init__.py:

from pyramid.config import Configurator


def main(global_config, **settings):
    config = Configurator(settings=settings)
    config.include('pyramid_chameleon')
    config.add_route('home', '/')
    config.add_route('hello', '/howdy')
    config.scan('.views')
    return config.make_wsgi_app()
Our templating/tutorial/views.py no longer has HTML in it:

from pyramid.view import view_config


# First view, available at http://localhost:6543/
@view_config(route_name='home', renderer='home.pt')
def home(request):
    return {'name': 'Home View'}


# /howdy
@view_config(route_name='hello', renderer='home.pt')
def hello(request):
    return {'name': 'Hello View'}
Instead we have templating/tutorial/home.pt as a template:

<!DOCTYPE html>
<html lang="en">
<head>
    <title>Quick Tutorial: ${name}</title>
</head>
<body>
<h1>Hi ${name}</h1>
</body>
</html>
For convenience, change templating/development.ini to reload templates automatically with pyramid.reload_templates:

[app:main]
use = egg:tutorial
pyramid.reload_templates = true
pyramid.includes =
    pyramid_debugtoolbar

[server:main]
use = egg:waitress#main
listen = localhost:6543
Our unit tests in templating/tutorial/tests.py can focus on data:

import unittest

from pyramid import testing


class TutorialViewTests(unittest.TestCase):
    def setUp(self):
        self.config = testing.setUp()

    def tearDown(self):
        testing.tearDown()

    def test_home(self):
        from .views import home

        request = testing.DummyRequest()
        response = home(request)
        # Our view now returns data
        self.assertEqual('Home View', response['name'])

    def test_hello(self):
        from .views import hello

        request = testing.DummyRequest()
        response = hello(request)
        # Our view now returns data
        self.assertEqual('Hello View', response['name'])


class TutorialFunctionalTests(unittest.TestCase):
    def setUp(self):
        from tutorial import main
        app = main({})
        from webtest import TestApp

        self.testapp = TestApp(app)

    def test_home(self):
        res = self.testapp.get('/', status=200)
        self.assertIn(b'<h1>Hi Home View', res.body)

    def test_hello(self):
        res = self.testapp.get('/howdy', status=200)
        self.assertIn(b'<h1>Hi Hello View', res.body)
Now run the tests:

$VENV/bin/pytest tutorial/tests.py -q
....
4 passed in 0.46 seconds
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/ and http://localhost:6543/howdy in your browser.

Analysis
Ahh, that looks better. We have a view that is focused on Python code. Our @view_config decorator specifies a renderer that points to our template file. Our view then simply returns data which is then supplied to our template. Note that we used the same template for both views.

Note the effect on testing. We can focus on having a data-oriented contract with our view code.

Templates, Debugging Templates, and Available Add-On Template System Bindings.
