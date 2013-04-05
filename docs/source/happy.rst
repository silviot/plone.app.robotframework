TODO:

- Testing needs Firefox ...
- Testing ...

Write Robot Framework tests for a Plone add-on -- the Happy Path
================================================================

See how we used this for Plomino to run some Robot Tests at SaucesLabs:

- https://github.com/fulv/Plomino/compare/github-main
- https://travis-ci.org/fulv/Plomino
- https://saucelabs.com/u/fulv_plomino

And also a minimal example at:

- https://github.com/datakurre/example.product/tree/p.a.robotframework
- https://travis-ci.org/datakurre/example.product
- https://saucelabs.com/u/exampleproduct

And get started by yourself:


Require plone.app.robotframework
--------------------------------

Update ``setup.py`` to require *plone.app.robotframework*::

    extras_require={
        'test': [
            ...
            'plone.app.robotframework',
        ],
    },

All your need is *plone.app.robotframework*.
It will require the rest (*selenium*, *robotframework*,
*robotframework-selenium2library* and *robotsuite*).


Define functional testing layer
-------------------------------

Plone add-on testing requires defining test layers,
which setup Plone, the add-ons and any custom configuration
required by the tests.

Update your ``src/my/product/testing.py`` to include::

    from plone.testing import z2
    from plone.app.testing import FunctionalTesting

    MY_PRODUCT_ROBOT_TESTING = FunctionalTesting(
        bases=(MY_PRODUCT_FIXTURE, z2.ZSERVER),
        name="MyProduct:Robot")

For example, a functional testing layer definition for Plomino could lool like::

    from plone.app.testing import FunctionalTesting
    from plone.app.robotframework.testing import AUTOLOGIN_LIBRARY_FIXTURE

    PLOMINO_ROBOT_TESTING = FunctionalTesting(
        bases=(AUTOLOGIN_LIBRARY_FIXTURE, PLOMINO_FIXTURE, z2.ZSERVER),
        name="Plomino:Robot")

.. note:: AUTOLOGIN_LIBRARY_FIXTURE is optional, but it will allow you to
   write faster Selenium tests, because tests don't need to spend time on
   login forms.

If you don't have any testing layers for your product yet or want to know
more about them, please, read: http://pypi.python.org/pypi/plone.app.testing


Install Robot-tools
-------------------

*plone.app.robotframework* ships with two main helper scripts for tests
writing:

* ``bin/robot-server`` starts a temporary Plone site with the given
  test layer set up

* ``bin/robot`` executes Robot Framework's ``pybot``-runner so that it
  will run the given test sure agains the running ``robot-server`` so
  that tests will be run in isolation (database is cleaned between the
  tests)

Update ``buildout.cfg``::

    [buildout]
    parts =
        ...
        robot

    [robot]
    recipe = zc.recipe.egg
    eggs =
        Pillow
        ${test:eggs}
        plone.app.robotframework[ride,reload]

.. note:: Robot-tools are optional, but will ease and speed up test
   development. [reload]-extras will make ``robot-server`` detect
   filesystem changes under ``./src`` and reload the test layer when
   a change is detected. [ride]-extras will create a script to start
   RIDE, the IDE for Robot Framework, but it can be launched only
   explicitly with a compatible system python with wxPython 2.8.x
   installed.


Start test server
-----------------

Once the buildout with Robot-tools is run, start the test server with::

    $ bin/robot-server my.product.testing.MY_PRODUCT_ROBOT_TESTING

This is how the test layer is started with the example Plomino testing
layer::

    $ bin/robot-server Products.CMFPlomino.testing.PLOMINO_ROBOT_TESTING

Once the test server has started, there's a test Plone-site served
at http://localhost:55001/plone/.

.. note:: Technically ``robot-server`` only duplicates some existing
   magic from ``zope.testrunner`` to figure out all the required test
   layers and set them up in the required order.



Write the first test
--------------------

Robot tests are written in test suites, which are plain text files, usually
ending with ``.robot`` (and older ones with ``.txt``).

The first test can be written anywhere in the filestystem.

For example, a ``test_hello.robot``::

    *** Settings ***

    Resource  plone/app/robotframework/selenium.robot

    Test Setup  Open test browser
    Test Teardown  Close all browsers

    *** Test Cases ***

    Plone is installed
        Go to  ${PLONE_URL}
        Page should contain  Powered by Plone

Robot is all about running test clauses called **keywords**.
Every test case may contain one more keywords, which are run in serial --
usually until first of them fails.

Keywords are defined in **keywords libraries** and as **user keywords**.
Keyword libraries can be Python libraries or XML-RPC-services.
User keywords are lists of test clauses reusing existing keywords.
User keywords may also re-use other user keywords.

Here is a more complicated example for Plomino::

    *** Settings ***

    Resource  plone/app/robotframework/selenium.robot

    Library  Remote  ${PLONE_URL}/RobotRemote

    Test Setup  Open test browser
    Test Teardown  Close all browsers

    *** Test Cases ***

    Plomino is installed
        Go to  ${PLONE_URL}
        Page should contain  mydb

    Form can be created
        Log in as the database owner
        Open the database
        Create test form

    *** Keywords ***

    Log in as the database owner
        Enable autologin as  Site Administrator
        Set autologin username  ${TEST_USER_ID}
        Go to  ${PLONE_URL}

    Open the database
        Go to  ${PLONE_URL}/mydb

    Create test form
        Click link  Form
        Page should contain element  css=input#id
        Input text  id  frm_test
        Input text  title  Test Form
        Click button  Save
        Page should contain  Changes saved.

.. note:: We use ``.robot`` as Robot Framework test suite
   file extension to make it easier for developers to
   configure Robot Framework syntax highlighting
   for their editors (otherwise ``.txt`` would work also).


Run the first test
------------------

Once the ``bin/robot-server`` has been started
and a test suite has been written,
the test suite can be run with ``bin/robot``::

    $ bin/robot test_plomino.robot


.. note::: ``bin/robot`` is a wrapper for Robot Framework's
   pybot test runner to  support plone.testing's test isolation
   for Plone when used with together with bin/robot-server.


Integrate to Zope-testrunner
----------------------------

Because it's convenient to run Robot tests with other tests
by zope.testrunner e.g. on Travis-CI, we usually want to integrate
robot test to be run with other tests using *zope.testrunner*.

For *zope.testrunner* integration, create
``src/my/product/tests/test_robot.py`::

    import unittest

    import robotsuite
    from my.product.testing import MY_PRODUCT_ROBOT_TESTING
    from plone.testing import layered


    def test_suite():
        suite = unittest.TestSuite()
        suite.addTests([
            layered(robotsuite.RobotTestSuite('test_hello.robot'),
                    layer=MY_PRODUCT_ROBOT_TESTING),
        ])
        return suite

For Plomino, we created ``Products/CMFPlomino/tests/test_robot.py``::

    import unittest

    import robotsuite
    from Products.CMFPlomino.testing import PLOMINO_ROBOT_TESTING
    from plone.testing import layered


    def test_suite():
        suite = unittest.TestSuite()
        suite.addTests([
            layered(robotsuite.RobotTestSuite('test_plomino.robot'),
                    layer=PLOMINO_ROBOT_TESTING),
        ])
        return suite

.. notes:: For this to work and ``zope.testrunner`` to discover your
   robot test suite, remember to move ``test_plomino.robot`` under
   ``Products/CMFPlomino/tests``.

It's good to know that this pattern is same how doctest suites are registered
(e.g. in https://pypi.python.org/pypi/plone.testing) to use layer.
Also, RobotSuite is a Collective-package, which only purpose is to wrap
Robot Framework tests to be Python unittest compatible.


Integrate to Sauce Labs
-----------------------

1. Register account for http://saucelabs.com/ with Open Sauce -plan.
   Derive username from product name. For example, ``myproduct`` or
   ``plomino``. User your own contact email for the beginning.
   It can be changed later.

2. Update ``.travis.yml`` to set up the Sauce Labs connection before tests::

       ---
       language: python
       python: '2.7'
       install:
       - mkdir -p buildout-cache/downloads
       - python bootstrap.py -c travis.cfg
       - bin/buildout -N -t 3 -c travis.cfg
       - curl -O http://saucelabs.com/downloads/Sauce-Connect-latest.zip
       - unzip Sauce-Connect-latest.zip
       - java -jar Sauce-Connect.jar $SAUCE_USERNAME $SAUCE_ACCESS_KEY -i $TRAVIS_JOB_ID -f CONNECTED &
       - JAVA_PID=$!
       before_script:
       - bash -c "while [ ! -f CONNECTED ]; do sleep 2; done"
       script: bin/test
       after_script:
       - kill $JAVA_PID
       env:
         global:
         - ROBOT_BUILD_NUMBER=travis-$TRAVIS_BUILD_NUMBER
         - ROBOT_REMOTE_URL=http://$SAUCE_USERNAME:$SAUCE_ACCESS_KEY@ondemand.saucelabs.com:80/wd/hub
         - ROBOT_DESIRED_CAPABILITIES=tunnel-identifier:$TRAVIS_JOB_ID

And this is an example we came up for Plomino::

       language: python
       python:
         - 2.7
       install:
         - mkdir -p buildout-cache/downloads
         - python bootstrap.py -c travis.cfg -v 1.7.1
         - bin/buildout -N -t 3 -c travis.cfg install download install
         - bin/buildout -N -t 3 -c travis.cfg
         - curl -O http://saucelabs.com/downloads/Sauce-Connect-latest.zip
         - unzip Sauce-Connect-latest.zip
         - java -jar Sauce-Connect.jar $SAUCE_USERNAME $SAUCE_ACCESS_KEY -i $TRAVIS_JOB_ID -f CONNECTED &
         - JAVA_PID=$!
       before_script:
         - bash -c "while [ ! -f CONNECTED ]; do sleep 2; done"
         - "export DISPLAY=:99.0"
         - "sh -e /etc/init.d/xvfb start"
       script:
        - bin/test --coverage -m Products.CMFPlomino
       env:
         global:
           - ROBOT_BUILD_NUMBER=travis-$TRAVIS_BUILD_NUMBER
           - ROBOT_REMOTE_URL=http://$SAUCE_USERNAME:$SAUCE_ACCESS_KEY@ondemand.saucelabs.com:80/wd/hub
           - ROBOT_DESIRED_CAPABILITIES=tunnel-identifier:$TRAVIS_JOB_ID

3. Install travis-gem for you Ruby active Ruby-installation::

       $ sudo gem install travis

4. Log in to Sauce Labs to see your Sauce Labs access key.

5. Encrypt Sauce Labs credentials into ``.travis.yml``::

       $ travis encrypt SAUCE_USERNAME=myusername -r mygithubname/myproduct --add env.global
       $ travis encrypt SAUCE_ACCESS_KEY=myaccesskey -r mygithubname/myproduct --add env.global

6. Update the test to use SauceLabs test browser::

       *** Settings ***

       ...
       Resource  plone/app/robotframework/saucelabs.robot

       Test Setup  Open SauceLabs test browser
       Test Teardown  Run keywords  Report test status  Close all browsers

       ...

7. Update ``travis.cfg`` to allow downloading robotframework-packages::

       [buildout]
       ...
       allow-hosts +=
           code.google.com
           robotframework.googlecode.com

.. note:: If you don't have Travis-CI-integration yet, you need to add ``travis.cfg``
   for the above ``.travis.yml`` to work::

       [buildout]
       extends = https://raw.github.com/collective/buildout.plonetest/master/travis-4.x.cfg

       package-name = my.product
       package-extras = [test]

       allow-hosts +=
           code.google.com
           robotframework.googlecode.com

       [environment]
       ZSERVER_HOST = 0.0.0.0
       ROBOT_ZOPE_HOST = 0.0.0.0

       [test]
       environment = environment

    The *environment*-part and line in *test*-part are optional, but are required to
    run tests using Internet Explorer (via SauceLabs) later.


How to write more tests
-----------------------

The most difficult parts in writing robot tests with Selenium-keywords is to know
the application you are testing: which link to click when and to which field to
input test data.

Robot Framework ships with a few selected standard libraries. One of them is
*Dialogs*-library, which provides a very usable keyword: *Pause execution*. By
importing Dialogs-library (while developing the test) and adding *Pause execution*
keyword, you can pause the test at any point to make it possible to figure out
what to do next.

For example::

    *** Settings ***

    Resource  plone/app/robotframework/selenium.robot
    Resource  plone/app/robotframework/saucelabs.robot

    Library  Remote  ${PLONE_URL}/RobotRemote
    Library  Dialogs

    Test Setup  Open SauceLabs test browser
    Test Teardown  Run keywords  Report test status  Close all browsers

    *** Test Cases ***

    Plomino is installed
        Go to  ${PLONE_URL}
        Pages should contain  mydb

    Let me think what to do next
        Enable autologin as  Site Administrator
        Go to  ${PLONE_URL}
        Pause execution

.. note:: Be sure to remove Dialogs-library import and its keywords
   before commit, because Dialogs-library may have dependencies,
   which are not available on your CI-machine.


Resources
---------

- http://robotframework.googlecode.com/hg/doc/libraries/BuiltIn.html?r=2.7.7
- http://rtomac.github.com/robotframework-selenium2library/doc/Selenium2Library.html
- http://code.google.com/p/robotframework/wiki/HowToWriteGoodTestCases
- http://code.google.com/p/robotframework/


Examples:
---------

- https://github.com/plone/plone.app.robotframework/tree/master/src/plone/app/robotframework/tests
- http://plone.293351.n2.nabble.com/Robot-Framework-How-to-fill-TinyMCE-s-text-field-tp7563662p7563691.html
