---
name: regression-result-parser
description: "Parse regression.log files into structured test results with pass/fail counts, failure-signature buckets, Unknown classification, and warnings for malformed records or unrecognized signatures. Use for daily regression result parsing and regression report preparation."
argument-hint: "Provide a regression directory, a regression.log path, or a parent directory containing regression logs."
user-invocable: true
disable-model-invocation: false
---

# Regression Result Parser

Parse one or more `regression.log` files into reliable structured data for regression analysis and reporting.

## When to Use

Use this skill when the task involves:

- Finding `regression.log` files under a supplied path or the workspace.
- Reading itemized test name, status, and signature fields.
- Counting passing and failing tests.
- Grouping failures by signature.
- Detecting missing, malformed, or unrecognized result data.
- Preparing structured input for an HTML regression report.

## Supported Record Format

Prefer records with these labeled fields, separated by blank lines or by the next `TEST` field:

```text
TEST: test_name
STATUS: PASS|FAIL
SIGNATURE: signature_name
```

Also accept equivalent labels such as `Test Name`, `Test Status`, and `Failure Signature`. Match field names and `PASS`/`FAIL` values case-insensitively, and trim surrounding whitespace. Do not infer a test result from arbitrary prose.

## Parsing Procedure

1. Resolve the input path.
   - If it is a file, parse that file.
   - If it is a directory, search it recursively for `regression.log`.
   - If no path is supplied, search the workspace recursively.
   - Group each log by its containing regression directory.
2. Read each log without modifying it.
3. Build one record per complete test item with:
   - `regression`: containing directory name
   - `source`: normalized log path
   - `testName`
   - `status`: `PASS` or `FAIL`
   - `signature`: trimmed signature or `Unknown` when blank or absent
4. Accept only explicit `PASS` and `FAIL` statuses. Add incomplete or invalid records to `warnings` and do not count them.
5. Use `TIMEOUT`, `ASSERTION`, and `CRASH` as the default recognized signatures unless the user provides a configured list.
6. For a failing test with a recognized nonblank signature, bucket it by that signature.
7. For a failing test with a blank or absent signature, bucket it as `Unknown`.
8. For a failing test with a nonblank signature outside the recognized list:
   - Preserve the exact trimmed signature in the test record and its own bucket.
   - Add a warning identifying the regression, test, and signature.
   - Do not additionally count it in `Unknown`.
9. Do not use signatures from passing tests in failure buckets, though retain them in test-level records if present.
10. Reconcile each log and the combined result:
    - `total = passing + failing`
    - The sum of failure-signature buckets equals `failing`.
    - No test is counted more than once.

## Output Contract

Return structured results with this shape:

```json
{
  "regressions": [
    {
      "name": "regression_01",
      "source": "regression_01/regression.log",
      "tests": [
        { "testName": "example", "status": "PASS", "signature": "Unknown" }
      ],
      "passing": 1,
      "failing": 0,
      "failureBuckets": {},
      "warnings": []
    }
  ],
  "overall": {
    "passing": 1,
    "failing": 0,
    "failureBuckets": {}
  },
  "warnings": [],
  "reconciliation": {
    "logsFound": 1,
    "regressionsFound": 1,
    "expectedDailyRegressions": 4,
    "countsReconciled": true
  }
}
```

If no valid records are found, return empty counts, `countsReconciled: true` for the empty set, and warnings explaining whether the cause was a missing log, an unreadable path, or malformed records.

## Handoff to Reporting

The caller owns HTML generation. Pass along all structured records and warnings; never hide malformed records or unrecognized signatures. The report should identify the source path and generated timestamp.
