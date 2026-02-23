# RPM Guide

A comprehensive guide to RPM (Repo Package Manager) for engineers who are new to this pattern.

## Table of Contents

- [1. The Problem It Solves](#1-the-problem-it-solves)
- [2. What RPM Is](#2-what-rpm-is)
- [3. How It Works](#3-how-it-works)
- [4. What Each Package Provides](#4-what-each-package-provides)
- [5. How to Use RPM in a New Project](#5-how-to-use-rpm-in-a-new-project)
- [6. How to Use RPM in an Existing Project (Migration)](#6-how-to-use-rpm-in-an-existing-project-migration)
- [7. How to Push Updates](#7-how-to-push-updates)
- [8. How to Create a New RPM Package](#8-how-to-create-a-new-rpm-package)
- [9. How envsubst Makes RPM Portable](#9-how-envsubst-makes-rpm-portable)
- [10. Browsing the Source](#10-browsing-the-source)
- [11. Architecture Decisions](#11-architecture-decisions)

---

## 1. The Problem It Solves

Java/Spring Boot projects at Smarsh share common build needs: Java compilation, checkstyle linting, security scanning, test framework configuration, code coverage, SonarQube analysis, and local development tooling. Without RPM, every project embeds all of this configuration directly in its `build.gradle`.

This creates several problems:

**Standards drift.** When each project owns its own checkstyle rules, test configuration, and security scanning setup, they diverge over time. Project A updates its checkstyle rules. Project B does not. Six months later, the rules are incompatible and there is no single source of truth.

**Update friction.** Updating a shared standard (a new checkstyle rule, a JaCoCo version bump, a security scanner configuration change) requires opening pull requests to every single project. With 20+ Java projects, this is impractical.

**Onboarding cost.** New projects start by copy-pasting hundreds of lines of build configuration from an existing project. This is error-prone and perpetuates whatever drift exists in the source project.

**Pipeline duplication.** CI/CD pipelines duplicate the same validation steps across projects with subtle differences. There is no way to share discrete pipeline stages.

**Agent limitations.** Development agents cannot run standardized validation locally because the validation configuration is scattered across project files rather than available as discrete, well-defined tasks.

---

## 2. What RPM Is

RPM is a **package manager for build tooling and project configuration** — not for application code. It provides:

- A **central manifest repository** ([smarsh-rpm](https://github.com/caylent-solutions/smarsh-rpm)) that defines which packages each project type needs
- Multiple **package repositories** ([smarsh-rpm-gradle-*](https://github.com/caylent-solutions?q=smarsh-rpm-gradle)) that each provide a focused set of Gradle tasks and configuration
- A **bootstrap mechanism** (`rpm-bootstrap.gradle`) committed to each project that syncs packages and auto-applies them

Under the hood, RPM uses a [fork of Google's `repo` tool](https://github.com/caylent-solutions/git-repo) (with `envsubst` support) to sync packages from Git repositories.

---

## 3. How It Works

### The 3-File Contract

Every RPM-managed project commits exactly three files:

```
my-project/
├── .rpmenv                 # Configuration (manifest URL, versions, Git base URL)
├── rpm-bootstrap.gradle    # Bootstrap script (rarely changes)
└── build.gradle            # Project-specific config only (group, version, dependencies)
```

Everything else — checkstyle rules, JaCoCo config, SonarQube setup, security scanning, test framework config, local dev tasks — comes from RPM packages synced to `.packages/`.

### The Sync Flow

When a developer runs `./gradlew rpmConfigure`, the bootstrap script executes these steps:

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Verify Python 3  │────>│ Install repo tool│
│ and pipx on PATH │     │   (via pipx)     │
└──────────────────┘     └──────────────────┘
                                  │
                                  v
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   repo sync     │<────│  repo envsubst   │<────│   repo init      │
│ (clone packages │     │ (resolve ${GIT-  │     │ (clone manifest  │
│  to .packages/) │     │  BASE} to URL)   │     │  with templates) │
└─────────────────┘     └──────────────────┘     └──────────────────┘
         │
         v
┌─────────────────────────────────────────────┐
│  .packages/                                  │
│  ├── smarsh-rpm-gradle-build/                │
│  ├── smarsh-rpm-gradle-checkstyle/           │
│  ├── smarsh-rpm-gradle-security/             │
│  ├── smarsh-rpm-gradle-unit-test/            │
│  ├── smarsh-rpm-gradle-integration-test/     │
│  ├── smarsh-rpm-gradle-sonarqube/            │
│  └── smarsh-rpm-gradle-local-dev/            │
└─────────────────────────────────────────────┘
```

### The Auto-Apply Mechanism

When Gradle evaluates `build.gradle`, the `apply from: 'rpm-bootstrap.gradle'` line triggers package auto-discovery:

```groovy
// rpm-bootstrap.gradle does this:
def rpmPackagesDir = file('.packages')
if (rpmPackagesDir.exists()) {
    rpmPackagesDir.eachDir { pkgDir ->
        project.ext.set('_rpmCurrentPkgDir', pkgDir.absolutePath)
        fileTree(pkgDir) { include '*.gradle' }.sort().each { scriptFile ->
            apply from: scriptFile
        }
    }
}
```

This iterates over every directory in `.packages/`, sets a `_rpmCurrentPkgDir` property so each script knows where its own assets are, and applies all `.gradle` files found. The result: every Gradle task and configuration from every package is available as if it were defined directly in `build.gradle`.

### The Manifest Hierarchy

```
smarsh-rpm (this repo)
├── repo-specs/git-connection/remote.xml     ← Defines Git remote with ${GITBASE}
└── repo-specs/java/gradle/springboot/
    └── microservice/
        ├── meta.xml                          ← Includes remote.xml + packages.xml
        └── packages.xml                      ← Lists 7 package repos with version tags
```

`meta.xml` ties together the Git remote definition and the package list. `packages.xml` pins each package to a specific Git tag (e.g., `refs/tags/1.0.0`). When a package is updated, the tag is bumped in `packages.xml`.

---

## 4. What Each Package Provides

| Package | Gradle Tasks | Config Files | Replaces |
|---|---|---|---|
| [smarsh-rpm-gradle-build](https://github.com/caylent-solutions/smarsh-rpm-gradle-build) | `build`, `bootJar`, `bootBuildInfo` | `build-conventions.gradle`, `rpm-manifest.properties` | Java version, Spring Boot plugin, repos, dependency management, dist task disabling |
| [smarsh-rpm-gradle-checkstyle](https://github.com/caylent-solutions/smarsh-rpm-gradle-checkstyle) | `checkstyleMain` | `checkstyle.gradle`, `config/checkstyle/checkstyle.xml`, `config/checkstyle/suppressions.xml` | Checkstyle plugin config, checkstyle rules files |
| [smarsh-rpm-gradle-security](https://github.com/caylent-solutions/smarsh-rpm-gradle-security) | `securityCheck`, `secretScan`, `dependencyCheckAnalyze` | `security.gradle`, `rpm-manifest.properties` | OWASP plugin config, TruffleHog local scanning via Docker |
| [smarsh-rpm-gradle-unit-test](https://github.com/caylent-solutions/smarsh-rpm-gradle-unit-test) | `test`, `unitTest`, `jacocoTestReport` | `unit-test.gradle` | TestNG config, JaCoCo config, test logging, coverage reporting |
| [smarsh-rpm-gradle-integration-test](https://github.com/caylent-solutions/smarsh-rpm-gradle-integration-test) | `integrationTest` | `integration-test.gradle` | Integration test source set, configurations, test task |
| [smarsh-rpm-gradle-sonarqube](https://github.com/caylent-solutions/smarsh-rpm-gradle-sonarqube) | `sonar` | `sonarqube.gradle`, `rpm-manifest.properties` | SonarQube plugin config |
| [smarsh-rpm-gradle-local-dev](https://github.com/caylent-solutions/smarsh-rpm-gradle-local-dev) | `localDevUp`, `localDevDown`, `localDevRestart`, `localDevDestroy`, `bootRunLocal` | `local-dev.gradle` | Docker Compose management, health checks, service provisioning |

### CI/CD Pipeline Mapping

Each package maps 1:1 to a discrete CI/CD pipeline stage:

```
┌─────────┐  ┌────────────┐  ┌───────────┐  ┌──────────────────┐  ┌──────────┐  ┌───────────┐
│  build   │  │ checkstyle │  │ unit-test │  │ integration-test │  │ security │  │ sonarqube │
│ (compile)│  │   (lint)   │  │ (test+cov)│  │    (int test)    │  │  (scan)  │  │ (quality) │
└─────────┘  └────────────┘  └───────────┘  └──────────────────┘  └──────────┘  └───────────┘
     │              │              │                  │                  │              │
     └──────────────┴──────────────┴──────────────────┴──────────────────┴──────────────┘
                                    Can run in parallel
```

---

## 5. How to Use RPM in a New Project

### Step 1: Download the three files

Copy from [examples/example-gradle-task-runner/](../examples/example-gradle-task-runner/):
- `.rpmenv` — Configuration
- `rpm-bootstrap.gradle` — Bootstrap script
- `build.gradle` — Template (customize for your project)

### Step 2: Customize `.rpmenv`

Set `REPO_MANIFESTS_PATH` to the manifest for your project type:
```properties
REPO_MANIFESTS_PATH=repo-specs/java/gradle/springboot/microservice/meta.xml
```

Set `GITBASE` to the GitHub organization hosting the packages:
```properties
GITBASE=https://github.com/your-org/
```

Set build configuration:
```properties
# Java source/target version
JAVA_VERSION=17

# Optional private Maven repository URL (leave empty if not needed)
ARTIFACTORY_URL=https://your-company.jfrog.io/libs-release-local
```

### Step 3: Customize `build.gradle`

Keep only project-specific configuration:
```groovy
buildscript {
    // ... (RPM buildscript block — copy as-is from template)
}
apply from: 'rpm-bootstrap.gradle'

group = 'com.yourcompany'
version = '1.0.0'
jar { archiveFileName = "your-service.jar" }

dependencies {
    // Your application-specific dependencies only
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

### Step 4: Run rpmConfigure

```bash
./gradlew rpmConfigure
```

This installs tooling and syncs all packages to `.packages/`.

### Step 5: Verify

```bash
./gradlew tasks --group rpm        # See RPM management tasks
./gradlew tasks                     # See all tasks including package-provided ones
./gradlew build                     # Compile + checkstyle + unit tests + coverage
./gradlew unitTest                  # Unit tests only
./gradlew integrationTest           # Integration tests only
./gradlew checkstyleMain            # Lint only
./gradlew securityCheck             # Security scan only
```

### Overriding Package Defaults

If a package sets a value you need to override for your specific project, use Gradle's `ext` block in your `build.gradle`:

```groovy
// Override a version set by the build package
ext {
    set('jacksonVersion', '2.18.0')  // Override the default
}
```

Or for repository URLs, set them in `gradle.properties`:
```properties
rpm.artifactory.url=https://your-company.jfrog.io/libs-release-local
```

---

## 6. How to Use RPM in an Existing Project (Migration)

### What to Remove

1. **Plugin declarations** — All `plugins { id '...' }` blocks for plugins provided by packages (java, checkstyle, jacoco, sonarqube, Spring Boot, dependency-management)
2. **Checkstyle config** — `config/checkstyle/` directory and checkstyle block in `build.gradle`
3. **JaCoCo config** — `jacoco {}` and `jacocoTestReport {}` blocks
4. **SonarQube config** — `sonarqube {}` block (project-specific overrides can stay)
5. **Test config** — `test {}` block with TestNG and logging config
6. **Repository definitions** — `repositories {}` block (mavenCentral, mavenLocal, Artifactory)
7. **Dependency management** — `dependencyManagement {}` block with BOMs
8. **Java version** — `sourceCompatibility`/`targetCompatibility`
9. **Dist task disabling** — `tasks.distTar.enabled = false`
10. **Local dev tasks** — All Docker Compose management tasks and helpers
11. **Security scanning config** — OWASP and TruffleHog Gradle task config (note: `.github/workflows/` files must stay in the project — GitHub Actions only reads from that path)

### What to Keep

- `group` and `version`
- `jar { archiveFileName }` if customized
- `dependencies {}` block (application-specific dependencies)
- `settings.gradle`
- All Java source code (`src/main/java/`, `src/test/java/`)
- Application configuration (`application.yml`, `application-local.yml`, etc.)
- `docker-compose.yml`

### What to Add

1. `.rpmenv` — RPM configuration
2. `rpm-bootstrap.gradle` — Bootstrap script
3. `rpm-local-dev.properties` — Local dev config (if using local-dev package)
4. `buildscript {}` block in `build.gradle` for external plugin resolution

### Integration Test Migration

Move integration tests from the unit test source set to a dedicated source set:

```
BEFORE:                                    AFTER:
src/test/java/                             src/test/java/
├── com/.../integration/  ← move          ├── com/.../controllers/  (unit tests stay)
├── com/.../controllers/                   └── com/.../service/      (unit tests stay)
└── com/.../service/
                                           src/integrationTest/java/
                                           └── com/.../
                                               ├── CaseServiceIntegrationTest.java
                                               └── ... (moved from src/test)
```

---

## 7. How to Push Updates

When a standard needs to change (new checkstyle rule, updated security scanner, bumped dependency version):

### Step 1: Update the package repo

```bash
cd smarsh-rpm-gradle-checkstyle
# Make changes to checkstyle.xml, checkstyle.gradle, etc.
git add . && git commit -m "Add new import order rule"
```

### Step 2: Tag a new version

```bash
git tag -a 1.1.0 -m "Release 1.1.0: new import order rule"
git push origin 1.1.0
```

### Step 3: Update packages.xml in smarsh-rpm

```xml
<!-- Change revision from 1.0.0 to 1.1.0 -->
<project name="smarsh-rpm-gradle-checkstyle" path=".packages/smarsh-rpm-gradle-checkstyle"
         remote="smarsh" revision="refs/tags/1.1.0" />
```

Tag and push smarsh-rpm with the updated packages.xml.

### Step 4: Projects pick up changes

Each project picks up the new version on next `./gradlew rpmConfigure`. No changes needed in individual project repos.

---

## 8. How to Create a New RPM Package

### Package Structure

```
smarsh-rpm-gradle-yourpackage/
├── yourpackage.gradle          # Gradle script with tasks and configuration
├── rpm-manifest.properties     # (optional) External plugin dependencies
├── config/                     # (optional) Configuration files referenced by the script
│   └── ...
├── README.md
└── CHANGELOG.md
```

### How Scripts Reference Their Own Assets

Each package script uses `_rpmCurrentPkgDir` to find files relative to its own directory:

```groovy
def PKG_DIR = project.ext.get('_rpmCurrentPkgDir')
// Now reference files at ${PKG_DIR}/config/... , ${PKG_DIR}/templates/..., etc.
```

### External Plugin Dependencies

If your package needs an external Gradle plugin (not a core plugin), declare it in `rpm-manifest.properties`:

```properties
# rpm-manifest.properties
buildscript.dependencies=org.some.group:some-plugin:1.2.3
```

Multiple dependencies use comma separation:
```properties
buildscript.dependencies=org.some.group:plugin-a:1.0.0,org.other.group:plugin-b:2.0.0
```

The consuming project's `buildscript {}` block reads these properties and adds the JARs to the classpath automatically.

### Naming Conventions

- Repository: `smarsh-rpm-gradle-<concern>` (e.g., `smarsh-rpm-gradle-checkstyle`)
- Gradle script: `<concern>.gradle` (e.g., `checkstyle.gradle`)
- Versioning: Semantic versioning with Git tags (e.g., `1.0.0`, `1.1.0`, `2.0.0`)

### Adding to packages.xml

Register the new package in `packages.xml`:
```xml
<project name="smarsh-rpm-gradle-yourpackage" path=".packages/smarsh-rpm-gradle-yourpackage"
         remote="smarsh" revision="refs/tags/1.0.0" />
```

---

## 9. How `envsubst` Makes RPM Portable

### What It Is

The [RPM fork](https://github.com/caylent-solutions/git-repo) of Google's `repo` tool provides a `repo envsubst` command that is **not available in the upstream repo tool**. This command replaces `${VARIABLE}` placeholders in all manifest XML files under `.repo/manifests/` with values from environment variables.

### Why It Matters

Without `envsubst`, the manifest `remote.xml` would need a hard-coded URL:

```xml
<!-- WITHOUT envsubst — hard-coded, not portable -->
<remote name="smarsh" fetch="https://github.com/your-org/"/>
```

With `envsubst`, the URL is a placeholder:

```xml
<!-- WITH envsubst — portable across organizations -->
<remote name="smarsh" fetch="${GITBASE}"/>
```

### The 3-Step Bootstrap

```
1. repo init     → Clones the manifest repo. ${GITBASE} placeholders remain as-is in the XML.
2. repo envsubst → Reads GITBASE from the environment and replaces ${GITBASE} in all manifests.
3. repo sync     → Clones packages using the now-resolved URLs.
```

The bootstrap script (`rpm-bootstrap.gradle`) exports `GITBASE` from `.rpmenv` before calling `repo envsubst`:

```groovy
exec {
    environment 'GITBASE', gitbase  // From .rpmenv: GITBASE=https://github.com/your-org/
    commandLine 'bash', '-c', 'repo envsubst'
}
```

### What This Lets You Do

**Point all packages to your own GitHub org:**
```properties
# .rpmenv — just change this one line
GITBASE=https://github.com/your-company/
```

**Use internal Git mirrors in CI/CD:**
```bash
# In your pipeline, override the environment variable
export GITBASE=https://git.internal.your-company.com/
./gradlew rpmConfigure
```

**Fork to a different Git host entirely:**
```properties
# Works with GitLab, Bitbucket, self-hosted Git — any Git host
GITBASE=https://gitlab.your-company.com/rpm-packages/
```

**Same manifests everywhere:**
The XML manifests in `smarsh-rpm` never need to change. Dev laptops, CI runners, air-gapped environments — they all use the same `remote.xml` with `fetch="${GITBASE}"`. Only the `.rpmenv` file (or environment variable) changes.

For full documentation on the `envsubst` feature, see the [git-repo fork README](https://github.com/caylent-solutions/git-repo#environment-variable-substitution-envsubst).

---

## 10. Browsing the Source

All RPM repos are public. You can browse the actual code to understand exactly what each package provides.

### Manifest Repository

- **[smarsh-rpm](https://github.com/caylent-solutions/smarsh-rpm)** — This repo. Contains manifest XML files, examples, and documentation.
  - [`repo-specs/git-connection/remote.xml`](https://github.com/caylent-solutions/smarsh-rpm/blob/main/repo-specs/git-connection/remote.xml) — The `${GITBASE}` remote definition
  - [`repo-specs/java/gradle/springboot/microservice/packages.xml`](https://github.com/caylent-solutions/smarsh-rpm/blob/main/repo-specs/java/gradle/springboot/microservice/packages.xml) — Which packages and versions a microservice project uses
  - [`examples/example-gradle-task-runner/`](https://github.com/caylent-solutions/smarsh-rpm/tree/main/examples/example-gradle-task-runner) — Template files to copy into your project

### Package Repositories

Each package repo contains a `.gradle` script (the tasks and configuration) and optionally `rpm-manifest.properties` (external plugin dependencies) and config files.

- **[smarsh-rpm-gradle-build](https://github.com/caylent-solutions/smarsh-rpm-gradle-build)** — `build-conventions.gradle` defines Java version, Spring Boot, repos, BOMs
- **[smarsh-rpm-gradle-checkstyle](https://github.com/caylent-solutions/smarsh-rpm-gradle-checkstyle)** — `checkstyle.gradle` + `config/checkstyle/checkstyle.xml` (Google Java Style)
- **[smarsh-rpm-gradle-security](https://github.com/caylent-solutions/smarsh-rpm-gradle-security)** — `security.gradle` with OWASP + TruffleHog tasks
- **[smarsh-rpm-gradle-unit-test](https://github.com/caylent-solutions/smarsh-rpm-gradle-unit-test)** — `unit-test.gradle` with TestNG, JaCoCo, coverage reporting
- **[smarsh-rpm-gradle-integration-test](https://github.com/caylent-solutions/smarsh-rpm-gradle-integration-test)** — `integration-test.gradle` with separate source set
- **[smarsh-rpm-gradle-sonarqube](https://github.com/caylent-solutions/smarsh-rpm-gradle-sonarqube)** — `sonarqube.gradle` with standard config
- **[smarsh-rpm-gradle-local-dev](https://github.com/caylent-solutions/smarsh-rpm-gradle-local-dev)** — `local-dev.gradle` with Docker Compose management

### Reading packages.xml

To see which package versions your project archetype uses, check [`packages.xml`](https://github.com/caylent-solutions/smarsh-rpm/blob/main/repo-specs/java/gradle/springboot/microservice/packages.xml). Each `<project>` entry shows:
- `name` — The GitHub repo name
- `path` — Where it syncs to locally (`.packages/<name>`)
- `revision` — The pinned Git tag (e.g., `refs/tags/1.0.0`)

---

## 11. Architecture Decisions

### Why Gradle script plugins (not binary plugins or buildSrc)?

**Script plugins** (`apply from: 'path/to/script.gradle'`) are the simplest mechanism that works. They require no compilation, no plugin portal publishing, no `buildSrc` directory, and no `pluginManagement` blocks. Each package is just a `.gradle` file with plain Groovy/Gradle DSL. Anyone who can read a `build.gradle` can read a package script.

Binary plugins require compilation, JAR publishing, version resolution through a plugin portal, and `pluginManagement` configuration. This is overhead with no benefit for scripts that configure existing plugins.

### Why the `repo` tool (not Git submodules, composite builds, or published plugins)?

**Git submodules** pin to commits (not tags), require manual initialization, create nested `.git` directories, and are fragile in CI/CD. They also lack the `envsubst` portability feature.

**Gradle composite builds** are designed for source-level dependencies between projects, not for distributing configuration scripts. They require `settings.gradle` changes and `includeBuild()` calls.

**Published plugins** on a plugin portal require a build pipeline for every package, a hosted plugin repository, and `pluginManagement` blocks in `settings.gradle`. This is the most infrastructure-heavy option.

The **`repo` tool** provides tag-pinned synchronization of multiple Git repos into a local directory with one command (`repo sync`). The RPM fork adds `envsubst` for URL portability. The packages are plain files on disk — no compilation, no resolution, no portal.

### Why separate repos per concern (not one monorepo)?

Each package has its own **versioning lifecycle**. Bumping a checkstyle rule should not require re-tagging the security scanner package. Separate repos enable:
- Independent version tags per concern
- Targeted updates (change one package, tag it, update `packages.xml`)
- 1:1 mapping to CI/CD pipeline stages
- Clear ownership boundaries

### Why envsubst (portability across organizations)?

Without `envsubst`, the manifest XMLs would contain hard-coded URLs like `fetch="https://github.com/your-org/"`. Any organization wanting to use RPM would need to fork `smarsh-rpm` and change every URL in every XML file.

With `envsubst`, the manifests use `fetch="${GITBASE}"` and the actual URL is resolved from `.rpmenv` at configure time. Adopting RPM for a different organization means changing one line in `.rpmenv`.

### Why public repos?

No proprietary or sensitive data exists in any RPM repo:
- No secrets or credentials (externalized to consuming project's config)
- No org-specific URLs (resolved by envsubst at configure time)
- No proprietary business logic (only build tool configuration)
- Checkstyle rules are based on public Google Java Style
- Security scanning uses open-source tools with standard configuration

Public repos enable:
- Direct browsing by engineers evaluating RPM
- Documentation linking to live source code
- No access management overhead
