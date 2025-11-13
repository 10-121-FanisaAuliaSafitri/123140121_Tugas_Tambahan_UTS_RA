20: Logins with authentication
Login views that authenticate a username and password against a list of users.

Background
Most web applications have URLs that allow people to add/edit/delete content via a web browser. Time to add security to the application. In this first step we introduce authentication. That is, logging in and logging out, using Pyramid's rich facilities for pluggable user storage.

In the next step we will introduce protection of resources with authorization security statements.

Objectives
Introduce the Pyramid concepts of authentication.

Create login and logout views.

Steps
We are going to use the view classes step as our starting point:

cd ..; cp -r view_classes authentication; cd authentication
Add bcrypt as a dependency in authentication/setup.py:

from setuptools import setup

# List of dependencies installed via `pip install -e .`
# by virtue of the Setuptools `install_requires` value below.
requires = [
    'bcrypt',
    'pyramid',
    'pyramid_chameleon',
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
We can now install our project in development mode:

$VENV/bin/pip install -e .
Put the security hash in the authentication/development.ini configuration file as tutorial.secret instead of putting it in the code:

[app:main]
use = egg:tutorial
pyramid.reload_templates = true
pyramid.includes =
    pyramid_debugtoolbar
tutorial.secret = 98zd

[server:main]
use = egg:waitress#main
listen = localhost:6543
Create an authentication/tutorial/security.py module that can find our user information by providing a security policy:

import bcrypt
from pyramid.authentication import AuthTktCookieHelper


def hash_password(pw):
    pwhash = bcrypt.hashpw(pw.encode('utf8'), bcrypt.gensalt())
    return pwhash.decode('utf8')

def check_password(pw, hashed_pw):
    expected_hash = hashed_pw.encode('utf8')
    return bcrypt.checkpw(pw.encode('utf8'), expected_hash)


USERS = {'editor': hash_password('editor'),
         'viewer': hash_password('viewer')}


class SecurityPolicy:
    def __init__(self, secret):
        self.authtkt = AuthTktCookieHelper(secret=secret)

    def identity(self, request):
        identity = self.authtkt.identify(request)
        if identity is not None and identity['userid'] in USERS:
            return identity

    def authenticated_userid(self, request):
        identity = self.identity(request)
        if identity is not None:
            return identity['userid']

    def remember(self, request, userid, **kw):
        return self.authtkt.remember(request, userid, **kw)

    def forget(self, request, **kw):
        return self.authtkt.forget(request, **kw)
Register the SecurityPolicy with the configurator in authentication/tutorial/__init__.py:

from pyramid.config import Configurator

from .security import SecurityPolicy


def main(global_config, **settings):
    config = Configurator(settings=settings)
    config.include('pyramid_chameleon')

    config.set_security_policy(
        SecurityPolicy(
            secret=settings['tutorial.secret'],
        ),
    )

    config.add_route('home', '/')
    config.add_route('hello', '/howdy')
    config.add_route('login', '/login')
    config.add_route('logout', '/logout')
    config.scan('.views')
    return config.make_wsgi_app()
Update the views in authentication/tutorial/views.py:

from pyramid.httpexceptions import HTTPFound
from pyramid.security import (
    remember,
    forget,
    )

from pyramid.view import (
    view_config,
    view_defaults
    )

from .security import (
    USERS,
    check_password
)


@view_defaults(renderer='home.pt')
class TutorialViews:
    def __init__(self, request):
        self.request = request
        self.logged_in = request.authenticated_userid

    @view_config(route_name='home')
    def home(self):
        return {'name': 'Home View'}

    @view_config(route_name='hello')
    def hello(self):
        return {'name': 'Hello View'}

    @view_config(route_name='login', renderer='login.pt')
    def login(self):
        request = self.request
        login_url = request.route_url('login')
        referrer = request.url
        if referrer == login_url:
            referrer = '/'  # never use login form itself as came_from
        came_from = request.params.get('came_from', referrer)
        message = ''
        login = ''
        password = ''
        if 'form.submitted' in request.params:
            login = request.params['login']
            password = request.params['password']
            hashed_pw = USERS.get(login)
            if hashed_pw and check_password(password, hashed_pw):
                headers = remember(request, login)
                return HTTPFound(location=came_from,
                                 headers=headers)
            message = 'Failed login'

        return dict(
            name='Login',
            message=message,
            url=request.application_url + '/login',
            came_from=came_from,
            login=login,
            password=password,
        )

    @view_config(route_name='logout')
    def logout(self):
        request = self.request
        headers = forget(request)
        url = request.route_url('home')
        return HTTPFound(location=url,
                         headers=headers)
Add a login template at authentication/tutorial/login.pt:

<!DOCTYPE html>
<html lang="en">
<head>
    <title>Quick Tutorial: ${name}</title>
</head>
<body>
<h1>Login</h1>
<span tal:replace="message"/>

<form action="${url}" method="post">
    <input type="hidden" name="came_from"
           value="${came_from}"/>
    <label for="login">Username</label>
    <input type="text" id="login"
           name="login"
           value="${login}"/><br/>
    <label for="password">Password</label>
    <input type="password" id="password"
           name="password"
           value="${password}"/><br/>
    <input type="submit" name="form.submitted"
           value="Log In"/>
</form>
</body>
</html>
Provide a login/logout box in authentication/tutorial/home.pt:

<!DOCTYPE html>
<html lang="en">
<head>
    <title>Quick Tutorial: ${name}</title>
</head>
<body>

<div>
    <a tal:condition="view.logged_in is None"
            href="${request.application_url}/login">Log In</a>
    <a tal:condition="view.logged_in is not None"
            href="${request.application_url}/logout">Logout</a>
</div>

<h1>Hi ${name}</h1>
<p>Visit <a href="${request.route_url('hello')}">hello</a></p>
</body>
</html>
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/ in a browser.

Click the "Log In" link.

Submit the login form with the username editor and the password editor.

Note that the "Log In" link has changed to "Logout".

Click the "Logout" link.

Analysis
Unlike many web frameworks, Pyramid includes a built-in but optional security model for authentication and authorization. This security system is intended to be flexible and support many needs. In this security model, authentication (who are you) and authorization (what are you allowed to do) are pluggable. To learn one step at a time, we provide a system that identifies users and lets them log out.

In this example we chose to use the bundled pyramid.authentication.AuthTktCookieHelper helper to store the user's logged-in state in a cookie. We enabled it in our configuration and provided a ticket-signing secret in our INI file.

Our view class grew a login view. When you reached it via a GET request, it returned a login form. When reached via POST, it processed the submitted username and password against the USERS data store.

The function hash_password uses a one-way hashing algorithm with a salt on the user's password via bcrypt, instead of storing the password in plain text. This is considered to be a "best practice" for security.

There are alternative libraries to bcrypt if it is an issue on your system. Just make sure that the library uses an algorithm approved for storing passwords securely.
The function check_password will compare the two hashed values of the submitted password and the user's password stored in the database. If the hashed values are equivalent, then the user is authenticated, else authentication fails.

Assuming the password was validated, we invoke pyramid.security.remember() to generate a cookie that is set in the response. Subsequent requests return that cookie and identify the user.

In our template, we fetched the logged_in value from the view class. We use this to calculate the logged-in user, if any. In the template we can then choose to show a login link to anonymous visitors or a logout link to logged-in users.

Extra credit
Can I use a database instead of USERS to authenticate users?

Once I am logged in, does any user-centric information get jammed onto each request? Use import pdb; pdb.set_trace() to answer this.

See also Security, pyramid.authentication.AuthTktCookieHelper, bcrypt
