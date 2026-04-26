---
layout: post
title: "Car Management API — MVP 1: Automated Testing with GitHub Actions"
description: "Setting up a CI pipeline so tests run automatically on every push. A YAML workflow file, a working-directory fix, and what green looks like."
date: 2026-04-26
category: DevOps
series: Car Management API
series_part: 4
---

*This is part 4 of the Car Management API series. [Part 3](/writing/2026-04-26-car-management-api-unit-tests) covers the unit tests themselves.*

---

Five tests passing locally is a good start. Tests that only run when you remember to run them are not a CI pipeline. The next step is making those tests run automatically every time code is pushed to the repository — and reporting pass or fail before anything gets merged.

GitHub Actions does this for free, and for a .NET project the setup is straightforward.

## What GitHub Actions is

GitHub Actions is a CI/CD platform built into GitHub. When something happens to your repository — a push, a pull request, a merge — a workflow can be triggered automatically. That workflow runs on a virtual machine hosted by GitHub, checks out your code, and executes whatever steps you define.

For this project the trigger is simple: any push to main runs the tests.

## The workflow file

GitHub Actions workflows live in a specific folder at the root of the repository:

```
.github/
└── workflows/
    └── tests.yml
```

The `.github` folder needs to be at the root of the repository — not inside a project subfolder. The name of the yaml file can be anything meaningful.

Here is the complete workflow:

```yaml
on:
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '9.0.x'

      - name: Restore
        run: dotnet restore
        working-directory: CarManagementSystem

      - name: Build
        run: dotnet build
        working-directory: CarManagementSystem

      - name: Test
        run: dotnet test
        working-directory: CarManagementSystem
```

## Breaking it down

**The trigger**

```yaml
on:
  push:
    branches: [ main ]
```

This runs the workflow on every push to the main branch. Pull request triggers can be added later when the project has more contributors or a more formal review process.

**The job**

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
```

GitHub spins up a fresh Ubuntu virtual machine for every run. Nothing persists between runs — each one starts completely clean. This is important: if the tests pass on GitHub they pass on a clean machine, not just your local setup.

**The steps**

Each step either uses a pre-built GitHub Action (`uses`) or runs a shell command directly (`run`).

`actions/checkout@v3` checks out the repository code onto the virtual machine.

`actions/setup-dotnet@v3` installs .NET. The version matches whatever your project targets — check your `.csproj` file for the `<TargetFramework>` value.

`dotnet restore`, `dotnet build`, and `dotnet test` are the same three commands you would run locally. They restore NuGet packages, compile the code, and run the tests.

**The working-directory fix**

This tripped me up. The repository structure has a subfolder:

```
CarManagementSystem/          ← repository root
└── CarManagementSystem/      ← solution folder
    ├── CMS_ClassLibrary/
    ├── CMS_Testing/
    └── CarManagementSystem.sln
```

Without `working-directory`, the dotnet commands run from the repository root and cannot find the solution file. Adding `working-directory: CarManagementSystem` to each step points them at the right folder.

The error without it:

```
MSBUILD : error MSB1003: Specify a project or solution file.
The current working directory does not contain a project or solution file.
```

The fix is those three `working-directory` lines.

## Creating the file on GitHub

The workflow file was created directly in the GitHub web interface:

1. Navigate to the repository
2. Create the folder path `.github/workflows/`
3. Create `tests.yml` and paste in the workflow
4. Commit directly to main

Then `git pull` locally to bring the file down.

## What a passing run looks like

After pushing any change to main, the Actions tab on the repository shows the workflow running. Each step ticks green as it completes:

```
✓ Checkout code
✓ Setup .NET
✓ Restore
✓ Build
✓ Test — 5 tests passed
```

The test output from the runner:

```
Test run for CMS_Testing.dll (.NETCoreApp,Version=v8.0)
A total of 1 test files matched the specified pattern.

Passed!  - Failed: 0, Passed: 5, Skipped: 0, Total: 5
```

Any failure shows a red cross on the commit in GitHub. The exact assertion that failed and the actual versus expected values are in the workflow log.

## Why this matters

From this point forward every push to main is automatically verified. If a change breaks one of the five tests the failure is visible immediately — before it affects anything else.

As the test suite grows through each MVP, the pipeline grows with it. Integration tests, Postman collections run with Newman, Playwright end-to-end tests — they all get added as steps in the same workflow. By MVP 7 the pipeline will block a deployment if any layer of the test suite fails.

The habit starts here with five unit tests and one YAML file.

---

*MVP 1 is complete. Next: MVP 2a — connecting to a real PostgreSQL database with Dapper and writing the first integration tests.*
