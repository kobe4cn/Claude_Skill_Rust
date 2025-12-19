---
description: >-
  Use when creating new Rust library projects with production-ready setup.
  Provides guidance for using the standardized lib template via cargo-generate,
  including CI/CD workflows, pre-commit hooks, security auditing, and changelog automation.
  Triggers: lib template, cargo generate, rust library template, new library project, 新库项目.
---

# Rust Lib Template

## Quick Reference

Create production-ready Rust library projects using the standardized template.

## Prerequisites

```bash
# Check if cargo-generate is installed
cargo generate --version

# If not installed, install it first
cargo install cargo-generate
```

## Quick Start

```bash
# Create new library project from template
cargo generate --git https://github.com/kobe4cn/rust-lib-template

# Follow prompts to enter project name
# Project will be created in current directory
```

## Template Structure

```
{{ project-name }}/
├── Cargo.toml              # Package manifest
├── cargo-generate.toml     # Template generation config
├── .pre-commit-config.yaml # Pre-commit hooks
├── cliff.toml              # Changelog generation (git-cliff)
├── deny.toml               # Dependency security (cargo-deny)
├── _typos.toml             # Spell checking (typos)
├── .gitignore
├── src/                    # Source code
├── examples/               # Example implementations
├── fixtures/               # Test fixtures/data
├── specs/                  # Specifications/contracts
├── ui/                     # UI-related components
└── _github/                # GitHub Actions (pre-generation)
    └── workflows/
```

## Post-Generation Checklist

After generating your project:

1. **Activate CI/CD**
   ```bash
   mv _github .github
   ```

2. **Install pre-commit hooks**
   ```bash
   pre-commit install
   ```

3. **Review workflow settings**
   - Check branch names in `.github/workflows/` (default: main)
   - Update tag patterns if needed

4. **Initialize git** (if not already)
   ```bash
   git init
   git add .
   git commit -m "Initial commit from lib template"
   ```

## Key Configuration Files

### deny.toml - Security Auditing
```bash
# Run security audit
cargo deny check
```

### cliff.toml - Changelog Generation
```bash
# Generate changelog
git cliff -o CHANGELOG.md
```

### Pre-commit Hooks
```bash
# Run hooks manually
pre-commit run --all-files
```

## Common Workflows

### Development Cycle
```bash
# After making changes
cargo fmt           # Format code
cargo clippy        # Lint
cargo test          # Run tests
pre-commit run      # Run all hooks
```

### Release Preparation
```bash
# Update changelog
git cliff -o CHANGELOG.md

# Security check
cargo deny check

# Publish dry-run
cargo publish --dry-run
```

## Fallback: Standard Project Creation

If the template is unavailable or cargo-generate is not desired:

```bash
# Standard library creation
cargo new my_lib --lib

# Then manually add tooling:
# - Copy deny.toml, cliff.toml from template examples
# - Add pre-commit config
# - Set up GitHub Actions manually
```

## Anti-Patterns

```bash
# ❌ Forgetting to activate CI/CD
# _github directory won't work - GitHub ignores underscore prefix

# ❌ Skipping pre-commit install
# Hooks won't run automatically before commits

# ❌ Not reviewing workflow branch settings
# Workflows may target wrong branch (default: main)

# ✅ Always complete post-generation checklist
```

## See Also

- `reference.md` - Complete template documentation and configuration details
- `../simple-crate/` - General single-crate project structure patterns
- `../core/` - Core Rust development standards
- `../testing/` - Testing standards for library projects
