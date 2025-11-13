16: Collecting Application Info With Logging
Capture debugging and error output from your web applications using standard Python logging.

Background
It's important to know what is going on inside our web application. In development we might need to collect some output. In production, we might need to detect problems when other people use the site. We need logging.

Fortunately Pyramid uses the normal Python approach to logging. The project generated in your development.ini has a number of lines that configure the logging for you to some reasonable defaults. You then see messages sent by Pyramid, for example, when a new request comes in.

Objectives
Inspect the configuration setup used for logging.

Add logging statements to your view code.

Steps
First we copy the results of the view_classes step:

cd ..; cp -r view_classes logging; cd logging
$VENV/bin/pip install -e .
Extend logging/tutorial/views.py to log a message:

import logging
log = logging.getLogger(__name__)

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
        log.debug('In home view')
        return {'name': 'Home View'}

    @view_config(route_name='hello')
    def hello(self):
        log.debug('In hello view')
        return {'name': 'Hello View'}
Finally let's edit development.ini configuration file to enable logging for our Pyramid application:

[app:main]
use = egg:tutorial
pyramid.reload_templates = true
pyramid.includes =
    pyramid_debugtoolbar

[server:main]
use = egg:waitress#main
listen = localhost:6543

# Begin logging configuration

[loggers]
keys = root, tutorial

[logger_tutorial]
level = DEBUG
handlers =
qualname = tutorial

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = INFO
handlers = console

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(asctime)s %(levelname)-5.5s [%(name)s][%(threadName)s] %(message)s

# End logging configuration
Make sure the tests still pass:

$VENV/bin/pytest tutorial/tests.py -q
....
4 passed in 0.41 seconds
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/ and http://localhost:6543/howdy in your browser. Note, both in the console and in the debug toolbar, the message that you logged.

Analysis
In our configuration file development.ini, our tutorial Python package is set up as a logger and configured to log messages at a DEBUG or higher level. When you visit http://localhost:6543, your console will now show:

2013-08-09 10:42:42,968 DEBUG [tutorial.views][MainThread] In home view
Also, if you have configured your Pyramid application to use the pyramid_debugtoolbar, logging statements appear in one of its menus.

See also Logging.
