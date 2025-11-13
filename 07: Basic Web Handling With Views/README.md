07: Basic Web Handling With Views
Organize a views module with decorators and multiple views.

Background
For the examples so far, the hello_world function is a "view". In Pyramid, views are the primary way to accept web requests and return responses.

So far our examples place everything in one file:

The view function

Its registration with the configurator

The route to map it to a URL

The WSGI application launcher

Let's move the views out to their own views.py module and change our startup code to scan that module, looking for decorators that set up the views. Let's also add a second view and update our tests.

Objectives
Move views into a module that is scanned by the configurator.

Create decorators that do declarative configuration.

Steps
Let's begin by using the previous package as a starting point for a new distribution, then making it active:

cd ..; cp -r functional_testing views; cd views
$VENV/bin/pip install -e .
Our views/tutorial/__init__.py gets a lot shorter:

from pyramid.config import Configurator


def main(global_config, **settings):
    config = Configurator(settings=settings)
    config.add_route('home', '/')
    config.add_route('hello', '/howdy')
    config.scan('.views')
    return config.make_wsgi_app()
Let's add a module views/tutorial/views.py that is focused on handling requests and responses:

from pyramid.response import Response
from pyramid.view import view_config


# First view, available at http://localhost:6543/
@view_config(route_name='home')
def home(request):
    return Response('<body>Visit <a href="/howdy">hello</a></body>')


# /howdy
@view_config(route_name='hello')
def hello(request):
    return Response('<body>Go back <a href="/">home</a></body>')
Update the tests to cover the two new views:

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
        self.assertEqual(response.status_code, 200)
        self.assertIn(b'Visit', response.body)

    def test_hello(self):
        from .views import hello

        request = testing.DummyRequest()
        response = hello(request)
        self.assertEqual(response.status_code, 200)
        self.assertIn(b'Go back', response.body)


class TutorialFunctionalTests(unittest.TestCase):
    def setUp(self):
        from tutorial import main
        app = main({})
        from webtest import TestApp

        self.testapp = TestApp(app)

    def test_home(self):
        res = self.testapp.get('/', status=200)
        self.assertIn(b'<body>Visit', res.body)

    def test_hello(self):
        res = self.testapp.get('/howdy', status=200)
        self.assertIn(b'<body>Go back', res.body)
Now run the tests:

$VENV/bin/pytest tutorial/tests.py -q
....
4 passed in 0.28 seconds
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/ and http://localhost:6543/howdy in your browser.

Analysis
We added some more URLs, but we also removed the view code from the application startup code in tutorial/__init__.py. Our views, and their view registrations (via decorators) are now in a module views.py, which is scanned via config.scan('.views').

We have two views, each leading to the other. If you start at http://localhost:6543/, you get a response with a link to the next view. The hello view (available at the URL /howdy) has a link back to the first view.

This step also shows that the name appearing in the URL, the name of the "route" that maps a URL to a view, and the name of the view, can all be different. More on routes later.

Earlier we saw config.add_view as one way to configure a view. This section introduces @view_config. Pyramid's configuration supports imperative configuration, such as the config.add_view in the previous example. You can also use declarative configuration, in which a Python decorator is placed on the line above the view. Both approaches result in the same final configuration, thus usually, it is simply a matter of taste.

Extra credit
What does the dot in .views signify?

Why might assertIn be a better choice in testing the text in responses than assertEqual?

Views, View Configuration, and Debugging View Configuration
