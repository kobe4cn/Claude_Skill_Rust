# Rust Lib Template - Complete Reference

## Overview

The `rust-lib-template` provides a production-ready foundation for Rust library projects with integrated tooling for CI/CD, security auditing, changelog generation, and code quality enforcement.

**Repository**: https://github.com/kobe4cn/rust-lib-template

## Prerequisites

### Required Tools

| Tool | Installation | Purpose |
|------|--------------|---------|
| cargo-generate | `cargo install cargo-generate` | Template scaffolding |
| pre-commit | `pip install pre-commit` or `brew install pre-commit` | Git hooks |
| cargo-deny | `cargo install cargo-deny` | Security auditing |
| git-cliff | `cargo install git-cliff` | Changelog generation |
| typos | `cargo install typos-cli` | Spell checking |

### Installation Verification

```bash
# Verify all tools are installed
cargo generate --version
pre-commit --version
cargo deny --version
git-cliff --version
typos --version
```

## Directory Structure

```
{{ project-name }}/
├── Cargo.toml              # Package manifest with template placeholders
├── cargo-generate.toml     # Template variable definitions
├── .pre-commit-config.yaml # Pre-commit hook configurations
├── cliff.toml              # git-cliff changelog settings
├── deny.toml               # cargo-deny security/license settings
├── _typos.toml             # typos spell checker configuration
├── .gitignore              # Git ignore patterns
│
├── src/                    # Source code
│   └── lib.rs              # Library entry point
│
├── examples/               # Example implementations
│   └── (example files)     # Usage demonstrations
│
├── fixtures/               # Test fixtures
│   └── (test data)         # Static test data files
│
├── specs/                  # Specifications
│   └── (spec files)        # API contracts, schemas
│
├── ui/                     # UI components (if applicable)
│   └── (ui files)          # Frontend assets
│
└── _github/                # GitHub configuration (pre-generation)
    └── workflows/          # GitHub Actions workflows
        └── (workflow.yml)  # CI/CD pipeline definitions
```

## Configuration Files

### Cargo.toml

The template Cargo.toml includes standard library configuration:

```toml
[package]
name = "{{ project-name }}"
version = "0.1.0"
edition = "2021"
description = "{{ description }}"
license = "MIT"
repository = "https://github.com/{{ github-username }}/{{ project-name }}"

[dependencies]
# Add your dependencies here

[dev-dependencies]
# Testing dependencies

[features]
default = []
# Feature flags

[lib]
# Library configuration
```

### cargo-generate.toml

Defines template variables:

```toml
[template]
cargo_generate_version = ">=0.10.0"

[placeholders.project-name]
type = "string"
prompt = "Project name?"
regex = "^[a-zA-Z][a-zA-Z0-9_-]*$"

[placeholders.description]
type = "string"
prompt = "Project description?"
default = "A Rust library"

[placeholders.github-username]
type = "string"
prompt = "GitHub username?"
```

### .pre-commit-config.yaml

Pre-commit hooks configuration:

```yaml
repos:
  - repo: local
    hooks:
      - id: cargo-fmt
        name: cargo fmt
        entry: cargo fmt --
        language: system
        types: [rust]
        pass_filenames: false

      - id: cargo-clippy
        name: cargo clippy
        entry: cargo clippy -- -D warnings
        language: system
        types: [rust]
        pass_filenames: false

      - id: cargo-test
        name: cargo test
        entry: cargo test
        language: system
        types: [rust]
        pass_filenames: false

      - id: typos
        name: typos
        entry: typos
        language: system
        pass_filenames: false
```

**Usage**:
```bash
# Install hooks (one-time setup)
pre-commit install

# Run all hooks manually
pre-commit run --all-files

# Run specific hook
pre-commit run cargo-fmt --all-files
```

### deny.toml

Cargo-deny configuration for security and license auditing:

```toml
[advisories]
vulnerability = "deny"
unmaintained = "warn"
yanked = "warn"
notice = "warn"

[licenses]
unlicensed = "deny"
allow = [
    "MIT",
    "Apache-2.0",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC",
    "Zlib",
]
copyleft = "warn"

[bans]
multiple-versions = "warn"
wildcards = "warn"
highlight = "all"

[sources]
unknown-registry = "deny"
unknown-git = "deny"
```

**Usage**:
```bash
# Run all checks
cargo deny check

# Check specific category
cargo deny check advisories
cargo deny check licenses
cargo deny check bans
cargo deny check sources
```

### cliff.toml

Git-cliff changelog configuration:

```toml
[changelog]
header = """
# Changelog

All notable changes to this project will be documented in this file.
"""
body = """
{% for group, commits in commits | group_by(attribute="group") %}
## {{ group | upper_first }}
{% for commit in commits %}
- {{ commit.message | upper_first }}\
{% endfor %}
{% endfor %}
"""
footer = ""
trim = true

[git]
conventional_commits = true
filter_unconventional = true
split_commits = false
commit_preprocessors = []
commit_parsers = [
    { message = "^feat", group = "Features" },
    { message = "^fix", group = "Bug Fixes" },
    { message = "^doc", group = "Documentation" },
    { message = "^perf", group = "Performance" },
    { message = "^refactor", group = "Refactoring" },
    { message = "^style", group = "Style" },
    { message = "^test", group = "Testing" },
    { message = "^chore", group = "Miscellaneous" },
]
filter_commits = false
tag_pattern = "v[0-9]*"
```

**Usage**:
```bash
# Generate changelog
git cliff -o CHANGELOG.md

# Generate for specific version range
git cliff --tag v1.0.0 -o CHANGELOG.md

# Preview without writing
git cliff --dry-run
```

### _typos.toml

Spell checker configuration:

```toml
[default]
extend-ignore-identifiers-re = [
    # Add patterns to ignore
]

[default.extend-words]
# Add custom words
# crate = "crate"

[files]
extend-exclude = [
    "target/",
    "Cargo.lock",
]
```

**Usage**:
```bash
# Check for typos
typos

# Fix typos automatically
typos --write-changes

# Check specific files
typos src/
```

## CI/CD Workflows

### Activating GitHub Actions

After generation, rename the `_github` directory:

```bash
mv _github .github
```

### Typical Workflow Structure

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo test
      - run: cargo deny check

  release:
    needs: test
    if: startsWith(github.ref, 'refs/tags/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo publish
        env:
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

## Post-Generation Workflow

### Complete Setup Checklist

```bash
# 1. Generate project
cargo generate --git https://github.com/kobe4cn/rust-lib-template
cd {{ project-name }}

# 2. Activate CI/CD
mv _github .github

# 3. Install pre-commit hooks
pre-commit install

# 4. Initialize git (if needed)
git init

# 5. Review and customize configurations
# - Edit Cargo.toml metadata
# - Review deny.toml license settings
# - Update cliff.toml commit parsers if needed
# - Customize _typos.toml dictionary

# 6. Initial commit
git add .
git commit -m "feat: initial project from lib template"

# 7. Verify setup
pre-commit run --all-files
cargo deny check
cargo test
```

## Development Workflow

### Daily Development

```bash
# Make changes to code

# Run formatting
cargo fmt

# Run linter
cargo clippy -- -D warnings

# Run tests
cargo test

# Commit (hooks run automatically)
git add .
git commit -m "feat: add new feature"
```

### Release Process

```bash
# 1. Update version in Cargo.toml
# 2. Generate changelog
git cliff --tag vX.Y.Z -o CHANGELOG.md

# 3. Commit release
git add .
git commit -m "chore: release vX.Y.Z"

# 4. Create tag
git tag -a vX.Y.Z -m "Release vX.Y.Z"

# 5. Push
git push origin main --tags

# CI will handle publishing if configured
```

## Troubleshooting

### cargo-generate Not Found

```bash
# Install cargo-generate
cargo install cargo-generate

# Or update if outdated
cargo install cargo-generate --force
```

### Pre-commit Hooks Failing

```bash
# Update hooks
pre-commit autoupdate

# Clean and reinstall
pre-commit clean
pre-commit install

# Run with verbose output
pre-commit run --all-files --verbose
```

### cargo-deny Failures

```bash
# Check specific issue
cargo deny check --show-stats

# Update deny.toml to allow specific licenses if needed
# Or update dependencies to fix vulnerabilities
cargo update
```

### GitHub Actions Not Running

- Ensure `_github` was renamed to `.github`
- Check workflow file syntax
- Verify branch names match your default branch

## Best Practices

### Library Design

1. **Public API**
   - Export stable types through `lib.rs`
   - Use `pub use` for re-exports
   - Document all public items

2. **Documentation**
   - Add module-level documentation
   - Include code examples in doc comments
   - Run `cargo doc --open` to preview

3. **Testing**
   - Unit tests in `src/` modules
   - Integration tests in `tests/`
   - Use `fixtures/` for test data

### Versioning

- Follow [Semantic Versioning](https://semver.org/)
- Use [Conventional Commits](https://www.conventionalcommits.org/)
- Keep CHANGELOG.md updated with git-cliff

### Security

- Run `cargo deny check` before releases
- Keep dependencies updated
- Review security advisories regularly

## Source

This skill documents the `rust-lib-template` at https://github.com/kobe4cn/rust-lib-template
