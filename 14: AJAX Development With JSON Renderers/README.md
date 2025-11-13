14: AJAX Development With JSON Renderers
Modern web apps are more than rendered HTML. Dynamic pages now use JavaScript to update the UI in the browser by requesting server data as JSON. Pyramid supports this with a JSON renderer.

Background
As we saw in 08: HTML Generation With Templating, view declarations can specify a renderer. Output from the view is then run through the renderer, which generates and returns the response. We first used a Chameleon renderer, then a Jinja2 renderer.

Renderers aren't limited, however, to templates that generate HTML. Pyramid supplies a JSON renderer which takes Python data, serializes it to JSON, and performs some other functions such as setting the content type. In fact you can write your own renderer (or extend a built-in renderer) containing custom logic for your unique application.

Steps
First we copy the results of the view_classes step:

cd ..; cp -r view_classes json; cd json
$VENV/bin/pip install -e .
We add a new route for hello_json in json/tutorial/__init__.py:

from pyramid.config import Configurator


def main(global_config, **settings):
    config = Configurator(settings=settings)
    config.include('pyramid_chameleon')
    config.add_route('home', '/')
    config.add_route('hello', '/howdy')
    config.add_route('hello_json', '/howdy.json')
    config.scan('.views')
    return config.make_wsgi_app()
Rather than implement a new view, we will "stack" another decorator on the hello view in views.py:

from pyramid.view import (
    view_config,
    view_defaults
    )


@view_defaults(renderer='home.pt')
class TutorialViews:
    def __init__(self, request):
        self.request = request

    @view_config(route_name='home')
    def home(self):
        return {'name': 'Home View'}

    @view_config(route_name='hello')
    @view_config(route_name='hello_json', renderer='json')
    def hello(self):
        return {'name': 'Hello View'}
We need a new functional test at the end of json/tutorial/tests.py:

import unittest

from pyramid import testing


class TutorialViewTests(unittest.TestCase):
    def setUp(self):
        self.config = testing.setUp()

    def tearDown(self):
        testing.tearDown()

    def test_home(self):
        from .views import TutorialViews

        request = testing.DummyRequest()
        inst = TutorialViews(request)
        response = inst.home()
        self.assertEqual('Home View', response['name'])

    def test_hello(self):
        from .views import TutorialViews

        request = testing.DummyRequest()
        inst = TutorialViews(request)
        response = inst.hello()
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

    def test_hello_json(self):
        res = self.testapp.get('/howdy.json', status=200)
        self.assertIn(b'{"name": "Hello View"}', res.body)
        self.assertEqual(res.content_type, 'application/json')

Run the tests:

$VENV/bin/pytest tutorial/tests.py -q
.....
5 passed in 0.47 seconds
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/howdy.json in your browser and you will see the resulting JSON response.

Analysis
Earlier we changed our view functions and methods to return Python data. This change to a data-oriented view layer made test writing easier, decoupling the templating from the view logic.

Since Pyramid has a JSON renderer as well as the templating renderers, it is an easy step to return JSON. In this case we kept the exact same view and arranged to return a JSON encoding of the view data. We did this by:

Adding a route to map /howdy.json to a route name.

Providing a @view_config that associated that route name with an existing view.

Overriding the view defaults in the view config that mentions the hello_json route, so that when the route is matched, we use the JSON renderer rather than the home.pt template renderer that would otherwise be used.

In fact, for pure AJAX-style web applications, we could re-use the existing route by using Pyramid's view predicates to match on the Accepts: header sent by modern AJAX implementations.

Pyramid's JSON renderer uses the base Python JSON encoder, thus inheriting its strengths and weaknesses. For example, Python can't natively JSON encode DateTime objects. There are a number of solutions for this in Pyramid, including extending the JSON renderer with a custom renderer.

Writing View Callables Which Use a Renderer, JSON Renderer, and Adding and Changing Renderers
