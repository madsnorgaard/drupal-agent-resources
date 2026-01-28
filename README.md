# Agent Resources

Reusable Claude Code resources for Drupal 10/11 development with DDEV and Docker-based local environments.

Built on the [agent-resources](https://github.com/kasperjunge/agent-resources) package manager by Kasper Junge.

## Installation

Install resources with a single command using `agr` (auto-detects resource type):

```bash
# Skills
agr add madsnorgaard/drupal-expert
agr add madsnorgaard/drupal-security
agr add madsnorgaard/drupal-migration
agr add madsnorgaard/ddev-expert
agr add madsnorgaard/docker-local

# Agents
agr add madsnorgaard/drupal-reviewer

# Commands
agr add madsnorgaard/drush-check
agr add madsnorgaard/module-scaffold
agr add madsnorgaard/config-export
agr add madsnorgaard/security-audit
agr add madsnorgaard/performance-check
```

### Quick Start

```bash
# Install agr via uv
uv tool install agent-resources

# Add a resource (type auto-detected)
agr add madsnorgaard/drupal-expert

# Or try temporarily without installing
agrx madsnorgaard/drupal-expert

# Remove a resource
agr remove drupal-expert

# List installed resources
agr list
```

## Linux/WSL Installation

If you're on Linux or WSL and don't have pip installed, follow these steps first:

### Prerequisites

For Ubuntu/Debian-based systems (including WSL):

```bash
# Update package list
sudo apt update

# Install Python 3 and pip
sudo apt install -y python3 python3-pip python3-venv

# Verify installation
python3 --version
pip3 --version
```

For other distributions:
- **Fedora/RHEL**: `sudo dnf install python3 python3-pip`
- **Arch**: `sudo pacman -S python python-pip`

### Install uv (Universal Package Manager)

```bash
# Option 1: Using pip (recommended for WSL/Linux)
pip3 install --user uv

# Option 2: Using curl
curl -LsSf https://astral.sh/uv/install.sh | sh

# Verify installation
uv --version
```

If `uv` command is not found, add to PATH:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Install agent-resources

```bash
# Install the agr CLI tool
uv tool install agent-resources

# Verify installation
agr --version

# If agr command not found, ensure .local/bin is in PATH:
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### WSL-Specific Notes

- **Path translation**: File paths in WSL use `/mnt/c/` for Windows drives
- **Line endings**: Ensure git is configured for Unix line endings:
  ```bash
  git config --global core.autocrlf input
  ```
- **Performance**: For best performance, work within the WSL filesystem (`~/projects/`) rather than `/mnt/c/`

### Troubleshooting

**"Command not found: agr"**
- Ensure `~/.local/bin` is in your PATH
- Check installation: `uv tool list`
- Reinstall: `uv tool uninstall agent-resources && uv tool install agent-resources`

**"Permission denied" errors**
- Don't use sudo with pip/uv user installs
- Check file permissions: `ls -la ~/.local/bin/agr`

**Resources not loading in Claude Code**
- Verify installation: `agr list`
- Check resource location: `ls -la ~/.claude/`
- Restart Claude Code after installing new resources

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

Push to GitHub and they're immediately available via `agr add`.

## License

MIT
