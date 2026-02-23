# Java/Gradle/Spring Boot/Microservice

RPM manifest for Spring Boot microservice projects built with Gradle.

## Packages Included

| Package | Purpose |
|---|---|
| smarsh-rpm-gradle-build | Java 17, Spring Boot, repos, dependency management |
| smarsh-rpm-gradle-checkstyle | Google Java Style + Smarsh customizations |
| smarsh-rpm-gradle-security | OWASP dependency check + TruffleHog |
| smarsh-rpm-gradle-unit-test | TestNG + JaCoCo coverage |
| smarsh-rpm-gradle-integration-test | Separate integration test source set |
| smarsh-rpm-gradle-sonarqube | SonarQube code quality analysis |
| smarsh-rpm-gradle-local-dev | Docker Compose management, health checks |

## Usage

Set in `.rpmenv`:
```properties
REPO_MANIFESTS_PATH=repo-specs/java/gradle/springboot/microservice/meta.xml
```

## Example

See the [example/](example/) directory for template files.
