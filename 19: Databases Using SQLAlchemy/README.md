19: Databases Using SQLAlchemy
Store and retrieve data using the SQLAlchemy ORM atop the SQLite database.

Background
Our Pyramid-based wiki application now needs database-backed storage of pages. This frequently means an SQL database. The Pyramid community strongly supports the SQLAlchemy project and its object-relational mapper (ORM) as a convenient, Pythonic way to interface to databases.

In this step we hook up SQLAlchemy to a SQLite database table, providing storage and retrieval for the wiki pages in the previous step.

The Pyramid cookiecutter pyramid-cookiecutter-starter is really helpful for getting a SQLAlchemy project going, including generation of the console script. Since we want to see all the decisions, we will forgo convenience in this tutorial, and wire it up ourselves.
Objectives
Store pages in SQLite by using SQLAlchemy models.

Use SQLAlchemy queries to list/add/view/edit pages.

Provide a database-initialize command by writing a Pyramid console script which can be run from the command line.

Steps
We are going to use the forms step as our starting point:

cd ..; cp -r forms databases; cd databases
We need to add some dependencies in databases/setup.py as well as an entry point for the command-line script:

from setuptools import setup

# List of dependencies installed via `pip install -e .`
# by virtue of the Setuptools `install_requires` value below.
requires = [
    'deform',
    'pyramid',
    'pyramid_chameleon',
    'pyramid_tm',
    'sqlalchemy',
    'waitress',
    'zope.sqlalchemy',
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
        'console_scripts': [
            'initialize_tutorial_db = tutorial.initialize_db:main'
        ],
    },
)
We aren't yet doing $VENV/bin/pip install -e . because we need to write a script and update configuration first.
Our configuration file at databases/development.ini wires together some new pieces:

[app:main]
use = egg:tutorial
pyramid.reload_templates = true
pyramid.includes =
    pyramid_debugtoolbar
    pyramid_tm

sqlalchemy.url = sqlite:///%(here)s/sqltutorial.sqlite

[server:main]
use = egg:waitress#main
listen = localhost:6543

# Begin logging configuration

[loggers]
keys = root, tutorial, sqlalchemy.engine.Engine

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

[logger_sqlalchemy.engine.Engine]
level = INFO
handlers =
qualname = sqlalchemy.engine.Engine

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(asctime)s %(levelname)-5.5s [%(name)s][%(threadName)s] %(message)s

# End logging configuration
This engine configuration now needs to be read into the application through changes in databases/tutorial/__init__.py:

from pyramid.config import Configurator

from sqlalchemy import engine_from_config

from .models import DBSession, Base

def main(global_config, **settings):
    engine = engine_from_config(settings, 'sqlalchemy.')
    DBSession.configure(bind=engine)
    Base.metadata.bind = engine

    config = Configurator(settings=settings,
                          root_factory='tutorial.models.Root')
    config.include('pyramid_chameleon')
    config.add_route('wiki_view', '/')
    config.add_route('wikipage_add', '/add')
    config.add_route('wikipage_view', '/{uid}')
    config.add_route('wikipage_edit', '/{uid}/edit')
    config.add_static_view('deform_static', 'deform:static/')
    config.scan('.views')
    return config.make_wsgi_app()
Make a command-line script at databases/tutorial/initialize_db.py to initialize the database:

import os
import sys
import transaction

from sqlalchemy import engine_from_config

from pyramid.paster import (
    get_appsettings,
    setup_logging,
    )

from .models import (
    DBSession,
    Page,
    Base,
    )


def usage(argv):
    cmd = os.path.basename(argv[0])
    print('usage: %s <config_uri>\n'
          '(example: "%s development.ini")' % (cmd, cmd))
    sys.exit(1)


def main(argv=sys.argv):
    if len(argv) != 2:
        usage(argv)
    config_uri = argv[1]
    setup_logging(config_uri)
    settings = get_appsettings(config_uri)
    engine = engine_from_config(settings, 'sqlalchemy.')
    DBSession.configure(bind=engine)
    Base.metadata.create_all(engine)
    with transaction.manager:
        model = Page(title='Root', body='<p>Root</p>')
        DBSession.add(model)
Now that we've got all the pieces ready, and because we changed setup.py, we now install all the goodies:

$VENV/bin/pip install -e .
The script references some models in databases/tutorial/models.py:

from pyramid.authorization import Allow, Everyone

from sqlalchemy import (
    Column,
    Integer,
    Text,
    )

from sqlalchemy.ext.declarative import declarative_base

from sqlalchemy.orm import (
    scoped_session,
    sessionmaker,
    )

from zope.sqlalchemy import register

DBSession = scoped_session(sessionmaker())
register(DBSession)
Base = declarative_base()


class Page(Base):
    __tablename__ = 'wikipages'
    uid = Column(Integer, primary_key=True)
    title = Column(Text, unique=True)
    body = Column(Text)


class Root:
    __acl__ = [(Allow, Everyone, 'view'),
               (Allow, 'group:editors', 'edit')]

    def __init__(self, request):
        pass
Let's run this console script, thus producing our database and table:

$VENV/bin/initialize_tutorial_db development.ini

2016-04-16 13:01:33,055 INFO  [sqlalchemy.engine.Engine][MainThread] SELECT CAST('test plain returns' AS VARCHAR(60)) AS anon_1
2016-04-16 13:01:33,055 INFO  [sqlalchemy.engine.Engine][MainThread] ()
2016-04-16 13:01:33,056 INFO  [sqlalchemy.engine.Engine][MainThread] SELECT CAST('test unicode returns' AS VARCHAR(60)) AS anon_1
2016-04-16 13:01:33,056 INFO  [sqlalchemy.engine.Engine][MainThread] ()
2016-04-16 13:01:33,057 INFO  [sqlalchemy.engine.Engine][MainThread] PRAGMA table_info("wikipages")
2016-04-16 13:01:33,057 INFO  [sqlalchemy.engine.Engine][MainThread] ()
2016-04-16 13:01:33,058 INFO  [sqlalchemy.engine.Engine][MainThread]
CREATE TABLE wikipages (
       uid INTEGER NOT NULL,
       title TEXT,
       body TEXT,
       PRIMARY KEY (uid),
       UNIQUE (title)
)


2016-04-16 13:01:33,058 INFO  [sqlalchemy.engine.Engine][MainThread] ()
2016-04-16 13:01:33,059 INFO  [sqlalchemy.engine.Engine][MainThread] COMMIT
2016-04-16 13:01:33,062 INFO  [sqlalchemy.engine.Engine][MainThread] BEGIN (implicit)
2016-04-16 13:01:33,062 INFO  [sqlalchemy.engine.Engine][MainThread] INSERT INTO wikipages (title, body) VALUES (?, ?)
2016-04-16 13:01:33,063 INFO  [sqlalchemy.engine.Engine][MainThread] ('Root', '<p>Root</p>')
2016-04-16 13:01:33,063 INFO  [sqlalchemy.engine.Engine][MainThread] COMMIT
With our data now driven by SQLAlchemy queries, we need to update our databases/tutorial/views.py:

import colander
import deform.widget

from pyramid.httpexceptions import HTTPFound
from pyramid.view import view_config

from .models import DBSession, Page


class WikiPage(colander.MappingSchema):
    title = colander.SchemaNode(colander.String())
    body = colander.SchemaNode(
        colander.String(),
        widget=deform.widget.RichTextWidget()
    )


class WikiViews:
    def __init__(self, request):
        self.request = request

    @property
    def wiki_form(self):
        schema = WikiPage()
        return deform.Form(schema, buttons=('submit',))

    @property
    def reqts(self):
        return self.wiki_form.get_widget_resources()

    @view_config(route_name='wiki_view', renderer='wiki_view.pt')
    def wiki_view(self):
        pages = DBSession.query(Page).order_by(Page.title)
        return dict(title='Wiki View', pages=pages)

    @view_config(route_name='wikipage_add',
                 renderer='wikipage_addedit.pt')
    def wikipage_add(self):
        form = self.wiki_form.render()

        if 'submit' in self.request.params:
            controls = self.request.POST.items()
            try:
                appstruct = self.wiki_form.validate(controls)
            except deform.ValidationFailure as e:
                # Form is NOT valid
                return dict(form=e.render())

            # Add a new page to the database
            new_title = appstruct['title']
            new_body = appstruct['body']
            DBSession.add(Page(title=new_title, body=new_body))

            # Get the new ID and redirect
            page = DBSession.query(Page).filter_by(title=new_title).one()
            new_uid = page.uid

            url = self.request.route_url('wikipage_view', uid=new_uid)
            return HTTPFound(url)

        return dict(form=form)


    @view_config(route_name='wikipage_view', renderer='wikipage_view.pt')
    def wikipage_view(self):
        uid = int(self.request.matchdict['uid'])
        page = DBSession.query(Page).filter_by(uid=uid).one()
        return dict(page=page)


    @view_config(route_name='wikipage_edit',
                 renderer='wikipage_addedit.pt')
    def wikipage_edit(self):
        uid = int(self.request.matchdict['uid'])
        page = DBSession.query(Page).filter_by(uid=uid).one()

        wiki_form = self.wiki_form

        if 'submit' in self.request.params:
            controls = self.request.POST.items()
            try:
                appstruct = wiki_form.validate(controls)
            except deform.ValidationFailure as e:
                return dict(page=page, form=e.render())

            # Change the content and redirect to the view
            page.title = appstruct['title']
            page.body = appstruct['body']
            url = self.request.route_url('wikipage_view', uid=uid)
            return HTTPFound(url)

        form = self.wiki_form.render(dict(
            uid=page.uid, title=page.title, body=page.body)
        )

        return dict(page=page, form=form)
Our tests in databases/tutorial/tests.py changed to include SQLAlchemy bootstrapping:

import unittest
import transaction

from pyramid import testing


def _initTestingDB():
    from sqlalchemy import create_engine
    from .models import (
        DBSession,
        Page,
        Base
        )
    engine = create_engine('sqlite://')
    Base.metadata.create_all(engine)
    DBSession.configure(bind=engine)
    with transaction.manager:
        model = Page(title='FrontPage', body='This is the front page')
        DBSession.add(model)
    return DBSession


class WikiViewTests(unittest.TestCase):
    def setUp(self):
        self.session = _initTestingDB()
        self.config = testing.setUp()

    def tearDown(self):
        self.session.remove()
        testing.tearDown()

    def test_wiki_view(self):
        from tutorial.views import WikiViews

        request = testing.DummyRequest()
        inst = WikiViews(request)
        response = inst.wiki_view()
        self.assertEqual(response['title'], 'Wiki View')


class WikiFunctionalTests(unittest.TestCase):
    def setUp(self):
        from pyramid.paster import get_app
        app = get_app('development.ini')
        from webtest import TestApp
        self.testapp = TestApp(app)

    def tearDown(self):
        from .models import DBSession
        DBSession.remove()

    def test_it(self):
        res = self.testapp.get('/', status=200)
        self.assertIn(b'Wiki: View', res.body)
        res = self.testapp.get('/add', status=200)
        self.assertIn(b'Add/Edit', res.body)
Run the tests in your package using pytest:

$VENV/bin/pytest tutorial/tests.py -q
..
2 passed in 1.41 seconds
Run your Pyramid application with:

$VENV/bin/pserve development.ini --reload
Open http://localhost:6543/ in a browser.

Analysis
Let's start with the dependencies. We made the decision to use SQLAlchemy to talk to our database. We also, though, installed pyramid_tm and zope.sqlalchemy. Why?

Pyramid has a strong orientation towards support for transactions. Specifically, you can install a transaction manager into your application either as middleware or a Pyramid "tween". Then, just before you return the response, all transaction-aware parts of your application are executed.

This means Pyramid view code usually doesn't manage transactions. If your view code or a template generates an error, the transaction manager aborts the transaction. This is a very liberating way to write code.

The pyramid_tm package provides a "tween" that is configured in the development.ini configuration file. That installs it. We then need a package that makes SQLAlchemy, and thus the RDBMS transaction manager, integrate with the Pyramid transaction manager. That's what zope.sqlalchemy does.

Where do we point at the location on disk for the SQLite file? In the configuration file. This lets consumers of our package change the location in a safe (non-code) way. That is, in configuration. This configuration-oriented approach isn't required in Pyramid; you can still make such statements in your __init__.py or some companion module.

The initialize_tutorial_db is a nice example of framework support. You point your setup at the location of some [console_scripts], and these get generated into your virtual environment's bin directory. Our console script follows the pattern of being fed a configuration file with all the bootstrapping. It then opens SQLAlchemy and creates the root of the wiki, which also makes the SQLite file. Note the with transaction.manager part that puts the work in the scope of a transaction, as we aren't inside a web request where this is done automatically.

The models.py does a little bit of extra work to hook up SQLAlchemy into the Pyramid transaction manager. It then declares the model for a Page.

Our views have changes primarily around replacing our dummy dictionary-of-dictionaries data with proper database support: list the rows, add a row, edit a row, and delete a row.

Extra credit
Why all this code? Why can't I just type two lines and have magic ensue?

Give a try at a button that deletes a wiki page.
