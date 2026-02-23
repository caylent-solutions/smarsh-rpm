# Changelog

## 1.0.0

Initial release of RPM (Repo Package Manager) for Java/Gradle/Spring Boot projects.

### Added
- Manifest repository structure with repo-specs hierarchy
- Support for Java/Gradle/Spring Boot/Microservice project archetype
- `remote.xml` with `${GITBASE}` envsubst for portable manifests
- `packages.xml` referencing 7 RPM package repos
- `rpm-bootstrap.gradle` bootstrap script with `rpmConfigure` and `rpmClean` tasks
- `.rpmenv` configuration template
- Example Gradle task runner project
- XML manifest validation script
- Comprehensive documentation (`docs/rpm-guide.md`)
