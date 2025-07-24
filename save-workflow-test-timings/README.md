# save-workflow-test-timings action

The `save-workflow-test-timings` action saves all the test timing reports of multiple jobs into a single cache entry.

The entry can then be restored during a future run using the `restore-workflow-test-timings` action.

## How does it work

At the end of a split test run, each split job is expected to save its test timing report using the `save-job-test-timings` action.

Then, when all jobs are done, the `save-workflow-test-timings` action collects the individual timing reports of each job. It then saves them into a cache entry, for later retrival by a future CI run.

## Inputs

The action takes two parameters:

- `workflow` (required): the name of the workflow that saved the tests reports.
- `pattern` (optional): the glob pattern of JUnit timing reports.

## Usage

The `save-workflow-test-timings` action must be run once all split jobs are complete.

The best way to do this is to add a post job that depends on your main job:

```yaml
jobs:
  tests:
    strategy:
      matrix:
        runners: [ 0, 1, 2, 3 ]
    steps:
      # Do the testing

  tests_post:
    needs: [ tests ]
    uses: enercoop/split-tests/save-workflow-test-timings@v1
```
