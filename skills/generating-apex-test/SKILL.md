---
name: generating-apex-test
description: >
  Generate and validate Apex test classes with TestDataFactory patterns, bulk testing (251+ records),
  mocking strategies, assertion best practices, and structured test-fix loops. Use when creating new
  Apex test classes, improving test coverage, fixing failing tests, running test execution and coverage
  analysis, or implementing testing patterns for triggers, services, controllers, batch jobs, queueables,
  and integrations. Triggers on: *Test.cls, *_Test.cls files, sf apex run test workflows, coverage reports,
  test-fix loops. Do NOT trigger for production Apex code (use generating-apex) or Jest/LWC tests.
---

# Generating Apex Tests

Generate production-ready Apex test classes and run disciplined test-fix loops with coverage analysis.

## Core Principles

1. **One behavior per method** — each test method validates a single scenario; separate positive, negative, and bulk tests
2. **Bulkify tests** — test with 251+ records to cross the 200-record trigger batch boundary
3. **Isolate test data** — use `@TestSetup` and TestDataFactory; never rely on org data (`SeeAllData=false`) or hardcoded IDs
4. **Assert meaningfully** — test behavior, not just coverage; always include failure messages
5. **Use `Assert` class only** — use `Assert.areEqual`, `Assert.isTrue`, `Assert.fail`, etc. Never use legacy `System.assert`, `System.assertEquals`, or `System.assertNotEquals`
6. **Mock external dependencies** — use `HttpCalloutMock`, `Test.setMock()`, DML mocking for integrations
7. **Test negative paths** — validate error handling and exception scenarios, not just happy paths
8. **Wrap with start/stop** — pair `Test.startTest()` with `Test.stopTest()` to reset governor limits and force async execution

## Workflow

### Step 1 — Gather Context

Before generating or fixing tests, identify:

- the target production class(es) under test
- existing test classes, test data factories, and setup helpers
- desired test scope (single class, specific methods, suite, or local tests)
- coverage threshold expectation (75% minimum for deploy, 90%+ recommended)
- org alias when running tests against an org

### Step 2 — Generate the Test Class

Apply the structure, naming conventions, and patterns below. Reference the appropriate asset templates and reference docs for the component type.

#### Test Class Structure

```apex
@IsTest
private class MyServiceTest {

    @TestSetup
    static void setupTestData() {
        List<Account> accounts = TestDataFactory.createAccounts(251);
        insert accounts;
    }

    @IsTest
    static void shouldProcessAllAccounts_WhenValidInput() {
        // Given
        List<Account> accounts = [SELECT Id, Name FROM Account];

        // When
        Test.startTest();
        MyService.processAccounts(accounts);
        Test.stopTest();

        // Then
        List<Account> updated = [SELECT Id, Status__c FROM Account];
        Assert.areEqual(251, updated.size(), 'All accounts should be processed');
        for (Account acc : updated) {
            Assert.areEqual('Processed', acc.Status__c, 'Status should be updated');
        }
    }

    @IsTest
    static void shouldThrowException_WhenInputIsEmpty() {
        // Given
        List<Account> emptyList = new List<Account>();

        // When / Then
        Test.startTest();
        try {
            MyService.processAccounts(emptyList);
            Assert.fail('Expected MyCustomException to be thrown');
        } catch (MyCustomException e) {
            Assert.isTrue(e.getMessage().contains('cannot be empty'),
                'Exception message should indicate empty input');
        }
        Test.stopTest();
    }
}
```

#### Naming Convention

Use descriptive method names: `test[SubjectOrAction]_[Scenario]_[ExpectedResult]`

- `testAccountUpdate_ChangeName_Success`
- `testEmailValidation_InvalidFormat_ThrowsException`
- `testOpportunity_ClosedWon_SendsNotification`
- `testBatchExecution_RunningAsBatch_TriggerBypassed`

### Step 3 — Run the Smallest Useful Test Set

Start narrow when debugging a failure; widen only after the fix is stable. See [references/cli-commands.md](references/cli-commands.md) for `sf apex run test` usage.

### Step 4 — Analyze Results

Focus on:

- failing methods — exception types and stack traces
- uncovered lines and weak coverage areas
- whether failures indicate bad test data, brittle assertions, or broken production logic

### Step 5 — Fix Loop

When tests fail, run a disciplined fix loop:

1. Read the failing test class and the class under test
2. Identify root cause from error messages and stack traces
3. Apply the fix (test data, assertion, or production code)
4. Rerun the focused test before broader regression
5. Repeat until all tests pass or root cause requires design change

See [references/test-fix-loop.md](references/test-fix-loop.md) for the full loop protocol.

### Step 6 — Validate Coverage

Ensure coverage meets thresholds:

| Level | Coverage | Purpose |
|-------|----------|---------|
| Production deploy | 75% minimum | Required by Salesforce |
| Recommended | 90%+ | Best practice target |
| Critical paths | 100% | Business-critical code |

Cover all paths: positive, negative/exception, bulk (251+ records), callout/async.

## What to Test by Component

| Component | Key Test Scenarios |
|-----------|-------------------|
| Trigger | Bulk insert/update/delete, recursion guard, field change detection |
| Service | Valid/invalid inputs, bulk operations, exception handling |
| Controller | Page load, action methods, view state |
| Batch | start/execute/finish, chunking, error records |
| Queueable | Chaining, bulkification, error handling |
| Callout | Success response, error response, timeout |
| Scheduled | Execution, CRON validation |

## Output Format

When reporting test results, use this structure:

```text
Test run: <scope>
Org: <alias>
Result: <passed / partial / failed>
Coverage: <percent / key classes>
Issues: <highest-signal failures>
Next step: <fix class, add test, rerun scope, or widen regression>
```

## Reference Files

Load these on demand for detailed patterns:

| Reference | When to use |
|-----------|-------------|
| [references/test-data-factory.md](references/test-data-factory.md) | TestDataFactory class patterns and field defaults |
| [references/assertion-patterns.md](references/assertion-patterns.md) | Assertion best practices, anti-patterns, common pitfalls |
| [references/mocking-patterns.md](references/mocking-patterns.md) | HttpCalloutMock, DML mocking, StubProvider, Selector mocking, Email and Platform Event testing |
| [references/async-testing.md](references/async-testing.md) | Batch, Queueable, Future, Scheduled job testing |
| [references/test-patterns.md](references/test-patterns.md) | Test patterns by component type with asset template pointers |
| [references/test-fix-loop.md](references/test-fix-loop.md) | Structured test-fix loop protocol |
| [references/cli-commands.md](references/cli-commands.md) | sf CLI test execution commands |
| [references/performance-optimization.md](references/performance-optimization.md) | Test execution speed and optimization |
