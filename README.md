# split-tests

A Github Action for automatically splitting a test suite between multiple runners.

## Introduction

This action splits a suite of tests files into equally sized buckets, to distribute the execution time between multiple runners.

To ensure an equivalent distribution of run time between runners, this action can use the timings of a previous run to inform the tests splits (otherwise the line-count of each file is used to estimate the run time). Sub-actions are provided to save and restore the test timings between runs.

## Inputs

- `job-index` (required): index of the current split job (usually `${{ strategy.job-index }}`).
- `job-total` (required): total number of split jobs (usually `${{ strategy.job-total }}`).
- `pattern` (optional): a glob pattern of test files to include (by default: `spec/**/*_spec.rb`).
- `exclude-pattern` (optional): a glob pattern of test files to exclude.
- `junit-pattern` (optional): a glob pattern to JUnit XML reports to use for test timings (by default: `tmp/*.junit.xml`).

## Outputs

- `tests-subset`: a list of test files to be run for the current job.

## Usage

### Basic workflow

Split a test suite between 4 runners, based on file lines count.

```yaml
jobs:
  tests:
    strategy:
      fail-fast: true
      matrix:
        # The number of runners is arbitrary.
        # The test suite will automatically be split between all requested runners.
        runners: [ 0, 1, 2, 3 ]

    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 'Determine the subset of test files to run'
        uses: enercoop/split-tests@v1
        id: split-tests
        with:
          pattern: 'spec/**'
          job-index : ${{ strategy.job-index }}
          job-total : ${{ strategy.job-total }}

      - name: 'Run the subset of test files'
        run: bin/rspec ${{ steps.split-tests.outputs.tests-subset }}
```

### Advanced workflow, with tests timings save and restoration

Split a test suite between 6 runners, based on test timings of a previous run.

```yaml
jobs:
  tests:
    strategy:
      fail-fast: true
      matrix:
        # The number of runners is arbitrary.
        # The test suite will automatically be split between all requested runners.
        runners: [ 0, 1, 2, 3, 4, 5 ]

    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 'Restore tests reports of a previous run'
        uses: enercoop/split-tests/restore-workflow-test-timings@v1
        with:
          pattern: 'tmp/*.junit.xml'

      - name: 'Determine the subset of test files to be run'
        uses: enercoop/split-tests@v1
        id: split-tests
        with:
          pattern: 'spec/**'
          job-index : ${{ strategy.job-index }}
          job-total : ${{ strategy.job-total }}

      - name: 'Run the subset of test files'
        run: bin/rspec --format RspecJunitFormatter --out tmp/results_${{ github.job }}_${{ strategy.job-index }}.junit.xml ${{ steps.split-tests.outputs.tests-subset }}

      - name: 'Save test timings for job ${{strategy.job-index }}'
        uses: enercoop/split-tests/save-job-test-timings@v1
        with:
          workflow: ${{ github.job }}
          job-index: ${{ strategy.job-index }}
          pattern: 'tmp/*.junit.xml'

  tests_post:
    needs: [ tests ]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 'Once all tests have run, collect and save the test reports'
        uses: enercoop/split-tests/save-workflow-test-timings@v1
        with:
          workflow: 'tests'
          pattern: 'tmp/*.junit.xml'
```

## Under the hood

This action uses [leonid-shevtsov/split_tests](https://github.com/leonid-shevtsov/split_tests) to perform the actual splitting of the tests.
