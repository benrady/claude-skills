---
name: makefile-automation-basic
description: All developer tasks should be fully automated using targets defined in a Makefile. Use this skill when automating developer tasks such as running tests, linting, formatting, compiling, or deploying. This applies when creating new repositories, or automating developer tasks in an existing repository.
---

## Overview

This skill explains principles and practices for automating common developer tasks using Makefiles.
## Key Principles

## 1. Automate Everything

When someone clones a repository, they shouldn't have to install, configure, create, update, or modify anything manually in order to compile, run tests, format code, or perform any other common task. Everything should be installed automatically when used. Dependencies should be resolved in the correct order so that tasks that depend on other tasks are run in the correct order. This should work on both OSX and Linux.

## 2. Use Makefiles

You should use _make targets_ to run automated developer tasks. Each repository should have a @Makefile that provides targets for all necessary developer tasks, and automatically installs any command line tools needed to make those targets works. `make` and `curl` are the only two command line tools assumed to be pre-installed in a development environment. Everything else should be installed on-demand when a target is run via `make`.

### Provide a `help` Target

The default target in the Makefile should be a `help` target. This needs to be the first target in order to be the default. This target greps the Makefile for target with a double-hash `##` comment. These comments should be aggregated together to create a helpful menu that is displayed when running `make` without a target specified.

```makefile
.PHONY: help
help:
	@grep -hE '^[%0-9a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

```

### Use Bash as the SHELL

In the Makefile, always specify the shell explicitly to ensure consistent behavior across different environments:
```makefile
SHELL:=$(shell which bash)
```

### Install Command Line Tools Into a `.tools` Directory

```makefile
GIT_ROOT=$(shell git rev-parse --show-toplevel)
TOOLS_HOME=$(GIT_ROOT)/.tools
TOOLS_BIN=$(TOOLS_HOME)/bin
```

* Command line tools needed to perform tasks should be installed into a `.tools` directory in the local repository. You should never rely on the operating system, shared packages, or tools installed into "global" locations like `/usr/bin`.
* Tool installation should be idempotent. Tools should be installed when needed and not be re-installed simply because they are used. Creating a make target for the path to the tool allows make to manage this process easily.
* Tool versions should be specified in the Makefile, and the path that the tool is installed to should include the version, so that if the version changes in the Makefile, the new version is automatically installed.
* The `.tools` directory should be added to a `.gitignore` file, so that the tools are not accidentally committed to git.

### Create Targets That Install Tools via `curl`

Use Make's built-in dependency system to ensure tools are installed in the correct order.

* When a command line tool is needed, create a Makefile variable for the path to that tool within the `.tools` directory
* Create a make target for that variable, and within the target, use `curl` to download and install the tool to the location specified by the make variable. This will ensure that make will not run the target if the tool already exists.
* Other targets that rely on this tool should include this installation target as a dependency, so that make will automatically install the tool when needed.
* Use order-only prerequisites (|) to check for dependencies without rebuilding them unnecessarily.

### Fully qualify paths relative to the Makefile

* Prefer make's `$(CURDIR)` variable to `pwd` and other methods of getting the current directory.
* Do not rely on the `PATH` environment variable. Qualify all paths relative to the repository root. Use make variables to remove duplication when needed.

### Default To Silent Mode
Implement a VERBOSE flag to control output verbosity:
```makefile
ifndef VERBOSE
.SILENT:
endif
```

### Install Git Hooks From Every Target

Running any target in the Makefile should trigger the installation of a Python environment with the `pre-commit` library. This should be used to manage pre-commit hooks. Configuration for the pre-commit hooks are kept in a @.pre-commit-config.yaml file

Automate git hook installation:
```makefile
GIT_PRE_COMMIT=$(GIT_ROOT)/.git/hooks/pre-commit
$(GIT_PRE_COMMIT): $(PRE_COMMIT)
	$(PRE_COMMIT) install

.PHONY: hooks
hooks: $(GIT_PRE_COMMIT) # Install pre-commit hooks

test: hooks ## Run unit tests
    # (Run unit test commands here)
```

### Other Makefile Best Practices

* **Phony Targets**: Mark non-file targets as .PHONY to avoid conflicts with files of the same name
* **Error Handling**: Use $(error ...) to provide clear error messages for missing dependencies
* **Platform Detection**: Use shell commands to detect platform and architecture for cross-platform compatibility
* **Default Values**: Use ?= for variables that can be overridden from the command line
