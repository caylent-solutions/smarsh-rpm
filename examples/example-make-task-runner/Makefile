# ============================================================================
# RPM Bootstrap - Repo Package Manager for Make
# ============================================================================
# This Makefile provides:
#   1. Auto-inclusion of RPM package Makefiles from .packages/
#   2. rpm-configure target: installs tooling and syncs packages
#   3. rpm-clean target: removes synced packages and repo metadata
#
# Prerequisites: Python 3 and pipx must be available on PATH.
#
# Commit this file and .rpmenv to your project. They rarely change.
# Configuration lives in .rpmenv (Make ?= syntax for env var overrides).
# ============================================================================

SHELL := /bin/bash
.DEFAULT_GOAL := help

# ---------------------------------------------------------------------------
# 1. Read .rpmenv configuration
# ---------------------------------------------------------------------------
RPM_ENV_FILE := .rpmenv
ifeq (,$(wildcard $(RPM_ENV_FILE)))
$(error RPM: $(RPM_ENV_FILE) not found. Create it from the smarsh-rpm example.)
endif
include $(RPM_ENV_FILE)

# ---------------------------------------------------------------------------
# 2. Auto-include package Makefiles from .packages/
# ---------------------------------------------------------------------------
-include $(PACKAGES_DIR)/*/Makefile

# ---------------------------------------------------------------------------
# 3. rpm-configure target
# ---------------------------------------------------------------------------
.PHONY: rpm-configure
rpm-configure: ## Install tooling and sync RPM packages from manifest repository
	@echo "RPM: Checking prerequisites (Python 3, pipx)..."
	@command -v python3 > /dev/null 2>&1 || command -v python > /dev/null 2>&1 || \
		{ echo "RPM ERROR: python3 (or python) is not installed or not on PATH."; \
		  echo ""; \
		  echo "Install Python 3 before running rpm-configure."; \
		  echo "  - DevContainer: Python is provided by the devcontainer Python feature"; \
		  echo "  - CI/CD: Add a Python installation step before running make"; \
		  echo "  - Local: Install Python 3 via your system package manager"; \
		  exit 1; }
	@command -v pipx > /dev/null 2>&1 || \
		{ echo "RPM ERROR: pipx is not installed or not on PATH."; \
		  echo ""; \
		  echo "Install pipx before running rpm-configure."; \
		  echo "  - DevContainer: pipx is included with the devcontainer Python feature (installTools=true)"; \
		  echo "  - pip: python3 -m pip install --user pipx"; \
		  echo "  - apt: sudo apt install pipx"; \
		  echo "  - brew: brew install pipx"; \
		  echo "After installing: pipx ensurepath"; \
		  exit 1; }
	@echo "RPM: Python 3 and pipx are available"
	@echo "RPM: Installing repo tool $(REPO_REV)..."
	@pipx install --force "git+$(REPO_URL)@$(REPO_REV)" || \
		{ echo "RPM ERROR: Failed to install repo tool via pipx"; exit 1; }
	@echo "RPM: Initializing repo from $(REPO_MANIFESTS_URL)..."
	@echo n | repo --color=never init --no-repo-verify \
		-u "$(REPO_MANIFESTS_URL)" \
		-b "$(REPO_MANIFESTS_REVISION)" \
		-m "$(REPO_MANIFESTS_PATH)" \
		--repo-rev="$(REPO_REV)" \
		|| { echo "RPM ERROR: repo init failed"; exit 1; }
	@echo "RPM: Running repo envsubst to resolve \$${GITBASE}..."
	@GITBASE="$(GITBASE)" repo envsubst || \
		{ echo "RPM ERROR: repo envsubst failed"; exit 1; }
	@echo "RPM: Syncing packages..."
	@repo sync || { echo "RPM ERROR: repo sync failed"; exit 1; }
	@if [ ! -f .gitignore ] || ! grep -q "^$(PACKAGES_DIR)/$$" .gitignore; then \
		echo "$(PACKAGES_DIR)/" >> .gitignore; \
		echo "RPM: Added $(PACKAGES_DIR)/ to .gitignore"; \
	fi
	@if [ ! -f .gitignore ] || ! grep -q "^\.repo/$$" .gitignore; then \
		echo ".repo/" >> .gitignore; \
		echo "RPM: Added .repo/ to .gitignore"; \
	fi
	@echo "RPM: Configuration complete. Packages synced to $(PACKAGES_DIR)/"

# ---------------------------------------------------------------------------
# 4. rpm-clean target
# ---------------------------------------------------------------------------
.PHONY: rpm-clean
rpm-clean: ## Remove synced RPM packages and repo metadata
	@if [ -d "$(PACKAGES_DIR)" ]; then \
		rm -rf "$(PACKAGES_DIR)"; \
		echo "RPM: Removed $(PACKAGES_DIR)/"; \
	fi
	@if [ -d ".repo" ]; then \
		rm -rf ".repo"; \
		echo "RPM: Removed .repo/"; \
	fi
	@echo "RPM: Clean complete"

# ---------------------------------------------------------------------------
# 5. help target
# ---------------------------------------------------------------------------
.PHONY: help
help: ## Show available targets
	@echo "RPM Targets:"
	@echo ""
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(firstword $(MAKEFILE_LIST)) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  %-20s %s\n", $$1, $$2}'
	@echo ""
	@if [ -d "$(PACKAGES_DIR)" ]; then \
		echo "Package Targets:"; \
		echo ""; \
		for makefile in $(PACKAGES_DIR)/*/Makefile; do \
			if [ -f "$$makefile" ]; then \
				grep -E '^[a-zA-Z_-]+:.*?## .*$$' "$$makefile" | \
					awk 'BEGIN {FS = ":.*?## "}; {printf "  %-20s %s\n", $$1, $$2}'; \
			fi; \
		done; \
	else \
		echo "No packages synced. Run 'make rpm-configure' first."; \
	fi
