21: Protecting Resources With Authorization
Assign security statements to resources describing the permissions required to perform an operation.

Background
Our application has URLs that allow people to add/edit/delete content via a web browser. Time to add security to the application. Let's protect our add/edit views to require a login (username of editor and password of editor). We will allow the other views to continue working without a password.

Objectives
Introduce the Pyramid concepts of authentication, authorization, permissions, and access control lists (ACLs).

Make a root factory that returns an instance of our class for the top of the application.

Assign security statements to our root resource.

Add a permissions predicate on a view.

Provide a Forbidden view to handle visiting a URL without adequate permissions.

Steps
We are going to use the authentication step as our starting point:

cd ..; cp -r authentication authorization; cd authorization
$VENV/bin/pip install -e .
Start by changing authorization/tutorial/__init__.py to specify a root factory to the configurator:

from pyramid.config import Configurator

from .security import SecurityPolicy


def main(global_config, **settings):
    config = Configurator(settings=settings,
                          root_factory='.resources.Root')
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
That means we need to implement authorization/tutorial/resources.py:

from pyramid.authorization import Allow, Everyone


class Root:
    __acl__ = [(Allow, Everyone, 'view'),
               (Allow, 'group:editors', 'edit')]

    def __init__(self, request):
        pass
Define a GROUPS data store and the permits method of our SecurityPolicy:

import bcrypt
from pyramid.authentication import AuthTktCookieHelper
from pyramid.authorization import (
    ACLHelper,
    Authenticated,
    Everyone,
)


def hash_password(pw):
    pwhash = bcrypt.hashpw(pw.encode('utf8'), bcrypt.gensalt())
    return pwhash.decode('utf8')

def check_password(pw, hashed_pw):
    expected_hash = hashed_pw.encode('utf8')
    return bcrypt.checkpw(pw.encode('utf8'), expected_hash)


USERS = {'editor': hash_password('editor'),
         'viewer': hash_password('viewer')}
GROUPS = {'editor': ['group:editors']}


class SecurityPolicy:
    def __init__(self, secret):
        self.authtkt = AuthTktCookieHelper(secret=secret)
        self.acl = ACLHelper()

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

    def permits(self, request, context, permission):
        principals = self.effective_principals(request)
        return self.acl.permits(context, principals, permission)

    def effective_principals(self, request):
        principals = [Everyone]
        userid = self.authenticated_userid(request)
        if userid is not None:
            principals += [Authenticated, 'u:' + userid]
            principals += GROUPS.get(userid, [])
        return principals
Change authorization/tutorial/views.py to require the edit permission on the hello view and implement the forbidden view:

from pyramid.httpexceptions import HTTPFound
from pyramid.security import (
    remember,
    forget,
)

from pyramid.view import (
    view_config,
    view_defaults,
    forbidden_view_config
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

    @view_config(route_name='hello', permission='edit')
    def hello(self):
        return {'name': 'Hello View'}

    @view_config(route_name='login', renderer='login.pt')
    @forbidden_view_config(renderer='login.pt')
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
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/ in a browser.

If you are still logged in, click the "Log Out" link.

Visit http://localhost:6543/howdy in a browser. You should be asked to login.

Analysis
This simple tutorial step can be boiled down to the following:

A view can require a permission (edit).

The context for our view (the Root) has an access control list (ACL).

This ACL says that the edit permission is available on Root to the group:editors principal.

The SecurityPolicy.effective_principals method answers whether a particular user (editor) is a member of a particular group (group:editors).

The SecurityPolicy.permits method is invoked when Pyramid wants to know whether the user is allowed to do something. To do this, it uses the pyramid.authorization.ACLHelper to inspect the ACL on the context and determine if the request is allowed or denied the specific permission.

In summary, hello wants edit permission, Root says group:editors has edit permission.

Of course, this only applies on Root. Some other part of the site (a.k.a. context) might have a different ACL.

If you are not logged in and visit /howdy, you need to get shown the login screen. How does Pyramid know what is the login page to use? We explicitly told Pyramid that the login view should be used by decorating the view with @forbidden_view_config.

Extra credit
What is the difference between a user and a principal?

Can I use a database instead of the GROUPS data store to look up principals?

Do I have to put a renderer in my @forbidden_view_config decorator?

Perhaps you would like the experience of not having enough permissions (forbidden) to be richer. How could you change this?

Perhaps we want to store security statements in a database and allow editing via a browser. How might this be done?

What if we want different security statements on different kinds of objects? Or on the same kinds of objects, but in different parts of a URL hierarchy?
