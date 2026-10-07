# Pipeline Audit

The baseline workflow was triggered with an empty commit (`f40c0cb`). GitHub did not
publish a run or register the workflow for this repository, so the baseline results
below combine the observed configuration with the behavior implied by the job steps.
The missing artifact makes `integration-tests` fail when it reaches its assertion;
the independent echo-only deployment jobs incorrectly succeed on a feature branch,
and `notify` is skipped when the integration job fails.

## `lint`

- **Purpose:** Install dependencies and run the ESLint checks.
- **Current issue:** It has no timeout or explicit place as the first stage in the dependency graph. The installed ESLint v10 also requires flat configuration, while the repository only had a legacy `.eslintrc.json`.
- **Correct fix:** Add a 10-minute timeout, keep lint dependency-free so it starts first, and provide an ESLint flat configuration.

## `unit-tests`

- **Purpose:** Run the unit test suite with `npm test`.
- **Current issue:** It starts immediately and can run before lint completes; it also has no timeout.
- **Correct fix:** Add `needs: lint` and a 15-minute timeout.

## `build`

- **Purpose:** Create the production files in `dist/`.
- **Current issue:** It starts immediately, has no timeout, and leaves `dist/` on its isolated runner, so later jobs cannot use it.
- **Correct fix:** Add `needs: lint`, a 20-minute timeout, and upload `dist/` as the `app-build` artifact.

## `integration-tests`

- **Purpose:** Verify the built application output with the integration test suite.
- **Current issue:** It runs independently and does not download the build output, so `dist/api.js` is missing and the test fails.
- **Correct fix:** Add `needs: build`, a 30-minute timeout, and download the `app-build` artifact into `dist/` before testing.

## `deploy-staging`

- **Purpose:** Deploy a validated revision to staging.
- **Current issue:** It has no dependencies, no branch guard, and no timeout, so it can report success on every branch without waiting for tests.
- **Correct fix:** Require `unit-tests` and `integration-tests`, restrict execution to `refs/heads/main`, and use a 15-minute timeout.

## `deploy-production`

- **Purpose:** Deploy the revision to production after staging.
- **Current issue:** It has no dependency, branch guard, or timeout, so it can run independently on any branch.
- **Correct fix:** Require `deploy-staging`, restrict execution to `refs/heads/main`, and use a 15-minute timeout.

## `notify`

- **Purpose:** Send the pipeline completion notification.
- **Current issue:** It has no dependency, no `always()` condition, and no timeout. It can run too early and is skipped when an upstream job fails.
- **Correct fix:** Depend on all pipeline jobs, use `if: always()`, and set a 15-minute timeout so notification is attempted for every outcome.
