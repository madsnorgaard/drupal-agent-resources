# Versioning & Release Standards

This document defines the versioning strategy, release process, and quality standards for the drupal-agent-resources project.

## Versioning Scheme

This project follows [Semantic Versioning](https://semver.org/) adapted for agent resources:

```
MAJOR.MINOR.PATCH
```

- **MAJOR** — Breaking changes to resource structure, frontmatter schema, or activation behavior that require users to update their workflows
- **MINOR** — New resources (skills, commands, agents), significant enhancements to existing resources, or new Drupal/PHP version coverage
- **PATCH** — Bug fixes, documentation corrections, minor wording improvements, or small accuracy updates to existing resources

### Examples

| Change | Version Bump |
|--------|-------------|
| New skill added (e.g., `drupal-testing`) | Minor |
| New command added (e.g., `/drupal11-readiness`) | Minor |
| Frontmatter schema change requiring user action | Major |
| Fix incorrect Drupal API example in a skill | Patch |
| Add PHP 8.4 coverage to existing skill | Minor |
| Typo fix in documentation | Patch |
| Remove a deprecated resource | Major |

## Release Process

### 1. Planning

Each release should be tracked in [ROADMAP.md](ROADMAP.md) and associated with a GitHub milestone. Releases align with the phased roadmap:

- **Phase 1**: Drupal 11 & PHP 8.4 support
- **Phase 2**: Testing & quality assurance
- **Phase 3**: Performance & advanced features

### 2. Pre-Release Checklist

Before tagging a release, verify:

- [ ] All new resources have valid YAML frontmatter (`name`, `description` at minimum)
- [ ] Code examples use PHP 8.2+ syntax (constructor promotion, typed properties)
- [ ] Code examples use PHP attributes, not annotations
- [ ] No `\Drupal::service()` in class-based code examples (dependency injection required)
- [ ] CI validation workflow passes (`validate.yml`)
- [ ] CHANGELOG.md is updated with all changes
- [ ] README.md resource tables are current
- [ ] Cross-references between related resources are accurate

### 3. Tagging & Release

```bash
# Tag the release
git tag -a v1.0.0 -m "v1.0.0 - Initial stable release"
git push origin v1.0.0

# Create GitHub Release with notes from CHANGELOG.md
```

Each GitHub Release should include:
- Summary of changes (from CHANGELOG.md)
- List of new resources added
- Any breaking changes with migration steps
- Links to relevant Drupal.org or PHP documentation

### 4. Post-Release

- Announce in relevant Drupal community channels
- Update any external documentation referencing specific versions
- Monitor issues for regressions

## What Constitutes a Release

### Major Release (e.g., 1.0.0 → 2.0.0)

A major release is warranted when:
- Resource frontmatter schema changes in a backward-incompatible way
- Resources are removed or renamed (breaking `agr add` commands)
- Minimum supported Drupal version changes (e.g., dropping Drupal 10 support)
- Fundamental changes to how resources are structured or activated

### Minor Release (e.g., 1.0.0 → 1.1.0)

A minor release is warranted when:
- One or more new skills, commands, or agents are added
- Significant new coverage is added to existing resources (e.g., Drupal 11 upgrade patterns)
- New Drupal or PHP version support is introduced
- A new phase from the roadmap is started

### Patch Release (e.g., 1.0.0 → 1.0.1)

A patch release is warranted when:
- Incorrect code examples are fixed
- Documentation typos or clarifications are made
- Minor adjustments to resource behavior without new features
- Links to external resources are updated

## Quality Standards for Releases

All releases must meet the standards defined in CLAUDE.md and the following additional criteria:

### Code Examples
- PHP 8.2+ syntax with constructor property promotion and typed properties
- PHP attributes for all plugin examples (`#[Block(...)]`, not `@Block`)
- Dependency injection in all class-based examples
- Config schema for any custom configuration examples
- Proper error handling and logging patterns

### Documentation
- Clear activation criteria and usage instructions
- Real-world examples grounded in Drupal best practices
- Troubleshooting guidance where applicable
- Cross-references to related resources within the project

### Validation
- YAML frontmatter validates via CI (`validate.yml`)
- Code examples are syntactically correct
- Installation tested via `agr add madsnorgaard/<resource>`

## Branching Strategy

- **`main`** — Stable, release-ready code. All releases are tagged from `main`.
- **`feature/<name>`** — Feature branches for new resources or significant changes.
- **Pull requests** — All changes go through PR review before merging to `main`.

## Changelog Format

See [CHANGELOG.md](CHANGELOG.md) for the full history. The changelog follows [Keep a Changelog](https://keepachangelog.com/) conventions:

- **Added** — New resources or features
- **Changed** — Modifications to existing resources
- **Deprecated** — Resources or patterns that will be removed in a future release
- **Removed** — Resources or patterns that have been removed
- **Fixed** — Bug fixes or corrections
- **Security** — Security-related changes

## Aligning with Drupal's Release Cycle

Drupal is a mature CMS with a well-established release cadence. This project aims to stay aligned:

- **Drupal minor releases** (e.g., 10.3, 11.1) — Evaluate whether resources need updates for new APIs or deprecations
- **Drupal major releases** (e.g., 11.0) — Plan a corresponding minor or major release with upgrade resources
- **PHP releases** (e.g., 8.4) — Add compatibility guidance and update code examples
- **Security advisories** — Update security-related resources promptly when Drupal Security Advisories are published
