# Agent Resources

Reusable Claude Code resources for Drupal 10/11 development with DDEV and Docker-based local environments.

## Installation

Install any resource with a single command:

```bash
# Skills
uvx add-skill madsnorgaard/drupal-expert
uvx add-skill madsnorgaard/drupal-security
uvx add-skill madsnorgaard/drupal-migration
uvx add-skill madsnorgaard/ddev-expert
uvx add-skill madsnorgaard/docker-local

# Agents
uvx add-agent madsnorgaard/drupal-reviewer

# Commands
uvx add-command madsnorgaard/drush-check
uvx add-command madsnorgaard/module-scaffold
uvx add-command madsnorgaard/config-export
uvx add-command madsnorgaard/security-audit
uvx add-command madsnorgaard/performance-check
```

## Philosophy

**Research before building.** These resources emphasize checking drupal.org for existing contrib modules before writing custom code. Maintainable Drupal sites minimize custom code.

## Available Resources

### Skills

| Skill | Description |
|-------|-------------|
| `drupal-expert` | Drupal 10/11 development - modules, themes, services, hooks, D10/D11 compatibility |
| `drupal-security` | Security expertise - auto-warns about XSS, SQL injection, access bypass while coding |
| `drupal-migration` | Migration expertise - D7-to-D10, CSV imports, custom source/process plugins |
| `ddev-expert` | DDEV local development - commands, Xdebug, custom services, performance tuning |
| `docker-local` | Custom Docker Compose patterns for non-DDEV projects |

### Agents

| Agent | Description |
|-------|-------------|
| `drupal-reviewer` | Code review for Drupal - security, standards, performance, DI compliance |

### Commands

| Command | Usage | Description |
|---------|-------|-------------|
| `/drush-check` | `/drush-check` | Run health checks on a Drupal site |
| `/module-scaffold` | `/module-scaffold [name]` | Generate a new module with best-practice structure |
| `/config-export` | `/config-export` | Export Drupal configuration with review workflow |
| `/security-audit` | `/security-audit [path]` | Audit site for security vulnerabilities |
| `/performance-check` | `/performance-check [path]` | Analyze caching, queries, and optimization opportunities |

## Usage Examples

After installing the `drupal-expert` skill, Claude automatically applies Drupal best practices when you work on Drupal code - dependency injection, proper hooks, cache metadata, and more.

After installing the `drupal-reviewer` agent, Claude uses it to review your code for security issues, coding standards violations, and performance problems.

Commands are invoked with a slash:
```
/module-scaffold my_custom_module
/security-audit modules/custom/
```

## Target Environment

These resources assume:
- Drupal 10.3+ or Drupal 11
- PHP 8.2+
- Composer-based project structure
- Configuration management (config sync directory)
- Either DDEV or custom Docker Compose for local development

## Coding Standards

All generated code follows:
- Drupal coding standards (phpcs with drupal/coder)
- PSR-4 autoloading
- Dependency injection (no static `\Drupal::service()` calls)
- PHP 8.2+ features (constructor property promotion, typed properties)
- PHP attributes for plugins (Drupal 11 style)

## Contributing

Add new resources by creating files in:

- `.claude/skills/<skill-name>/SKILL.md` - For skills
- `.claude/agents/<agent-name>.md` - For agents
- `.claude/commands/<command-name>.md` - For slash commands

Push to GitHub and they're immediately available via `uvx add-skill/add-agent/add-command`.

## License

MIT
