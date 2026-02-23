# Contributing to RPM

How to create, maintain, and update RPM packages and manifests.

## Package Development

### Creating a New Package

1. Create a new GitHub repository at `caylent-solutions/smarsh-rpm-gradle-<concern>`
2. Add a `.gradle` script with your tasks and configuration
3. If your script needs external Gradle plugins, create `rpm-manifest.properties`
4. Add `README.md` and `CHANGELOG.md`
5. Tag `1.0.0`
6. Add the package to `packages.xml` in this repo

### Package Structure Requirements

```
smarsh-rpm-gradle-<concern>/
├── <concern>.gradle            # Required: Gradle script with tasks/config
├── rpm-manifest.properties     # Optional: External plugin dependencies
├── config/                     # Optional: Configuration files
├── README.md                   # Required: Package documentation
└── CHANGELOG.md                # Required: Version history
```

### Package Script Guidelines

1. **Use `_rpmCurrentPkgDir`** to reference package-local files:
   ```groovy
   def PKG_DIR = project.ext.get('_rpmCurrentPkgDir')
   ```

2. **Do not hard-code organization-specific values.** Use project properties or `.rpmenv` for URLs, server addresses, etc.

3. **Use `apply plugin:` for core plugins** (java, checkstyle, jacoco). Use `rpm-manifest.properties` for external plugins.

4. **Document all provided tasks** in the package README.

### Versioning

- Use [semantic versioning](https://semver.org/) for Git tags
- MAJOR: Breaking changes (renamed tasks, removed config, changed behavior)
- MINOR: New features (new tasks, new config options)
- PATCH: Bug fixes (corrected config, fixed task behavior)

## Manifest Development

### Adding a New Project Archetype

1. Create a new directory under `repo-specs/`:
   ```
   repo-specs/java/gradle/springboot/library/
   ```
2. Create `meta.xml` that includes `remote.xml` and a new `packages.xml`
3. Create `packages.xml` listing the packages for this archetype
4. Add example files

### Updating Package Versions

1. Edit `packages.xml` to change the `revision` attribute
2. Tag and push the `smarsh-rpm` repo

### Validating Manifests

Run the validation script:
```bash
make validate-xml
```

This checks:
- Well-formed XML
- Required attributes on `<project>`, `<remote>`, and `<include>` elements
- `<include>` references point to existing files

## Code Review Checklist

- [ ] Package script uses `_rpmCurrentPkgDir` for file references
- [ ] No hard-coded organization-specific values
- [ ] External plugin dependencies declared in `rpm-manifest.properties`
- [ ] README documents all provided tasks
- [ ] CHANGELOG updated with version entry
- [ ] Manifest XML passes `make validate-xml`
