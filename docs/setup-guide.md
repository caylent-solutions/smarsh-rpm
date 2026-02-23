# Setup Guide

Step-by-step instructions for setting up RPM in new and existing projects.

## Prerequisites

- Git
- Bash shell
- Python 3 with pipx (`command -v python3 && command -v pipx`)
- Internet access (to install the repo tool and clone packages)

The `rpmConfigure` task installs the repo tool via pipx and syncs all packages.

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
GITBASE=https://github.com/your-org/

# Java source/target version
JAVA_VERSION=17

# Optional private Maven repository URL (leave empty if not needed)
ARTIFACTORY_URL=https://your-company.jfrog.io/libs-release-local
```

All `.rpmenv` values can be overridden by environment variables of the same name (useful for CI/CD pipelines).

### 3. Customize `build.gradle`

Replace the template's project-specific section with your own. RPM provides plugins but does NOT manage dependency versions — each project owns its version pins and BOM imports:

```groovy
group = 'com.yourcompany'
version = '1.0.0'
jar { archiveFileName = "your-service.jar" }

// Dependency versions and BOM imports (owned by each project, not RPM)
ext {
    set('springCloudVersion', '2023.0.3')
    set('jacksonVersion', '2.17.2')
    testcontainersVersion = '1.19.8'
}

dependencyManagement {
    imports {
        mavenBom "com.fasterxml.jackson:jackson-bom:${jacksonVersion}"
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}

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

### `rpmConfigure` fails with "python3 is not installed"

Python 3 must be available on PATH before running `rpmConfigure`.
- **DevContainer:** Python is provided by the devcontainer Python feature.
- **CI/CD:** Add a Python installation step before running Gradle.
- **Local:** Install Python 3 via your system package manager.

### `rpmConfigure` fails with "pipx is not installed"

pipx must be available on PATH. Install it with `python3 -m pip install --user pipx && pipx ensurepath`.

### `rpmConfigure` fails at "Installing repo tool"

The repo tool is installed via `pipx`. Ensure pipx is configured correctly and internet access is available.

### `repo envsubst` fails

Ensure `GITBASE` is set in `.rpmenv` and is a valid URL ending with `/`.

### `repo sync` fails with authentication errors

For public repos, no authentication is needed. For private repos, ensure `git` can authenticate (SSH keys or credential helper).

### Package scripts are not applied

Ensure `.packages/` exists and contains package directories. Run `./gradlew rpmConfigure` first.
