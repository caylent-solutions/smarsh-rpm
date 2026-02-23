# Setup Guide

Step-by-step instructions for setting up RPM in new and existing projects.

## Prerequisites

- Git
- Bash shell
- Internet access (to clone asdf, Python, repo tool, and packages)

The `rpmConfigure` task installs all other tooling automatically (asdf, Python, repo tool).

## New Project Setup

### 1. Copy Template Files

Download the three files from this repository's [examples/example-gradle-task-runner/](../examples/example-gradle-task-runner/) directory:

```bash
curl -O https://raw.githubusercontent.com/caylent-solutions/smarsh-rpm/main/examples/example-gradle-task-runner/.rpmenv
curl -O https://raw.githubusercontent.com/caylent-solutions/smarsh-rpm/main/examples/example-gradle-task-runner/rpm-bootstrap.gradle
curl -O https://raw.githubusercontent.com/caylent-solutions/smarsh-rpm/main/examples/example-gradle-task-runner/build.gradle
```

### 2. Customize `.rpmenv`

Edit `.rpmenv` to match your project:

```properties
# Point to the manifest for your project type
REPO_MANIFESTS_PATH=repo-specs/java/gradle/springboot/microservice/meta.xml

# Set the Git base URL where packages are hosted
GITBASE=https://github.com/caylent-solutions/

# Java source/target version
JAVA_VERSION=17

# Optional private Maven repository URL (leave empty if not needed)
ARTIFACTORY_URL=https://your-company.jfrog.io/libs-release-local
```

All `.rpmenv` values can be overridden by environment variables of the same name (useful for CI/CD pipelines).

### 3. Customize `build.gradle`

Replace the template's project-specific section with your own:

```groovy
group = 'com.yourcompany'
version = '1.0.0'
jar { archiveFileName = "your-service.jar" }

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // Add your application dependencies
}
```

### 4. Run rpmConfigure

```bash
./gradlew rpmConfigure
```

### 5. Verify

```bash
./gradlew tasks       # Should show package-provided tasks
./gradlew build       # Should compile and run checks
```

## Existing Project Migration

See [RPM Guide - Section 6](rpm-guide.md#6-how-to-use-rpm-in-an-existing-project-migration) for detailed migration instructions.

## Troubleshooting

### `rpmConfigure` fails at "Installing asdf"

Check internet connectivity. asdf is cloned from GitHub.

### `rpmConfigure` fails at "Installing repo tool"

The repo tool is installed via `pip`. Ensure Python is available after asdf install completes. Check `.tool-versions` has a `python` entry.

### `repo envsubst` fails

Ensure `GITBASE` is set in `.rpmenv` and is a valid URL ending with `/`.

### `repo sync` fails with authentication errors

For public repos, no authentication is needed. For private repos, ensure `git` can authenticate (SSH keys or credential helper).

### Package scripts are not applied

Ensure `.packages/` exists and contains package directories. Run `./gradlew rpmConfigure` first.
