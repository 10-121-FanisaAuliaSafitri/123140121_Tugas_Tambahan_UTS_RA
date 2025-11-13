12: Templating With jinja2
We just said Pyramid doesn't prefer one templating language over another. Time to prove it. Jinja2 is a popular templating system, used in Flask and modeled after Django's templates. Let's add pyramid_jinja2, a Pyramid add-on which enables Jinja2 as a renderer in our Pyramid applications.

Objectives
Show Pyramid's support for different templating systems.

Learn about installing Pyramid add-ons.

Steps
In this step let's start by copying the view_class step's directory from a few steps ago.

cd ..; cp -r view_classes jinja2; cd jinja2
Add pyramid_jinja2 to our project's dependencies in setup.py:

from setuptools import setup

# List of dependencies installed via `pip install -e .`
# by virtue of the Setuptools `install_requires` value below.
requires = [
    'pyramid',
    'pyramid_chameleon',
    'pyramid_jinja2',
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
Install our project and its newly added dependency.

$VENV/bin/pip install -e .
We need to include pyramid_jinja2 in jinja2/tutorial/__init__.py:

from pyramid.config import Configurator


def main(global_config, **settings):
    config = Configurator(settings=settings)
    config.include('pyramid_jinja2')
    config.add_route('home', '/')
    config.add_route('hello', '/howdy')
    config.scan('.views')
    return config.make_wsgi_app()
Our jinja2/tutorial/views.py simply changes its renderer:

from pyramid.view import (
    view_config,
    view_defaults
    )


@view_defaults(renderer='home.jinja2')
class TutorialViews:
    def __init__(self, request):
        self.request = request

    @view_config(route_name='home')
    def home(self):
        return {'name': 'Home View'}

    @view_config(route_name='hello')
    def hello(self):
        return {'name': 'Hello View'}
Add jinja2/tutorial/home.jinja2 as a template:

<!DOCTYPE html>
<html lang="en">
<head>
    <title>Quick Tutorial: {{ name }}</title>
</head>
<body>
<h1>Hi {{ name }}</h1>
</body>
</html>
Now run the tests:

$VENV/bin/pytest tutorial/tests.py -q
....
4 passed in 0.40 seconds
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/ in your browser.

Analysis
Getting a Pyramid add-on into Pyramid is simple. First you use normal Python package installation tools to install the add-on package into your Python virtual environment. You then tell Pyramid's configurator to run the setup code in the add-on. In this case the setup code told Pyramid to make a new "renderer" available that looked for .jinja2 file extensions.

Our view code stayed largely the same. We simply changed the file extension on the renderer. For the template, the syntax for Chameleon and Jinja2's basic variable insertion is very similar.

Extra credit
Our project now depends on pyramid_jinja2. We installed that dependency manually. What is another way we could have made the association?

We used config.include which is an imperative configuration to get the Configurator to load pyramid_jinja2's configuration. What is another way we could include it into the config?

Jinja2 homepage, and pyramid_jinja2 Overview
