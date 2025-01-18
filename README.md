# PlcTestSuite Mission

A user-friendly Unit Test Suite for PLC programmers, crafted to lower the barriers to Test-Driven Development.

## How-To

Main
```
PROGRAM MAIN
VAR
	ReRun		: BOOL;
END_VAR

TEST_Add();
IF ReRun THEN
	TestSuite.ReRunTests();
	ReRun := FALSE;
END_IF
```

Add one Method per test under Main
```
METHOD TEST_Add
VAR_INST
	Expected : DINT;
	Actual	 : DINT;
END_VAR

IF TestSuite.Test(__POUNAME()).ExecuteTest() THEN
	Expected := 0;
	Actual	 := -5 + 5;
	TestSuite.Test(__POUNAME()).AssertEqual(Expected, Actual);
END_IF
```
