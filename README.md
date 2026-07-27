# PlcTestSuite

A user-friendly unit testing framework for TwinCAT 3 PLC programmers, designed to lower the barriers to Test-Driven Development.

Current version: **v1.1.4** (`Tc3_PlcTestSuite`)

## Installation

1. Download the latest `TestSuite_V1.1.4.library` from the [`Releases`](Releases) folder.
2. In TwinCAT XAE, open `PLC` -> `References` -> right-click -> `Library Repository...` -> `Install...` and select the downloaded file.
3. Right-click `References` -> `Add library...` and pick `Tc3_PlcTestSuite` (company: *Simmel*).

The library references the standard Beckhoff libraries `Tc2_Standard`, `Tc2_System`, `Tc2_Utilities`, `Tc2_EtherCAT`, `Tc3_Module`, `Tc3_EventLogger` and `Tc3_JsonXml`, which are resolved automatically.

After adding the reference, the shared `TestSuite` instance and the shared `JUnitResultFile` are available globally from the `GlobalTestSuite` GVL - no declaration needed to get started.

## Quick Start

### 1. Write Tests in Your MAIN Program

```iecst
PROGRAM MAIN
VAR
    ReRun : BOOL;
END_VAR

TEST_Add();
TEST_Bool();

IF ReRun THEN
    TestSuite.ReRunTests();
    ReRun := FALSE;
END_IF
```

### 2. Add Test Methods

```iecst
METHOD TEST_Add
VAR_INST
    Expected : DINT;
    Actual   : DINT;
END_VAR

IF TestSuite.Test(__POUNAME()).ExecuteTest() THEN
    Expected := 0;
    Actual   := -5 + 5;
    TestSuite.Test(__POUNAME()).AssertEqual(Expected, Actual);
END_IF
```

**Key Points:**
- Use `VAR_INST` for test variables (maintains state between cycles)
- Use `__POUNAME()` for automatic test ID generation
- Wrap logic in `IF TestSuite.Test(__POUNAME()).ExecuteTest() THEN`

### 3. Supported Types

`AssertEqual()` supports integers, booleans, strings, and floating-point (REAL/LREAL) with epsilon tolerance:

```iecst
TestSuite.Test(__POUNAME()).AssertEqual(Expected, Actual, 0.1);  // epsilon = 0.1
```

Default epsilon: `0.000001`

## Configuration

**Max Tests:** Maximum number of tests per suite. Adjustable in library parameter settings (`MaxTestCount`, default: 50)

**Max Test Suites:** Maximum number of distinct test suites that can write to a single result file. Adjustable in library parameter settings (`MaxTestSuiteCount`, default: 10)

**Output Path:** `C:\ProgramData\Beckhoff\TwinCAT\3.1\Boot\TEST_Result.xml` (JUnit XML format)

**Custom Suite Name:** `TestSuite.TestSuiteName := 'My Tests';`

![Library Parameters](https://github.com/user-attachments/assets/8fe82687-2cdc-4290-9789-8bbcd90b384d)

## Multiple Test Suites

You can run more than one test suite and have them all write to the same JUnit result file. The `TestSuite` provided in the `GlobalTestSuite` GVL is a shared instance, but you can also declare your own `TestSuite` in a separate program and inject the shared `JUnitResultFile` so its results are appended to the same output file.

Each suite appears as its own `<testsuite>` element in the XML, so keep the suite names unique (`MaxTestSuiteCount` controls how many distinct suites a single result file can hold).

The following `SecondTestSuite` program declares its own `TestSuite`, injects the shared `JUnitResultFile`, and runs a simple `1 + 1 = 2` test:

```iecst
PROGRAM SecondTestSuite
VAR
    TestSuite : TestSuite<TestSuiteParameter.MaxTestCount>(ResultFile := JUnitResultFile);
    ReRun     : BOOL;
END_VAR

TestSuite.TestSuiteName := 'My Second Test-Suite';
Test_OnePlusOne();

IF ReRun THEN
    TestSuite.ReRunTests();
    ReRun := FALSE;
END_IF
```

```iecst
METHOD Test_OnePlusOne
VAR_INST
    Expected : DINT;
    Actual   : DINT;
END_VAR

IF TestSuite.Test(__POUNAME()).ExecuteTest() THEN
    Expected := 2;
    Actual   := 1 + 1;
    TestSuite.Test(__POUNAME()).AssertEqual(Expected, Actual);
END_IF
```

Call the program from `MAIN` alongside your other tests. Both the global suite and `SecondTestSuite` write to the same file, producing two `<testsuite>` entries in `TEST_Result.xml`.

## Output

Each test suite reports its results two ways: a TwinCAT logger message when it completes, and an entry in the JUnit XML result file.

### TwinCAT Logger Message

```
7/15/2026 2:14:09 PM    842 ms | 'My Test Test-Suite': Completed - Total Tests - 8, OK Tests - 8, Failed Tests - 0
```

This appears in the TwinCAT error list / event logger once a suite's tests have all finished, showing the suite name and a pass/fail summary.

### JUnit XML (`TEST_Result.xml`)

Multiple suites append to the same file as separate `<testsuite>` elements:

```xml
<?xml version="1.0"?>
<testsuites>
    <testsuite name="My Test Test-Suite" tests="8" failures="0" errors="0" skipped="0" time="0.0199599">
        <testcase classname="My Test Test-Suite" name="MAIN.Test_Counter_AddTwo" time="7e-06"></testcase>
        <testcase classname="My Test Test-Suite" name="MAIN.Test_Counter_SubtractTwo" time="6.9e-06"></testcase>
        <testcase classname="My Test Test-Suite" name="MAIN.Test_Bool" time="7.3e-06"></testcase>
        <testcase classname="My Test Test-Suite" name="MAIN.Test_String" time="7e-06"></testcase>
        <testcase classname="My Test Test-Suite" name="MAIN.Test_Lreal" time="7.1e-06"></testcase>
        <testcase classname="My Test Test-Suite" name="MAIN.Test_Real" time="7.1e-06"></testcase>
    </testsuite>
    <testsuite name="My Second Test-Suite" tests="1" failures="0" errors="0" skipped="0" time="0.0199154">
        <testcase classname="My Second Test-Suite" name="SecondTestSuite.Test_OnePlusOne" time="6.7e-06"></testcase>
    </testsuite>
</testsuites>
```

A failed test adds a `<failure>` element with the assertion details:

```xml
<testcase classname="My Test Test-Suite" name="MAIN.Test_Counter_AddTwo" time="7e-06">
    <failure message="Test failed!" type="AssertionError">Expected: 10 , Actual: 9</failure>
</testcase>
```

## Version History

**v1.1.4** - Fixed debug line left in the example `SecondTestSuite` test
**v1.1.3** - Constructor-injected result files, multiple test suites sharing one result file, `MaxTestSuiteCount` limit with overflow handling, fixed `failures` attribute spelling
**v1.1.2** - Added ADS logging functionality
**v1.1.1** - RPC enabled for `ReRunTests()` and `ReRunIndividualTest()` methods
**v1.1.0** - Result file generation (JUnit XML)
**v1.0.9** - Enhanced test infrastructure
**v1.0.8** - Added test suite naming for multiple instances
**v1.0.7** - Message improvements
**v1.0.6** - Code refactoring and naming consistency
**v1.0.5** - Naming adjustments
**v1.0.4** - Added `ReRunIndividualTest()` method
**v1.0.3** - Max tests exceeded handling, individual test suites
**v1.0.2** - REAL/LREAL epsilon comparison, Expected/Actual in failure messages
**v1.0.1** - Library packaging and releases folder
**v1.0.0** - Initial release with core test framework
