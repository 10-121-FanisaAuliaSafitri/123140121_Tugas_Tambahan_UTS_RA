05: Unit Tests and pytest
Provide unit testing for our project's Python code.

Background
As the mantra says, "Untested code is broken code." The Python community has had a long culture of writing test scripts which ensure that your code works correctly as you write it and maintain it in the future. Pyramid has always had a deep commitment to testing, with 100% test coverage from the earliest pre-releases.

Python includes a unit testing framework in its standard library. Over the years a number of Python projects, such as pytest, have extended this framework with alternative test runners that provide more convenience and functionality. The Pyramid developers use pytest, which we'll use in this tutorial.

Don't worry, this tutorial won't be pedantic about "test-driven development" (TDD). We'll do just enough to ensure that, in each step, we haven't majorly broken the code. As you're writing your code, you might find this more convenient than changing to your browser constantly and clicking reload.

We'll also leave discussion of pytest-cov for another section.

Objectives
Write unit tests that ensure the quality of our code.

Install a Python package (pytest) which helps in our testing.

Steps
First we copy the results of the previous step.

cd ..; cp -r debugtoolbar unit_testing; cd unit_testing
Add pytest to our project's dependencies in setup.py as a Setuptools "extra":

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
Now we write a simple unit test in unit_testing/tutorial/tests.py:

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
Now run the tests:

$VENV/bin/pytest tutorial/tests.py -q
.
1 passed in 0.14 seconds
Analysis
Our tests.py imports the Python standard unit testing framework. To make writing Pyramid-oriented tests more convenient, Pyramid supplies some pyramid.testing helpers which we use in the test setup and teardown. Our one test imports the view, makes a dummy request, and sees if the view returns what we expect.

The tests.TutorialViewTests.test_hello_world test is a small example of a unit test. First, we import the view inside each test. Why not import at the top, like in normal Python code? Because imports can cause effects that break a test. We'd like our tests to be in units, hence the name unit testing. Each test should isolate itself to the correct degree.

Our test then makes a fake incoming web request, then calls our Pyramid view. We test the HTTP status code on the response to make sure it matches our expectations.

Note that our use of pyramid.testing.setUp() and pyramid.testing.tearDown() aren't actually necessary here; they are only necessary when your test needs to make use of the config object (it's a Configurator) to add stuff to the configuration state before calling the view.

Extra credit
Change the test to assert that the response status code should be 404 (meaning, not found). Run pytest again. Read the error report and see if you can decipher what it is telling you.

As a more realistic example, put the tests.py back as you found it, and put an error in your view, such as a reference to a non-existing variable. Run the tests and see how this is more convenient than reloading your browser and going back to your code.

Finally, for the most realistic test, read about Pyramid Response objects and see how to change the response code. Run the tests and see how testing confirms the "contract" that your code claims to support.

How could we add a unit test assertion to test the HTML value of the response body?

Why do we import the hello_world view function inside the test_hello_world method instead of at the top of the module?

See also Unit, Integration, and Functional Testing and Setuptools Declaring "Extras" (optional features with their own dependencies).
