# smarsh-rpm

RPM (Repo Package Manager) is a Gradle-native build tooling package manager for Java/Spring Boot projects. It centralizes build configuration, linting rules, security scanning, test framework setup, and code quality tooling into versioned packages that any project can consume.

RPM is adapted from [Caylent's CPM](https://github.com/caylent-solutions/cpm) (Caylent Package Manager), which provides the same pattern for Terraform/Make projects.

## How It Works

```
.rpmenv (config) → rpm-bootstrap.gradle (bootstrap) → repo init/envsubst/sync → .packages/*/*.gradle (auto-apply)
```

1. A project commits three files: `.rpmenv`, `rpm-bootstrap.gradle`, and `build.gradle`
2. Running `./gradlew rpmConfigure` installs tooling and syncs packages from this manifest repo
3. Package `.gradle` scripts are auto-discovered and applied, providing Gradle tasks and configuration
4. The project's `build.gradle` contains only project-specific config (group, version, dependencies)

## Quick Start

1. Copy the three files from [examples/example-gradle-task-runner/](examples/example-gradle-task-runner/) to your project root
2. Customize `.rpmenv` with your manifest path and Git base URL
3. Run `./gradlew rpmConfigure` to sync packages
4. Run `./gradlew tasks` to see all available RPM-provided tasks

## Packages

| Package | Purpose | Gradle Tasks |
|---|---|---|
| [smarsh-rpm-gradle-build](https://github.com/caylent-solutions/smarsh-rpm-gradle-build) | Java 17, Spring Boot, repos, dependency management | `build`, `bootJar` |
| [smarsh-rpm-gradle-checkstyle](https://github.com/caylent-solutions/smarsh-rpm-gradle-checkstyle) | Google Java Style + Smarsh customizations | `checkstyleMain` |
| [smarsh-rpm-gradle-security](https://github.com/caylent-solutions/smarsh-rpm-gradle-security) | OWASP dependency check + TruffleHog | `securityCheck`, `secretScan` |
| [smarsh-rpm-gradle-unit-test](https://github.com/caylent-solutions/smarsh-rpm-gradle-unit-test) | TestNG + JaCoCo coverage | `unitTest`, `jacocoTestReport` |
| [smarsh-rpm-gradle-integration-test](https://github.com/caylent-solutions/smarsh-rpm-gradle-integration-test) | Separate integration test source set | `integrationTest` |
| [smarsh-rpm-gradle-sonarqube](https://github.com/caylent-solutions/smarsh-rpm-gradle-sonarqube) | SonarQube code quality analysis | `sonar` |
| [smarsh-rpm-gradle-local-dev](https://github.com/caylent-solutions/smarsh-rpm-gradle-local-dev) | Docker Compose management, health checks | `localDevUp`, `localDevDown`, `bootRunLocal` |

## Manifest Structure

```
repo-specs/
├── git-connection/
│   └── remote.xml              # fetch="${GITBASE}" (resolved by repo envsubst)
└── java/gradle/springboot/
    └── microservice/
        ├── meta.xml            # Includes remote + packages
        └── packages.xml        # Lists all package repos with version tags
```

The `${GITBASE}` placeholder in `remote.xml` is resolved at configure time by the [Caylent fork](https://github.com/caylent-solutions/git-repo) of the repo tool's `envsubst` command. This makes the manifests portable — change `GITBASE` in `.rpmenv` and all packages resolve to a different Git host.

## Documentation

- [RPM Guide](docs/rpm-guide.md) — Comprehensive guide for engineers new to this pattern
- [How It Works](docs/how-it-works.md) — Technical deep-dive into RPM internals
- [Setup Guide](docs/setup-guide.md) — Step-by-step setup for new and existing projects
- [Pipeline Integration](docs/pipeline-integration.md) — Using RPM tasks in CI/CD pipelines
- [Contributing](docs/contributing.md) — How to create and maintain RPM packages

## Repository List

| Repository | Purpose |
|---|---|
| [smarsh-rpm](https://github.com/caylent-solutions/smarsh-rpm) | This repo — manifest definitions and documentation |
| [smarsh-rpm-gradle-build](https://github.com/caylent-solutions/smarsh-rpm-gradle-build) | Build conventions and dependency management |
| [smarsh-rpm-gradle-checkstyle](https://github.com/caylent-solutions/smarsh-rpm-gradle-checkstyle) | Code style enforcement |
| [smarsh-rpm-gradle-security](https://github.com/caylent-solutions/smarsh-rpm-gradle-security) | Security scanning |
| [smarsh-rpm-gradle-unit-test](https://github.com/caylent-solutions/smarsh-rpm-gradle-unit-test) | Unit testing and code coverage |
| [smarsh-rpm-gradle-integration-test](https://github.com/caylent-solutions/smarsh-rpm-gradle-integration-test) | Integration testing |
| [smarsh-rpm-gradle-sonarqube](https://github.com/caylent-solutions/smarsh-rpm-gradle-sonarqube) | Code quality analysis |
| [smarsh-rpm-gradle-local-dev](https://github.com/caylent-solutions/smarsh-rpm-gradle-local-dev) | Local development environment |

## License

MIT
