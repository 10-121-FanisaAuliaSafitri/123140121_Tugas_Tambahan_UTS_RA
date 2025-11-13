09: Organizing Views With View Classes
Change our view functions to be methods on a view class, then move some declarations to the class level.

Background
So far our views have been simple, free-standing functions. Many times your views are related to one another. They may consist of different ways to look at or work on the same data, or be a REST API that handles multiple operations. Grouping these views together as a view class makes sense:

Group views.

Centralize some repetitive defaults.

Share some state and helpers.

In this step we just do the absolute minimum to convert the existing views to a view class. In a later tutorial step, we'll examine view classes in depth.

Objectives
Group related views into a view class.

Centralize configuration with class-level @view_defaults.

Steps
First we copy the results of the previous step:

cd ..; cp -r templating view_classes; cd view_classes
$VENV/bin/pip install -e .
Our view_classes/tutorial/views.py now has a view class with our two views:

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
    def hello(self):
        return {'name': 'Hello View'}
Our unit tests in view_classes/tutorial/tests.py don't run, so let's modify them to import the view class, and make an instance before getting a response:

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
Now run the tests:

$VENV/bin/pytest tutorial/tests.py -q
....
4 passed in 0.34 seconds
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/ and http://localhost:6543/howdy in your browser.

Analysis
To ease the transition to view classes, we didn't introduce any new functionality. We simply changed the view functions to methods on a view class, then updated the tests.

In our TutorialViews view class, you can see that our two view functions are logically grouped together as methods on a common class. Since the two views shared the same template, we could move that to a @view_defaults decorator at the class level.

The tests needed to change. Obviously we needed to import the view class. But you can also see the pattern in the tests of instantiating the view class with the dummy request first, then calling the view method being tested.

Defining a View Callable as a Class
