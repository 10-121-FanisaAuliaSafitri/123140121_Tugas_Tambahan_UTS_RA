11: Dispatching URLs To Views With Routing¶
Routing matches incoming URL patterns to view code. Pyramid's routing has a number of useful features.

Background
Writing web applications usually means sophisticated URL design. We just saw some Pyramid machinery for requests and views. Let's look at features that help in routing.

Previously we saw the basics of routing URLs to views in Pyramid.

Your project's "setup" code registers a route name to be used when matching part of the URL

Elsewhere a view is configured to be called for that route name.

Why do this twice? Other Python web frameworks let you create a route and associate it with a view in one step. As illustrated in Routes need relative ordering, multiple routes might match the same URL pattern. Rather than provide ways to help guess, Pyramid lets you be explicit in ordering. Pyramid also gives facilities to avoid the problem. It's relatively easy to build a system that uses implicit route ordering with Pyramid too. See The Groundhog series of screencasts if you're interested in doing so.
Objectives
Define a route that extracts part of the URL into a Python dictionary.

Use that dictionary data in a view.

Steps
First we copy the results of the view_classes step:

cd ..; cp -r view_classes routing; cd routing
$VENV/bin/pip install -e .
Our routing/tutorial/__init__.py needs a route with a replacement pattern:

from pyramid.config import Configurator


def main(global_config, **settings):
    config = Configurator(settings=settings)
    config.include('pyramid_chameleon')
    config.add_route('home', '/howdy/{first}/{last}')
    config.scan('.views')
    return config.make_wsgi_app()
We just need one view in routing/tutorial/views.py:

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
        first = self.request.matchdict['first']
        last = self.request.matchdict['last']
        return {
            'name': 'Home View',
            'first': first,
            'last': last
        }
We just need one view in routing/tutorial/home.pt:

<!DOCTYPE html>
<html lang="en">
<head>
    <title>Quick Tutorial: ${name}</title>
</head>
<body>
<h1>${name}</h1>
<p>First: ${first}, Last: ${last}</p>
</body>
</html>
Update routing/tutorial/tests.py:

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
        request.matchdict['first'] = 'First'
        request.matchdict['last'] = 'Last'
        inst = TutorialViews(request)
        response = inst.home()
        self.assertEqual(response['first'], 'First')
        self.assertEqual(response['last'], 'Last')


class TutorialFunctionalTests(unittest.TestCase):
    def setUp(self):
        from tutorial import main
        app = main({})
        from webtest import TestApp

        self.testapp = TestApp(app)

    def test_home(self):
        res = self.testapp.get('/howdy/Jane/Doe', status=200)
        self.assertIn(b'Jane', res.body)
        self.assertIn(b'Doe', res.body)
Now run the tests:

$VENV/bin/pytest tutorial/tests.py -q
..
2 passed in 0.39 seconds
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/howdy/amy/smith in your browser.

Analysis
In __init__.py we see an important change in our route declaration:

config.add_route('hello', '/howdy/{first}/{last}')
With this we tell the configurator that our URL has a "replacement pattern". With this, URLs such as /howdy/amy/smith will assign amy to first and smith to last. We can then use this data in our view:

self.request.matchdict['first']
self.request.matchdict['last']
request.matchdict contains values from the URL that match the "replacement patterns" (the curly braces) in the route declaration. This information can then be used anywhere in Pyramid that has access to the request.

Extra credit
What happens if you to go the URL http://localhost:6543/howdy? Is this the result that you expected?
