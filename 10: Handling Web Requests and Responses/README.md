10: Handling Web Requests and Responses
Web applications handle incoming requests and return outgoing responses. Pyramid makes working with requests and responses convenient and reliable.

Objectives
Learn the background on Pyramid's choices for requests and responses.

Grab data out of the request.

Change information in the response headers.

Background
Developing for the web means processing web requests. As this is a critical part of a web application, web developers need a robust, mature set of software for web requests and returning web responses.

Pyramid has always fit nicely into the existing world of Python web development (virtual environments, packaging, cookiecutters, first to embrace Python 3, and so on). Pyramid turned to the well-regarded WebOb Python library for request and response handling. In our example above, Pyramid hands hello_world a request that is based on WebOb.

Steps
First we copy the results of the view_classes step:

cd ..; cp -r view_classes request_response; cd request_response
$VENV/bin/pip install -e .
Simplify the routes in request_response/tutorial/__init__.py:

from pyramid.config import Configurator


def main(global_config, **settings):
    config = Configurator(settings=settings)
    config.add_route('home', '/')
    config.add_route('plain', '/plain')
    config.scan('.views')
    return config.make_wsgi_app()
We only need one view in request_response/tutorial/views.py:

from pyramid.httpexceptions import HTTPFound
from pyramid.response import Response
from pyramid.view import view_config


class TutorialViews:
    def __init__(self, request):
        self.request = request

    @view_config(route_name='home')
    def home(self):
        return HTTPFound(location='/plain')

    @view_config(route_name='plain')
    def plain(self):
        name = self.request.params.get('name', 'No Name Provided')

        body = 'URL %s with name: %s' % (self.request.url, name)
        return Response(
            content_type='text/plain',
            body=body
        )
Update the tests in request_response/tutorial/tests.py:

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
        self.assertEqual(response.status, '302 Found')

    def test_plain_without_name(self):
        from .views import TutorialViews

        request = testing.DummyRequest()
        inst = TutorialViews(request)
        response = inst.plain()
        self.assertIn(b'No Name Provided', response.body)

    def test_plain_with_name(self):
        from .views import TutorialViews

        request = testing.DummyRequest()
        request.GET['name'] = 'Jane Doe'
        inst = TutorialViews(request)
        response = inst.plain()
        self.assertIn(b'Jane Doe', response.body)


class TutorialFunctionalTests(unittest.TestCase):
    def setUp(self):
        from tutorial import main

        app = main({})
        from webtest import TestApp

        self.testapp = TestApp(app)

    def test_plain_without_name(self):
        res = self.testapp.get('/plain', status=200)
        self.assertIn(b'No Name Provided', res.body)

    def test_plain_with_name(self):
        res = self.testapp.get('/plain?name=Jane%20Doe', status=200)
        self.assertIn(b'Jane Doe', res.body)
Now run the tests:

$VENV/bin/pytest tutorial/tests.py -q
.....
5 passed in 0.30 seconds
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/ in your browser. You will be redirected to http://localhost:6543/plain.

Open http://localhost:6543/plain?name=alice in your browser.

Analysis
In this view class, we have two routes and two views, with the first leading to the second by an HTTP redirect. Pyramid can generate redirects by returning a special object from a view or raising a special exception.

In this Pyramid view, we get the URL being visited from request.url. Also, if you visited http://localhost:6543/plain?name=alice, the name is included in the body of the response:

URL http://localhost:6543/plain?name=alice with name: alice
Finally, we set the response's content type and body, then return the response.

We updated the unit and functional tests to prove that our code does the redirection, but also handles sending and not sending /plain?name.

Extra credit
Could we also raise HTTPFound(location='/plain') instead of returning it? If so, what's the difference?

Request and Response Objects, generate redirects
