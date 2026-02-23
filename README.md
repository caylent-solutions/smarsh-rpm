# Smarsh RPM (Repo Package Manager)

Centralized platform and development automation for Smarsh projects.

---

## What is RPM?

RPM is a **DevOps Platform Dependency Manager** that brings version-controlled, reproducible automation to your projects through declarative manifests. Adapted from [Caylent's CPM](https://github.com/caylent-solutions/cpm) (Caylent Package Manager), RPM enables you to centralize, version, and share automation across your organization without replacing your existing tools.

**Solves a common problem:**
Organizations have quality automation scattered across teams — build conventions, linting rules, security scanning, test frameworks, and local dev tooling that work well but aren't widely adopted because they're hard to discover, version, test, and distribute. RPM enables you to package this automation and share it across projects in a tested, reproducible way.

**Fully customizable:**
- **Public or Private** — Use public repositories or host everything privately within your organization
- **Your Infrastructure** — Point to your own Git repositories and package sources
- **Your Standards** — Define your own manifests, packages, and automation
- **Portable** — Teams retain access to automation even after external partnerships end

**Core Purpose:**
- **Platform Dependency Management** — Centralize and version your DevOps automation, dependencies, and standards
- **Flexible Overlay** — Works alongside your preferred task runners (Make, npm, Gradle, Maven) and dependency managers
- **Team Standards** — Share tested, versioned automation, tasks, and approaches across teams dynamically
- **Tool Agnostic** — Adapts to your workflow, not the other way around

---

## Quick Start

### 1. Setup

Copy the task runner files from your chosen manifest's `example/` directory.

**For Gradle/Spring Boot Microservices:**
```bash
curl -O https://raw.githubusercontent.com/caylent-solutions/smarsh-rpm/main/examples/example-gradle-task-runner/.rpmenv
curl -O https://raw.githubusercontent.com/caylent-solutions/smarsh-rpm/main/examples/example-gradle-task-runner/rpm-bootstrap.gradle
curl -O https://raw.githubusercontent.com/caylent-solutions/smarsh-rpm/main/examples/example-gradle-task-runner/build.gradle
```

The `.rpmenv` file is pre-configured with the correct manifest path for that ecosystem.

### 2. Verify Configuration

Check that `.rpmenv` has the correct `REPO_MANIFESTS_PATH` for your manifest:

```bash
cat .rpmenv | grep REPO_MANIFESTS_PATH
```

### 3. Configure

```bash
./gradlew rpmConfigure
```

This automatically installs asdf, Python, and the repo tool. It also syncs all packages to `.packages/` and adds `.packages/` and `.repo/` to `.gitignore`.

**Important:** All synced files in `.packages/` and `.repo/` are ephemeral and should not be committed. Only commit your task runner bootstrap files and `.rpmenv` to your repository.

### 4. Use

RPM is **tool agnostic** — it works with any task runner or build system. Each manifest provides different artifacts (automation tasks, dependencies, configurations, or code assets) tailored to your ecosystem.

**Gradle-based projects:**
```bash
./gradlew tasks      # See all available tasks
./gradlew build      # Run build with RPM-provided conventions
```

RPM's orchestration pattern adapts to your workflow. Current examples use Gradle, but the architecture supports Make, npm, Maven, and any other task runner.

**[Full Setup Guide ->](docs/setup-guide.md)**

---

## Use Cases

**Unify Disparate Automation:**
Your organization has quality automation scattered across teams — testing frameworks, linting configs, deployment scripts, security scans — but they're not widely adopted because they're hard to find, version, and integrate. RPM lets you package this automation, version it, and make it available to all teams through simple manifests.

**Platform Engineering:**
Provide golden paths and paved roads to development teams. Package your organization's standards, policies, and automation as versioned dependencies that teams can pull into their projects.

**Multi-Project Consistency:**
Ensure the same testing, linting, security scanning, and deployment automation across projects without copy-pasting or manual synchronization.

---

## Available Manifests

- **[Git Connection](repo-specs/git-connection/)** — Shared remote definitions for all manifests
- **[Java/Gradle/Spring Boot Microservice](repo-specs/java/gradle/springboot/microservice/)** — Build conventions, testing, security scanning, code quality, and local dev tooling for Spring Boot microservices

**Note:** These are reference implementations. You can create your own manifests pointing to your organization's repositories. See [Contributing Guide](docs/contributing.md) for creating custom manifests.

---

## How It Works

RPM uses [a fork of the Gerrit `repo` tool](https://github.com/caylent-solutions/git-repo) to orchestrate dependencies across Git repositories. Manifests define what to clone, where to place it, and how to wire it together. The Caylent fork provides `repo envsubst`, which resolves `${VARIABLE}` placeholders in manifest XML with values from your `.rpmenv` configuration — making manifests portable across organizations and environments.

**[Complete Technical Walkthrough ->](docs/how-it-works.md)**

---

## Documentation

- [RPM Guide](docs/rpm-guide.md) — Comprehensive guide for engineers new to this pattern
- [How It Works](docs/how-it-works.md) — Technical deep-dive into RPM internals
- [Setup Guide](docs/setup-guide.md) — Step-by-step setup for new and existing projects
- [Pipeline Integration](docs/pipeline-integration.md) — Using RPM tasks in CI/CD pipelines
- [Contributing](docs/contributing.md) — How to create and maintain RPM packages

---

## Architecture

```text
                   ┌──────────────────────────┐
                   │   Smarsh Repo Package    │
                   │     Manager (RPM)        │
                   └────────────┬─────────────┘
                                │
               defines          │            uses
                                ▼
              ┌────────────────────────────────────┐
              │   smarsh-rpm (Manifests Repo)      │
              │  - Top-level dependency manifests  │
              │  - Declares relationships between  │
              │    domain and automation repos     │
              └──────────────────┬─────────────────┘
                                 │
        references               │                references
                                 │
             ▼                                       ▼
┌───────────────────────┐                ┌────────────────────────┐
│  Package Repositories │                │ Automation Repositories│
│ (build conventions,   │                │ (shared tasks,         │
│  linting, security)   │                │  validation, scanning) │
└────────────┬──────────┘                └───────────┬────────────┘
             │                                       │
             └───────────────────┬───────────────────┘
                                 │
                                 ▼
                   ┌────────────────────────────┐
                   │ Caylent Gerrit `repo` Fork │
                   │ (git-repo with envsubst)   │
                   │ Executes manifests, syncs  │
                   │ repos, manages workspace   │
                   └────────────────────────────┘
```

---

## License

MIT
