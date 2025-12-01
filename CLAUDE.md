# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PlcTestSuite is a user-friendly unit testing framework for TwinCAT 3 PLC programmers, designed to lower barriers to Test-Driven Development (TDD). The project is structured as a TwinCAT library that can be installed into PLC projects to enable unit testing with JUnit XML output.

## TwinCAT Project Structure

This is a TwinCAT 3 project with the following key components:

- **Solution**: `PlcTestSuite/TestSuite Project.sln`
- **TwinCAT Project**: `PlcTestSuite/TwinCAT Project1/TestSuite Project.tsproj`
- **Test Framework Library**: `PlcTestSuite/TwinCAT Project1/TestSuite/TestSuite.plcproj`
- **Example PLC Project**: `PlcTestSuite/TwinCAT Project1/PLC/PLC.plcproj`

### File Types

- `.TcPOU` - Program Organization Units (function blocks, programs, methods)
- `.TcIO` - Interface definitions
- `.TcDUT` - Data Unit Types (structures, enums)
- `.TcGVL` - Global Variable Lists
- `.tmc` - Type Management Component files
- `.library` - Compiled library releases (in `Releases/` folder)

## Core Architecture

### Test Framework Components

The test framework uses an interface-based architecture with the following core components:

**Interfaces** (`TestSuite/Interfaces/`):
- `I_TestSuite` - Main test suite interface with methods `Test()`, `ReRunTests()`, `ReRunIndividualTest()`, and property `Duration`
- `I_TestOperation` - Individual test operations with methods `ExecuteTest()`, `AssertEqual()` and properties `State`, `TestId`, `Duration`, `TestSuiteName`
- `I_UnitTestResultFile` - Result file generation interface with methods for initializing, adding test results, and saving (currently implemented by `JUnitResultFile`)

**Core Function Blocks** (`TestSuite/`):
- `TestSuite.TcPOU` - Main test suite manager that coordinates test execution, tracks test states, generates completion messages, and writes results
- `Test.TcPOU` - Individual test implementation with assertion logic supporting multiple data types (strings, integers, floating-point with epsilon comparison)

**Supporting Components**:
- `JUnitResultFile.TcPOU` - Generates JUnit XML format test results using `FB_XmlDomParser`
- `Stopwatch.TcPOU` - High-precision timing using `F_GetActualDcTime64()` for test duration measurement
- `E_TestStatus.TcDUT` - Test state enumeration (Init, Running, DoneOk, DoneFailed)

**Configuration**:
- `TestSuiteParameter.TcGVL` - Library parameters including `MaxTestCount` (default: 50) and `DefaultPath` for XML output (`C:\ProgramData\Beckhoff\TwinCAT\3.1\Boot\TEST_Result.xml`)

### Test Execution Flow

1. Tests are registered on first call by passing `__POUNAME()` to `TestSuite.Test()`
2. Each test calls `ExecuteTest()` which returns TRUE when the test should run
3. Test logic executes and calls `AssertEqual(Expected, Actual, Epsilon)` to validate results
4. Test state transitions: Init -> Running -> DoneOk/DoneFailed
5. When all tests complete, statistics are calculated and results written to XML file
6. Tests can be re-run individually via `ReRunIndividualTest(TestId)` or all tests via `ReRunTests()`

## Writing Tests

Tests are written as methods in the PLC MAIN program (or any program):

```iecst
METHOD Test_Counter_AddTwo
VAR_INST
    Expected : DINT;
    Actual   : DINT;
END_VAR

IF TestSuite.Test(__POUNAME()).ExecuteTest() THEN
    Expected := 10;
    Actual   := Counter.AddTwo(5, 5);
    TestSuite.Test(__POUNAME()).AssertEqual(Expected, Actual);
END_IF
```

Each test method must:
1. Use `VAR_INST` for Expected/Actual variables to maintain state between cycles
2. Guard test execution with `IF TestSuite.Test(__POUNAME()).ExecuteTest() THEN`
3. Use `__POUNAME()` to generate unique test IDs automatically
4. Call `AssertEqual()` on the same test instance to validate results

The test methods must be called from the program body:

```iecst
PROGRAM MAIN
VAR
    ReRun : BOOL;
END_VAR

Test_Counter_AddTwo();
Test_Counter_SubtractTwo();

IF ReRun THEN
    TestSuite.ReRunTests();
    ReRun := FALSE;
END_IF
```

## Supported Assertion Types

`AssertEqual()` supports comparison of:
- Integer types (DINT, UDINT, etc.) - byte-by-byte comparison
- Boolean types - byte-by-byte comparison
- String types (STRING, WSTRING, etc.) - string comparison
- REAL types - floating-point comparison with epsilon (default: 0.000001)
- LREAL types - floating-point comparison with epsilon (default: 0.000001)
- Any other types - byte-by-byte comparison via MEMCPY

Floating-point comparisons use epsilon tolerance: value must be within (Expected ± Epsilon).

## Result File Output

Test results are written to JUnit XML format by default. The output location is configurable via library parameters:
- Default path: `C:\ProgramData\Beckhoff\TwinCAT\3.1\Boot\TEST_Result.xml`
- Modify in library parameter settings (max 50 tests default)

Alternative result file implementations can be created by implementing `I_UnitTestResultFile` and assigning via `TestSuite.ResultFileFunction` property.

## Key Implementation Details

- Tests run sequentially, one per cycle, controlled by the `ActiveTest` index in `TestSuite`
- Each test gets a unique stopwatch for duration tracking
- Test completion triggers TwinCAT event messages with statistics
- The framework uses TwinCAT's `TwinCAT_SystemInfoVarList._TaskInfo[GETCURTASKINDEXEX()].CycleCount` to detect new PLC cycles
- Result file writes are triggered on completion but executed asynchronously via `Cyclic()` method
- The test collection is a fixed-size array configured by `TestSuiteParameter.MaxTestCount`

## Common Development Tasks

When modifying the test framework:
- Always maintain backward compatibility with existing test methods
- Preserve the interface contracts (`I_TestSuite`, `I_TestOperation`, `I_UnitTestResultFile`)
- Test state machine logic is critical - maintain Init -> Running -> Done transitions
- When adding new assertion types, update `Test.AssertEqual()` method
- Library releases are stored in `Releases/` folder as `.library` files

## Project Context

The current branch `JunitResultFile` shows active work on the JUnit XML result file generation feature. Recent changes involve:
- Modifications to `I_UnitTestResultFile.TcIO` interface
- Updates to `JUnitResultFile.TcPOU` implementation
- Changes to `TestSuite.TcPOU` to integrate result file functionality
- Addition of functions under `TestSuite/Functions/` directory
