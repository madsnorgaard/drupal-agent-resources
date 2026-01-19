# Agent Resources - Development Guidelines

This repository contains reusable Claude Code resources for Drupal development. When working in this repo, follow these guidelines.

## Repository Structure

```
.claude/
├── skills/<name>/SKILL.md    # Expertise areas (auto-activated)
├── commands/<name>.md         # Slash commands (/command-name)
└── agents/<name>.md           # Specialized subagents
```

## Creating Resources

### Skills
- File: `.claude/skills/<skill-name>/SKILL.md`
- Purpose: Domain expertise that activates automatically based on context
- Frontmatter: `name`, `description`

### Commands
- File: `.claude/commands/<command-name>.md`
- Purpose: User-invoked workflows via `/command-name`
- Frontmatter: `name`, `description`, optional `arguments`

### Agents
- File: `.claude/agents/<agent-name>.md`
- Purpose: Specialized subagents for delegation
- Frontmatter: `name`, `description`, `tools`, optional `model`

## Code Style Requirements

All Drupal code examples must follow:
- **PHP 8.2+ features**: Constructor property promotion, typed properties
- **PHP Attributes**: Use `#[Block(...)]` style, not `@Block` annotations
- **Dependency injection**: No `\Drupal::service()` in classes
- **Config schema**: Required for all custom configuration

## Philosophy

**Research before building.** All skills and commands should encourage checking drupal.org for existing contrib modules before writing custom code.

## Testing Changes

After modifying resources:
1. Validate YAML frontmatter syntax
2. Check code examples compile/run
3. Test installation: `agr add madsnorgaard/<resource>`

## Naming Conventions

- Skill names: `domain-expert` pattern (e.g., `drupal-expert`)
- Command names: `action-noun` pattern (e.g., `module-scaffold`)
- Agent names: `domain-reviewer` pattern (e.g., `drupal-reviewer`)

## Installation Commands

Resources are installed via Kasper Junge's [agent-resources](https://github.com/kasperjunge/agent-resources) CLI:

```bash
# Install agr
uv tool install agent-resources

# Add resources (type auto-detected from repo)
agr add <username>/<resource-name>

# Try temporarily without installing
agrx <username>/<resource-name>

# Remove a resource
agr remove <resource-name>

# List installed resources
agr list

# Use --type flag if multiple resources share a name
agr add <username>/<resource-name> --type skill
```
