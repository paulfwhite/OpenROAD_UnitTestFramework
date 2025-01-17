# OpenROAD UnitTest Framework

## Overview

The OpenROAD unit testing framework is a OpenROAD language version of JUnit, by smart cookies Kent Beck and Erich Gamma. JUnit is, in turn, a Java version of Kent's Smalltalk testing framework. Each is the de facto standard unit testing framework for its respective language.

This document explains the OpenROAD-specific aspects of the design and usage of the OpenROAD UnitTest Framework.
For background information on the basic design of the framework the reader is referred to Kent's original paper, "Simple Smalltalk Testing: With Patterns".

The following information assumes knowledge of OpenROAD.

## System requirements

Getting Started:
  * OpenROAD Developer version 6.2 or later
  * Ingres Net client  (for example Ingres 10.2 + latest patch) to match the version of OpenROAD
  * OpenROAD Repository database (your application code)

Automation. To use the supplied scripts you may also need:
  * Cygwin (from https://www.cygwin.com/)
  * bash (some bash extensions have been used)
  * diffutils (diff) for example https://www.gnu.org/software/diffutils/diffutils.html
  * Ingres Terminal Monitor or SQL command line utility
  * Scheduling tool such as cron or taskschd

## Using the OpenROAD UnitTest Framework to write tests

### Installation

The components needed to write and run tests can be found in the 'UnitTestFramework' application.

To be able to use it from your own code, import the `UnitTestFramework` application from the `UnitTestFramework.xml` file into your repository database. Then you can use this application as an included application in your own application.

Note that you will have to do this before you can run the examples that are provided with the OpenROAD UnitTest Framework unless you run one of the command line scripts (or_tests.bash or or_tests.bat), which will import the 'UnitTestFramework.xml' file before importing and executing other test applications.

### The Framework

The framework is very simple, it consists of:

  * A set of user classes similar to JUnit (Test, TestCase, Assert, Error, etc.)
  * A set of global variables (G_Error to reference the error of the last "Exception", G_Assert to reference the Assert object providing assertion methods, G_Helper to provide Helper functionally)
  * The TestRunnerGhost frame, that invokes the "run" method on the test suite referenced in an attribute of the Helper object (G_Helper global variable), and writes out test results and statistics/timings.
  * The 4GL procedure "executeTests" that adds a test to the test suite in the Helper object, which will start the TestRunnerGhost frame (as independent thread).
  * The 3GL Procedure GetTickCount() - used for timing calculation

### Environment variables used by the OpenROAD UnitTest Framework

The following environment variables are used when running tests using the framework:

  * `OR_UNITTEST_TESTDIR` Specifies a directory used for testing input/output. If not set it defaults to the current working directory. This directory must be writeable by the user.
  * `OR_UNITTEST_GEN_XML_STATS` Specifies if test statistics should be written in JUnit XML format. Valid values TRUE and FALSE (default).
  * `OR_UNITTEST_STATSFILE` Specifies the file (including path) the test statistics will be written to. If not set it defaults to `orunitteststats.log` in the directory specified by the `OR_UNITTEST_TESTDIR` environment variable.
  * `OR_UNITTEST_STATSFILE_XML` Specifies the file (including path) the XML test statistics will be written to. If not set it defaults to `orunitteststats.xml` in the directory specified by the `OR_UNITTEST_TESTDIR` environment variable. It is only used if `OR_UNITTEST_GEN_XML_STATS=TRUE`.
  * `OR_UNITTEST_TMPDIR` Specifies a directory used for temporary files used by the test cases. If not set it defaults to to a directory named `temp` in the directory specified by the `OR_UNITTEST_TESTDIR` environment variable.
  If this does not exist, a `testtemp` directory in the current working directory will be used.
  If this doesn't exist either, the following envronment variables will be checked (in this order): `II_TEMPORARY`, `TMPDIR`, `TEMP`, `TMP`.
  If they are not set the following directories are checked (in this order):
  On Unix: `/tmp`, `/var/tmp`, `/usr/tmp`
  On Windows: `C:\TEMP`, `C:\TMP`, the `TEMP` and `TMP` directories under the current working directory
  If none of the above exists, then the current working directory will be used.
  * `OR_UNITTEST_RESOURCEDIR` Specifies a directory containing additional resources (files) used for the tests. If not set it defaults to to a directory named `resources` in the directory specified by the `OR_UNITTEST_TESTDIR` environment variable.
  If this does not exist, a `resources` directory in the current working directory will be used.
  If this doesn't exist either, the temporary directory (see `OR_UNITTEST_TMPDIR` above) will be used.

### Test Cases

The basic building blocks of unit testing are "test cases", which represent scenarios that must be set up and checked for correctness. In the OpenROAD UnitTest Framework, test cases are represented by the TestCase class in the UnitTestFramework application. To make your own test cases you must create subclasses of TestCase.

An instance of a TestCase class is an object that can run test methods, together with optional set-up and tidy-up code.

The testing code of a TestCase instance should be entirely self contained and [idempotent](https://en.wikipedia.org/wiki/Idempotence), such that it can be run either in isolation or in arbitrary combination with any number of other test cases.

### Creating a simple test case

  1. Create a new application:
       1. Include application `UnitTestFramework` in the new application.
  2. Within the application create a new User Class:
       1. Make sure the superclass is `testcase` (from the `UnitTestFramework` application).
       2. Create a new test method in the Userclass editor, make sure the method's name starts with `test`.
       3. Create the code for the test method in the Script of the class.
       
         Within the test\* methods you execute the methods/procedures to be tested (add other components needed within the test methods to the application).
         In order to test something, use one of the 'assert\*' methods of the Assert class (using G_Assert global variable). If the assertion fails when the test case runs, an AssertionError will be raised, and the testing framework will identify the test case as a 'failure'. Other exceptions that do not arise from explicit 'assert' checks are identified by the testing framework as 'errors'.

         Example script:

```
         METHOD test1()=
         {
             G_Assert.assertEquals(expectedVarchar='ingres', actualVarchar=CurSession.UserName, errortext = 'This test has to be run by user "ingres"!');
         }
```
  3. Create 4gl procedure named `runtests`
  
      1. Ensure `runtests` calls the 4GL procedure `executeTests`with parameter `test` set to an object of the TestCase userclass above

         Example script:

```
         PROCEDURE runtests()=
         {
             CALLPROC executeTests(test = MyTest.Create());
         }
```

      2. Ensure `runtests` is the starting component for the application
  4. Run your test application - results will be written to the log file (and Trace Window if it exists)


### Re-using set-up and tear-down code

Now, such test cases can be numerous, and their set-up can be repetitive.

In order to prevent duplication such set-up code can be factored out by overriding a method called `setUp()`, which the testing framework will automatically invoke before the test method(s) run.
Similarly, tear-down code can be factored out by overriding a method called `tearDown()`, which the testing framework will automatically invoke after all test method(s) ran in order to tidy-up.

Example script:
```
    INITIALIZE=
    DECLARE
        table_names = ARRAY OF StringObject;
        i = INTEGER NOT NULL;
    ENDDDECLARE
     
    METHOD setUp()=
    {
        i=1;
        // Populate the table_name array once - it will be used in several test methods
        SELECT table_name as table_names[i].Value FROM iitables WHERE owner = _dba()
        BEGIN
            i=i+1;
        END;
        COMMIT;
    }
```

### TestCase classes with several test methods

A TestCase subclass can have several test methods, all of them having a name starting with `test`.
All these methods will be executed when running the test.
The methods are invoked in alphabetical order (case-insensitive).

### Applications with several test classes

When having multiple test classes in a test application,
they can all be executed by adding a call to `executeTests()` for each of them within the in the `runtests()` procedure.

### Skipping tests

A test can be skipped by invoking the `skipTest()` method (using the G_Assert global variable).
This can be done in either the `setUp()` method (which will skip all tests in the class) or in each test method.

Example script:

```
    METHOD Setup()=
    BEGIN
        IF CurSession.OperatingSystem = SY_UNIX THEN
            G_Assert.SkipTest(errortext = 'The features to be tested are not supported on Unix.');
        ENDIF;
    END
```

### Where to place testing code

You can place the definitions of test cases and test suites in the same application as the code they are to test,
but there are several advantages to placing the test code in a separate application:

* The shipped code does not need to include the UnitTestFramework application
* The test application can be run standalone from the command line
* The test code can more easily be separated from shipped code
* There is less temptation to change test code to fit the code it tests without a good reason
* Test code should be modified much less frequently than the code it tests
* Tested code can be refactored more easily
* If the testing strategy changes, there is no need to change the source code 

### Test Results
          
Test results (status, timings, etc.) are logged in the log file (and trace window).
You can also run the application from command line and test the return status of the application.
The exit codes have the following meaning:

Code | Explanation
---- | -----------
0 | Success (OK)
1 | Error/failure
2 | Skipped tests (but otherwise ok)

### Running tests interactively

A test application can be run interactively from the OpenROAD Workbench using a "Run" menu item or toolbar button.

### Running tests from the command line

In order to run tests from the command line you can use the following command:

w4gldev rundbapp _database testapplication_ -Llogfile -Tyes

There is also a script file `or_tests.bash` (and wrapper batch script `or_tests.bat` for Windows calling `or_tests.bash`),
which will execute all test case applications which have been placed as XML export files in the `unittests` subdirectory.
This script will import the UnitTestFramework application from the 'UnitTestFramework.xml' file before importing and executing other test applications in the `unittests` subdirectory.

Usage:    bash or_tests.bash _testdatabase_

The script will also print out the test results to the console.

### Configuring tests run by or_tests.bash

There are configuration options for tests that are run automated by using or_tests.bash script described above.

#### Ignoring tests

If there is a file called `ignore_apps.lst` in the `unittests` subdirectory, then the applications listed in this file will be ignored.
That is, the _application_ will not be executed.
These tests will be indicated with an `IGNORED` status in the output of or_tests.bash.

#### Configuring tests

Tests for an application can be configured using an optional _application_.cfg file in the `unittests` subdirectory.
The file can contain lines in the format

OPTION=value

with the following meaning:

OPTION | Explanation
------ | -----------
RUNNER | Specifies the command used to execute the test application after importing. This could also be a (batch) script. Default: `w4gldev rundbapp ...`; If set to `runimage` the application will be imaged using makeimage before running using "w4gldev rundbapp"
RUNTIMEOUT | Specifies the timeout period in secs when running the test application - default value: 300. After the timeout period the process running the test will be aborted.
RUNFLAGS | Specifies additional command line flags passed to the RUNNER.
MAKEIMAGEFLAGS | Specifies additional flags used by the makeimage command, if RUNNER is set to `runimage`.

### The GUI test runner

There is a graphical front end that you can use in order to run your tests.
To use the GUI test runner, simply run the `UnitTestRunner` application (can be imported from `UnitTestRunner.xml`)
either from the Workbench or from command line.

### Assertion methods

There are several methods of the Assert class (available via the G_Assert global variable), which can be used to test different conditions,
usually to compare an actual value with an expected value:
* assertEquals()
* assertGreaterEquals()
* assertGreaterThan()
* assertLessEquals()
* assertLessThan()
* assertNotEquals()
* assertNotNull()
* assertNotSame()
* assertNull()
* assertSame()

### Helper attributes and methods

The Helper class (available via the G_Helper global variable) provides useful attributes and methods that can be used within the test methods:

Attribute | Explanation
--------- | -----------
DirectorySeparator | The OS specific directory separator
ResourceDir | The directory containing resource files - see description of the `OR_UNITTEST_RESOURCEDIR` environment variable
TempDir | The directory containing temporary files - see description of the `OR_UNITTEST_TMPDIR` environment variable
TestDir | The directory containing test files - see description of the `OR_UNITTEST_TESTDIR` environment variable
TestStatsFile | The file test statistics will be written to - see description of the `OR_UNITTEST_STATSFILE` environment variable
TestStatsFileXml | The XML file test statistics will be written to - see description of the `OR_UNITTEST_STATSFILE_XML` environment variable

Method              | Explanation
------------------- | -----------
GetResourceFilePath | Returns a file path in the ResourceDir created from a `filename` in portable format (using "!" directory separator)
GetTempFilePath     | Returns a file path in the TempDir created from a `filename` in portable format (using "!" directory separator); optional `register_name` parameter specifies if the filename should be registered for deletion using RemoveRegisteredTempFile() or RemoveRegisteredTempFiles()
LocalFilename       | Returns a file name in local (OS specific) format created from a `filename` in portable format (using "!" directory separator)
RemoveFile          | Removes file with the given `filename` in local (OS specific) format
RemoveRegisteredTempFile | Removes registered file with `filename` in portable format
RemoveRegisteredTempFiles | Removes all registered files

## Terms of use

See the LICENSE file.

