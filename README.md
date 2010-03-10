nodeunit
========

nodeunit offers easy unit testing based on a simplified version of the
[QUnit](http://docs.jquery.com/QUnit) API. However, unlike QUnit, it assumes
all your tests are asynchronous, and plays nicely with the existing module
system. Because of these assumptions, the already minimal API offered by QUnit
can be further reduced.


Usage
-----

Here is an example unit test module:

    exports.testSomething = function(test){
        test.expect(1);
        test.ok(true, "this assertion should pass");
        test.done();
    };

    exports.testSomethingElse = function(test){
        test.ok(false, "this assertion should fail");
        test.done();
    };

Tests are simply exported from the module, but they are still run in the order
they are defined. The module() call from QUnit can be omitted, since it is
inside a module file, and we can refer to it by filename.

One feature you might expect from a unit test framework is the ability to setup
an envrionment for tests to run in. In QUnit this is done as an argument to the
module() function. However, nodeunit avoids implementing this functionality
since running tests with a specific context is fairly simple in javascript:

    var setup = function(fn){
        return function(test){
            fn.call({hello: "world"}, test);
        }
    };

    exports.testSetup = setup(function(test){
        test.equals(this.hello, "world", "test run with env");
        test.done();
    });

If you wanted the setup to occur when the test is run (instead of when the
module is loaded), you could have setup() return a continuation instead.


API Documentation
-----------------

* __expect(amount)__ - Specify how many assertions are expected to run within a
  test.
* __ok(state, message)__ - A boolean assertion, equivalent to [assert.ok](http://nodejs.org/api.html#_assert_module)
* __equals(actual, expected, message)__ - A comparison assertion, equivalent
  to [assert.equal](http://nodejs.org/api.html#_assert_module)
* __same(actual, expected, message)__ - A deep recursive comparison, equivalent
  to [assert.deepEquals](http://nodejs.org/api.html#_assert_module)
* __done()__ - Finish this test function, and move on to the next. ALL tests
  should call this!

These 5 functions are all you need to know!


Asynchronous Testing
--------------------

Testing asyncronous functions in nodeunit _just works_ for the most part, since
tests are only considered complete once test.done() is called and each test
gets its own object with assertion methods to use. This allows nodeunit
to track which assertions comes from which test functions.

Despite this, there are still some gotacha's when testing asynchronous code.
Be sure to use the test.expect() method at the start of your tests to make
sure callbacks actually fired and assertions were run. Otherwise, through not
calling the callbacks you expect, a test may accidentally pass.


Running Tests
-------------

nodeunit comes with a basic command-line test runner, which exists in the lib
folder. Example usage:

    node nodeunit/lib/testrunner.js testmodule1.js testfolder [...]

You can also add some code to the bottom of your test modules so they can be
run directly from the command-line:

    if(module.id == '.'){
        var testrunner = require('nodeunit').testrunner;
        testrunner.run([__filename]);
    }

NOTE: this requires nodeunit to be in your require paths. You can make nodeunit
available to all your projects by copying it to ~/.node-libraries


Adding nodeunit to Your Projects
--------------------------------

Usually, you'll want to create a script for running the tests for your project
with the correct require paths set up. Here's an example test script, with
deps, lib and test directories:

    #!/usr/local/bin/node

    require.paths.push(__dirname);
    require.paths.push(__dirname + '/deps');
    require.paths.push(__dirname + '/libs');
    var testrunner = require('nodeunit').testrunner;

    process.chdir(__dirname);
    testrunner.run(['test']);

If you're using git, you might find it useful to include nodeunit as a
submodule, making it easy to download for users wanting to run your project's
test suite. To add nodeunit as a git submodule do the following:

    git submodule add git://github.com/caolan/nodeunit.git deps/nodeunit

This will add nodeunit to the deps folder of your project. Now, when cloning
the repositiory, nodeunit can be downloaded by doing the following:

    git submodule init
    git submodule update

Let's update the test script above with a helpful hint on how to get nodeunit,
if its missing:

    #!/usr/local/bin/node

    require.paths.push(__dirname);
    require.paths.push(__dirname + '/deps');
    require.paths.push(__dirname + '/libs');

    try {
        var testrunner = require('nodeunit').testrunner;
    }
    catch(e) {
        var sys = require('sys');
        sys.puts("Cannot find nodeunit module.");
        sys.puts("You can download submodules for this project by doing:");
        sys.puts("");
        sys.puts("    git submodule init");
        sys.puts("    git submodule update");
        sys.puts("");
        process.exit();
    }

    process.chdir(__dirname);
    testrunner.run(['test']);


Writing a Test Runner
---------------------

nodeunit exports runTest(fn, options), runModule(mod, options) and
runFiles(paths, options). You'll most likely want to run test suites from
files, which can be done using the latter function. The _options_ argument can
contain callbacks which run during testing. nodeunit provides the following
callbacks:

* __moduleStart(name)__ - called before a module is tested
* __moduleDone(failures, total)__ - called once all test functions within the
  module have completed, with the failed and total number of assertions across
  ALL tests within the module
* __testStart(name)__ - called before a test function is run
* __testDone(failures, total)__ - called once a test function has completed
  (by calling test.done()), with the number of failed and total assertions
  which were run
* __log(assertion)__ - called whenever an assertion is made (see assertion
  object reference below)
* __done(failures, total)__ - called after all tests/modules are complete, with
  the number of failed and total assertions across all test/modules

The __assertion__ object:
* passed() - did the assertion pass?
* failed() - did the assertion fail?
* error - the AssertionError if the assertion failed
* method - the nodeunit assertion method used (ok, same, equals...)
* message - the message the assertion method was called with (optional)

For a reference implementation of a test runner, see lib/testrunner.js in the
nodeunit project directory.


Running the nodeunit Tests
--------------------------

The tests for nodeunit itself use the old-fashioned assert module. Even though
writing tests for nodeunit in nodeunit syntax would be all cool and meta, this
approach doesn't lend itself to TDD and easy debugging ;).

To run the nodeunit tests do: node test.js