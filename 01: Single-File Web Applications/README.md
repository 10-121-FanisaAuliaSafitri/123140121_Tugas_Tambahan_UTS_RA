01: Single-File Web Applications
What's the simplest way to get started in Pyramid? A single-file module. No Python packages, no pip install -e ., no other machinery.

Background
Microframeworks were all the rage, until the next shiny thing came along. "Microframework" is a marketing term, not a technical one. They have a low mental overhead: they do so little, the only things you have to worry about are your things.

Pyramid is special because it can act as a single-file module microframework. You can have a single Python file that can be executed directly by Python. But Pyramid also provides facilities to scale to the largest of applications.

Python has a standard called WSGI that defines how Python web applications plug into standard servers, getting passed incoming requests, and returning responses. Most modern Python web frameworks obey an "MVC" (model-view-controller) application pattern, where the data in the model has a view that mediates interaction with outside systems.

In this step we'll see a brief glimpse of WSGI servers, WSGI applications, requests, responses, and views.

Objectives
Get a running Pyramid web application, as simply as possible.

Use that as a well-understood base for adding each unit of complexity.

Initial exposure to WSGI apps, requests, views, and responses.

Steps
Make sure you have followed the steps in Requirements.

Starting from your workspace directory (~/projects/quick_tutorial), create a directory for this step:

cd ~/projects/quick_tutorial; mkdir hello_world; cd hello_world
Copy the following into hello_world/app.py:

from waitress import serve
from pyramid.config import Configurator
from pyramid.response import Response


def hello_world(request):
    print('Incoming request')
    return Response('<body><h1>Hello World!</h1></body>')


if __name__ == '__main__':
    with Configurator() as config:
        config.add_route('hello', '/')
        config.add_view(hello_world, route_name='hello')
        app = config.make_wsgi_app()
    serve(app, host='0.0.0.0', port=6543)
Run the application:

$VENV/bin/python app.py
Open http://localhost:6543/ in your browser.

Analysis
New to Python web programming? If so, some lines in the module merit explanation:

Line 11. The if __name__ == '__main__': is Python's way of saying, "Start here when running from the command line", rather than when this module is imported.

Lines 12-14. Use Pyramid's configurator in a context manager to connect view code to a particular URL route.

Lines 6-8. Implement the view code that generates the response.

Lines 15-17. Publish a WSGI app using an HTTP server.

As shown in this example, the configurator plays a central role in Pyramid development. Building an application from loosely-coupled parts via Application Configuration is a central idea in Pyramid, one that we will revisit regularly in this Quick Tutorial.

Extra credit
Why do we do this:

print('Incoming request')
...instead of:

print 'Incoming request'
What happens if you return a string of HTML? A sequence of integers?

Put something invalid, such as print xyz, in the view function. Kill your python app.py with ctrl-C and restart, then reload your browser. See the exception in the console?

The GI in WSGI stands for "Gateway Interface". What web standard is this modelled after?
