17: Transient Data Using Sessions
Store and retrieve non-permanent data in Pyramid sessions.

Background
When people use your web application, they frequently perform a task that requires semi-permanent data to be saved. For example, a shopping cart. This is called a session.

Pyramid has basic built-in support for sessions. Third party packages such as pyramid_redis_sessions provide richer session support. Or you can create your own custom sessioning engine. Let's take a look at the built-in sessioning support.

Objectives
Make a session factory using a built-in, simple Pyramid sessioning system.

Change our code to use a session.

Steps
First we copy the results of the view_classes step:

cd ..; cp -r view_classes sessions; cd sessions
$VENV/bin/pip install -e .
Our sessions/tutorial/__init__.py needs a choice of session factory to get registered with the configurator:

from pyramid.config import Configurator
from pyramid.session import SignedCookieSessionFactory


def main(global_config, **settings):
    my_session_factory = SignedCookieSessionFactory(
        'itsaseekreet')
    config = Configurator(settings=settings,
                          session_factory=my_session_factory)
    config.include('pyramid_chameleon')
    config.add_route('home', '/')
    config.add_route('hello', '/howdy')
    config.scan('.views')
    return config.make_wsgi_app()
Our views in sessions/tutorial/views.py can now use request.session:

from pyramid.view import (
    view_config,
    view_defaults
    )


@view_defaults(renderer='home.pt')
class TutorialViews:
    def __init__(self, request):
        self.request = request

    @property
    def counter(self):
        session = self.request.session
        if 'counter' in session:
            session['counter'] += 1
        else:
            session['counter'] = 1

        return session['counter']


    @view_config(route_name='home')
    def home(self):
        return {'name': 'Home View'}

    @view_config(route_name='hello')
    def hello(self):
        return {'name': 'Hello View'}
The template at sessions/tutorial/home.pt can display the value:

<!DOCTYPE html>
<html lang="en">
<head>
    <title>Quick Tutorial: ${name}</title>
</head>
<body>
<h1>Hi ${name}</h1>
<p>Count: ${view.counter}</p>
</body>
</html>
Make sure the tests still pass:

$VENV/bin/pytest tutorial/tests.py -q
....
4 passed in 0.42 seconds
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/ and http://localhost:6543/howdy in your browser. As you reload and switch between those URLs, note that the counter increases and is not specific to the URL.

Restart the application and revisit the page. Note that counter still increases from where it left off.

Analysis
Pyramid's request object now has a session attribute that we can use in our view code. It acts like a dictionary.

Since all the views are using the same counter, we made the counter a Python property at the view class level. With this, each reload will increase the counter displayed in our template.

In web development, "flash messages" are notes for the user that need to appear on a screen after a future web request. For example, when you add an item using a form POST, the site usually issues a second HTTP Redirect web request to view the new item. You might want a message to appear after that second web request saying "Your item was added." You can't just return it in the web response for the POST, as it will be tossed out during the second web request.

Flash messages are a technique where messages can be stored between requests, using sessions, then removed when they finally get displayed.

Sessions, Flash Messages, and pyramid.session.
