# GitHub Workflows

My workflows centralized for reuse across my repositories.

## Published workflows

### npm-ci-build

Job that executes `ci-build` logic for npm packages.
`v1` runs `ci-build` directly and assumes that the underlying package orchestrates the full build correctly.

Use a `.node-version` file in the root of the repository to set the Node.js version.

The job includes GitVersion for automatic versioning.
It will replace `0.0.0-gitversion` with the correct version.

The job will output GitVersion variables under `npm-ci-build.GitVersion` (`job.step`).

Note: The job will ignore any local `GitVersion.yaml`; it configures GitVersion for continuous deployment internally.
Use `+semver:(major|minor)` in commit messages appropriately.
(Patch happens automatically.)

## Goal: Universal targets

I find that a universal set of targets (also called tasks) simplifies polyglot development.
I observed the benefits with Amazon’s internal build system, and I built similar mechanisms at Total Quality Logistics (TQL).

This repository represents my personal take on the idea, implemented via a consistent set of GitHub workflows.

### Industry observations

This section details my observations from industry tools that I have used.

- Amazon: Standardized install, build, test (unit-test and integration-test) all under release.
  (Source: per my memory 18+ months later)

- Gradle: Standardizes check, assemble, and build tasks.
  Includes some sort of standardized dependency installation (but I cannot find the task name).
  Specific plugins (such as java) extend with their own standard set (compile, javadoc, and more).
  (Source: [base plugin](https://docs.gradle.org/current/userguide/base_plugin.html))

- Maven: Standardizes validate, compile, test, package, verify, install, and deploy
  (Source: [Maven build lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html))

- .NET (`dotnet`): Restore, build, test, format, run, and publish.
  (Source: My limited usage and [dotnet commands](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet))

- npm: Standardizes install/ci/rebuild, start/stop/restart, test, and many related to prepare and publish.
  Many libraries suggest build and lint, but there are others related (compile, tsc, specific linters, and formatters).
  (Source: My experience and [npm lifecycle scripts](https://docs.npmjs.com/cli/v10/using-npm/scripts))

- Python: Unsure.
  I know of install from pipenv.
  I see build, fmt, and test from hatch, install and build from flit, and easy_install and build from Setuptools.

Each of these also had the concept of clean.

### My universal targets

Combining my experiences predominantly with Java, npm, and .NET, I arrive at the following universal targets.
Each underlying build system should be able to hook into these standardized targets.

- `install`: Installs (or restores) dependencies.

- `build`: Builds the project.

- `format`: Formats and lints the project.

- `test`: Runs all tests that do not require a network connection.
  Often thought of as unit tests, but can contain network-free integration tests (aka component or module tests).
  This should include code coverage.

- `publish`: Publish the library to its package manager.

- `version`: Handles automatic versioning, if desired.

- `clean`: Cleans any build or test artifacts.
  SHOULD NOT remove dependencies.

In CI (think code reviews or pull requests), the “build” executes many of these targets.
Using the term “build” here would overload the target name though.
Therefore, I use the term `ci-build` to group all targets that should run during the “CI build.”

We called this `release` at Amazon, but I have found that folks think `release` means “make the release; actually publish,” which represents the wrong conclusion.
Hence, I chose the term `ci-build`.
_(I also considered terms such as `full-build`, `release-build`, and `ci` or replacing the underlying `build` term with `compile`.
However, others came to the right conclusion with `ci-build`, which reinforces its choice.)_

If I was building my own CLI that wrapped underlying tools with these universal targets, I would include `compile` and `lint` as aliases for `build` and `format`.
If both targets were present, the main name would take precedence.

### My universal output locations

Standardizing the output locations also simplifies polyglot development.
While tooling (GitHub workflows) receives the most benefits, developers can benefit from standardized code coverage output locations.
Standardized outputs can also simplify `.gitignore` configuration.
_(Nit-pick side note: The 100s-lines-long default `.gitignore` files pain me.)_

The standard output locations follow.

- `artifacts` should contain as much output as possible.
  This also simplifies clean.

- Dependencies will use underlying tool conventions (example: `node_modules` for npm).

- Publish will use underlying tool conventions where appropriate.
  Use `artifacts/dist` to vend transpiled sources.

- Mono-repos will use `$projectName` in the path per artifact type (example: `artifacts/dist/$projectName`).
  This preserves output structure semantics between polyrepo (single project repositories) and monorepo use cases.

- Tests will use `artifacts/test-results`.
  Prefer JUnit format for the test results file and use the name `test-result.xml`.
  Prefer Cobertura format for the code coverage results file and use the name `cobertura.xml`.

  In monorepos, each project likely has these files in `artifacts/test-results/$projectName` _AND_ the monorepo workspace aggregate in `artifacts/test-results` (N+1 output files).
  Note that tools like [reportgenerator](https://reportgenerator.io) exist to simplify aggregation.

- The HTML output from code coverage should have the name `artifacts/coverage/index.html`.
  This differs from `artifacts/test-results` to simplify developers opening the report and to prevent conflicts in monorepos.
