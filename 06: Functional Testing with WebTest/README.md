06: Functional Testing with WebTest
Write end-to-end full-stack testing using webtest.

Background
Unit tests are a common and popular approach to test-driven development (TDD). In web applications, though, the templating and entire apparatus of a web site are important parts of the delivered quality. We'd like a way to test these.

WebTest is a Python package that does functional testing. With WebTest you can write tests which simulate a full HTTP request against a WSGI application, then test the information in the response. For speed purposes, WebTest skips the setup/teardown of an actual HTTP server, providing tests that run fast enough to be part of TDD.

Objectives
Write a test which checks the contents of the returned HTML.

Steps
First we copy the results of the previous step.

cd ..; cp -r unit_testing functional_testing; cd functional_testing
Add webtest to our project's dependencies in setup.py as a Setuptools "extra":

from setuptools import setup

# List of dependencies installed via `pip install -e .`
# by virtue of the Setuptools `install_requires` value below.
requires = [
    'pyramid',
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
Install our project and its newly added dependency. Note that we use the extra specifier [dev] to install testing requirements for development and surround it and the period with double quote marks.

$VENV/bin/pip install -e ".[dev]"
Let's extend functional_testing/tutorial/tests.py to include a functional test:

import unittest

from pyramid import testing


class TutorialViewTests(unittest.TestCase):
    def setUp(self):
        self.config = testing.setUp()

    def tearDown(self):
        testing.tearDown()

    def test_hello_world(self):
        from tutorial import hello_world

        request = testing.DummyRequest()
        response = hello_world(request)
        self.assertEqual(response.status_code, 200)


class TutorialFunctionalTests(unittest.TestCase):
    def setUp(self):
        from tutorial import main
        app = main({})
        from webtest import TestApp

        self.testapp = TestApp(app)

    def test_hello_world(self):
        res = self.testapp.get('/', status=200)
        self.assertIn(b'<h1>Hello World!</h1>', res.body)
Be sure this file is not executable, or pytest may not include your tests.

Now run the tests:

$VENV/bin/pytest tutorial/tests.py -q
..
2 passed in 0.25 seconds
Analysis
We now have the end-to-end testing we were looking for. WebTest lets us simply extend our existing pytest-based test approach with functional tests that are reported in the same output. These new tests not only cover our templating, but they didn't dramatically increase the execution time of our tests.

Extra credit
Why do our functional tests use b''?

