# PlcTestSuite

A user-friendly unit testing framework for TwinCAT 3 PLC programmers, designed to lower the barriers to Test-Driven Development.

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

**Max Tests:** Adjustable in library parameter settings (default: 50)

**Output Path:** `C:\ProgramData\Beckhoff\TwinCAT\3.1\Boot\TEST_Result.xml` (JUnit XML format)

**Custom Suite Name:** `TestSuite.TestSuiteName := 'My Tests';`

![Library Parameters](https://github.com/user-attachments/assets/8fe82687-2cdc-4290-9789-8bbcd90b384d)

## Version History

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
