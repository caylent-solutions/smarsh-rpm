# Pipeline Integration

How to use RPM tasks in CI/CD pipelines.

## Overview

Each RPM package provides discrete Gradle tasks that map to CI/CD pipeline stages. These tasks can run in parallel for faster pipelines.

## GitHub Actions Example

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  rpm-configure:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      - name: RPM Configure
        run: ./gradlew rpmConfigure
      - uses: actions/cache/save@v4
        with:
          path: |
            .packages
            .repo
          key: rpm-packages-${{ hashFiles('.rpmenv') }}

  build:
    needs: rpm-configure
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/cache/restore@v4
        with:
          path: |
            .packages
            .repo
          key: rpm-packages-${{ hashFiles('.rpmenv') }}
      - run: ./gradlew build

  checkstyle:
    needs: rpm-configure
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/cache/restore@v4
        with:
          path: |
            .packages
            .repo
          key: rpm-packages-${{ hashFiles('.rpmenv') }}
      - run: ./gradlew checkstyleMain

  unit-test:
    needs: rpm-configure
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/cache/restore@v4
        with:
          path: |
            .packages
            .repo
          key: rpm-packages-${{ hashFiles('.rpmenv') }}
      - run: ./gradlew unitTest jacocoTestReport

  integration-test:
    needs: rpm-configure
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/cache/restore@v4
        with:
          path: |
            .packages
            .repo
          key: rpm-packages-${{ hashFiles('.rpmenv') }}
      - run: ./gradlew integrationTest

  security:
    needs: rpm-configure
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/cache/restore@v4
        with:
          path: |
            .packages
            .repo
          key: rpm-packages-${{ hashFiles('.rpmenv') }}
      - run: ./gradlew securityCheck

  sonarqube:
    needs: [unit-test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/cache/restore@v4
        with:
          path: |
            .packages
            .repo
          key: rpm-packages-${{ hashFiles('.rpmenv') }}
      - run: ./gradlew sonar
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

## Overriding GITBASE in Pipelines

CI/CD pipelines can override `GITBASE` to use internal Git mirrors:

```yaml
- name: RPM Configure
  run: ./gradlew rpmConfigure
  env:
    GITBASE: https://git.internal.company.com/rpm-packages/
```

The `.rpmenv` value is overridden by the environment variable. No file changes needed.
