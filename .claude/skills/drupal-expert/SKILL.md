---
name: drupal-expert
description: Drupal 10/11 development expertise. Use when working with Drupal modules, themes, hooks, services, configuration, or migrations. Triggers on mentions of Drupal, Drush, Twig, modules, themes, or Drupal API.
---

# Drupal Development Expert

You are an expert Drupal developer with deep knowledge of Drupal 10 and 11.

## Research-First Philosophy

**CRITICAL: Before writing ANY custom code, ALWAYS research existing solutions first.**

When a developer asks you to implement functionality:

1. **Ask the developer**: "Have you checked drupal.org for existing contrib modules that solve this?"
2. **Offer to research**: "I can help search for existing solutions before we build custom code."
3. **Only proceed with custom code** after confirming no suitable contrib module exists.

### How to Research Contrib Modules

Search on [drupal.org/project/project_module](https://www.drupal.org/project/project_module):

**Evaluate module health by checking:**
- Drupal 10/11 compatibility
- Security coverage (green shield icon)
- Last commit date (active maintenance?)
- Number of sites using it
- Issue queue responsiveness
- Whether it's covered by Drupal's security team

**Ask these questions:**
- Is there a well-maintained contrib module for this?
- Can an existing module be extended rather than building from scratch?
- Is there a Drupal Recipe (10.3+) that bundles this functionality?
- Would a patch to an existing module be better than custom code?

## Core Principles

### 1. Follow Drupal Coding Standards
- PSR-4 autoloading for all classes in `src/`
- Use PHPCS with Drupal/DrupalPractice standards
- Proper docblock comments on all functions and classes
- Use `t()` for all user-facing strings with proper placeholders:
  - `@variable` - sanitized text
  - `%variable` - sanitized and emphasized
  - `:variable` - URL (sanitized)

### 2. Use Dependency Injection
- **Never use** `\Drupal::service()` in classes - inject via constructor
- Define services in `*.services.yml`
- Use `ContainerInjectionInterface` for forms and controllers
- Use `ContainerFactoryPluginInterface` for plugins

```php
// WRONG - static service calls
class MyController {
  public function content() {
    $user = \Drupal::currentUser();
  }
}

// CORRECT - dependency injection
class MyController implements ContainerInjectionInterface {
  public function __construct(
    protected AccountProxyInterface $currentUser,
  ) {}

  public static function create(ContainerInterface $container) {
    return new static(
      $container->get('current_user'),
    );
  }
}
```

### 3. Hooks vs Event Subscribers

Both are valid in modern Drupal. Choose based on context:

**Use OOP Hooks when:**
- Altering Drupal core/contrib behavior
- Following core conventions
- Hook order (module weight) matters

**Use Event Subscribers when:**
- Integrating with third-party libraries (PSR-14)
- Building features that bundle multiple customizations
- Working with Commerce or similar event-heavy modules

```php
// OOP Hook (Drupal 11+)
#[Hook('form_alter')]
public function formAlter(&$form, FormStateInterface $form_state, $form_id): void {
  // ...
}

// Event Subscriber
public static function getSubscribedEvents() {
  return [
    KernelEvents::REQUEST => ['onRequest', 100],
  ];
}
```

### 4. Security First
- Never trust user input - always sanitize
- Use parameterized database queries (never concatenate)
- Check access permissions properly
- Use `#markup` with `Xss::filterAdmin()` or `#plain_text`
- Review OWASP top 10 for Drupal-specific risks

## Testing Requirements

**Tests are not optional for production code.**

### Test Types (Choose Appropriately)

| Type | Base Class | Use When |
|------|------------|----------|
| Unit | `UnitTestCase` | Testing isolated logic, no Drupal dependencies |
| Kernel | `KernelTestBase` | Testing services, entities, with minimal Drupal |
| Functional | `BrowserTestBase` | Testing user workflows, page interactions |
| FunctionalJS | `WebDriverTestBase` | Testing JavaScript/AJAX functionality |

### Test File Location
```
my_module/
└── tests/
    └── src/
        ├── Unit/           # Fast, isolated tests
        ├── Kernel/         # Service/entity tests
        └── Functional/     # Full browser tests
```

### When to Write Each Type

- **Unit tests**: Pure PHP logic, utility functions, data transformations
- **Kernel tests**: Services, database queries, entity operations, hooks
- **Functional tests**: Forms, controllers, access control, user flows
- **FunctionalJS tests**: Dynamic forms, AJAX, JavaScript behaviors

### Running Tests
```bash
# Run specific test
./vendor/bin/phpunit modules/custom/my_module/tests/src/Unit/MyTest.php

# Run all module tests
./vendor/bin/phpunit modules/custom/my_module

# Run with coverage
./vendor/bin/phpunit --coverage-html coverage modules/custom/my_module
```

## Module Structure

```
my_module/
├── my_module.info.yml
├── my_module.module           # Hooks only (keep thin)
├── my_module.services.yml     # Service definitions
├── my_module.routing.yml      # Routes
├── my_module.permissions.yml  # Permissions
├── my_module.libraries.yml    # CSS/JS libraries
├── config/
│   ├── install/               # Default config
│   ├── optional/              # Optional config (dependencies)
│   └── schema/                # Config schema (REQUIRED for custom config)
├── src/
│   ├── Controller/
│   ├── Form/
│   ├── Plugin/
│   │   ├── Block/
│   │   └── Field/
│   ├── Service/
│   ├── EventSubscriber/
│   └── Hook/                  # OOP hooks (Drupal 11+)
├── templates/                 # Twig templates
└── tests/
    └── src/
        ├── Unit/
        ├── Kernel/
        └── Functional/
```

## Common Patterns

### Service Definition
```yaml
services:
  my_module.my_service:
    class: Drupal\my_module\Service\MyService
    arguments: ['@entity_type.manager', '@current_user', '@logger.factory']
```

### Route with Permission
```yaml
my_module.page:
  path: '/my-page'
  defaults:
    _controller: '\Drupal\my_module\Controller\MyController::content'
    _title: 'My Page'
  requirements:
    _permission: 'access content'
```

### Plugin (Block Example)
```php
#[Block(
  id: "my_block",
  admin_label: new TranslatableMarkup("My Block"),
)]
class MyBlock extends BlockBase implements ContainerFactoryPluginInterface {
  // Always use ContainerFactoryPluginInterface for DI in plugins
}
```

### Config Schema (Required!)
```yaml
# config/schema/my_module.schema.yml
my_module.settings:
  type: config_object
  label: 'My Module settings'
  mapping:
    enabled:
      type: boolean
      label: 'Enabled'
    limit:
      type: integer
      label: 'Limit'
```

## Database Queries

Always use the database abstraction layer:

```php
// CORRECT - parameterized query
$query = $this->database->select('node', 'n');
$query->fields('n', ['nid', 'title']);
$query->condition('n.type', $type);
$query->range(0, 10);
$results = $query->execute();

// NEVER do this - SQL injection risk
$result = $this->database->query("SELECT * FROM node WHERE type = '$type'");
```

## Cache Metadata

**Always add cache metadata to render arrays:**

```php
$build['content'] = [
  '#markup' => $content,
  '#cache' => [
    'tags' => ['node_list', 'user:' . $uid],
    'contexts' => ['user.permissions', 'url.query_args'],
    'max-age' => 3600,
  ],
];
```

### Cache Tag Conventions
- `node:123` - specific node
- `node_list` - any node list
- `user:456` - specific user
- `config:my_module.settings` - configuration

## CLI-First Development Workflows

**Before writing custom code, use Drush generators to scaffold boilerplate code.**

Drush's code generation features follow Drupal best practices and coding standards, reducing errors and accelerating development. Always prefer CLI tools over manual file creation for standard Drupal structures.

### Content Types and Fields

**CRITICAL: Use CLI commands to create content types and fields instead of manual configuration or PHP code.**

#### Create Content Types

```bash
# Interactive mode - Drush prompts for all details
drush generate content-entity

# Create via PHP eval (for scripts/automation)
drush php:eval "
\$type = \Drupal\node\Entity\NodeType::create([
  'type' => 'article',
  'name' => 'Article',
  'description' => 'Articles with images and tags',
  'new_revision' => TRUE,
  'display_submitted' => TRUE,
  'preview_mode' => 1,
]);
\$type->save();
echo 'Content type created.';
"
```

#### Create Fields

```bash
# Interactive mode (recommended for first-time use)
drush field:create

# Non-interactive mode with all parameters
drush field:create node article \
  --field-name=field_subtitle \
  --field-label="Subtitle" \
  --field-type=string \
  --field-widget=string_textfield \
  --is-required=0 \
  --cardinality=1

# Create a reference field
drush field:create node article \
  --field-name=field_tags \
  --field-label="Tags" \
  --field-type=entity_reference \
  --field-widget=entity_reference_autocomplete \
  --cardinality=-1 \
  --target-type=taxonomy_term

# Create an image field
drush field:create node article \
  --field-name=field_image \
  --field-label="Image" \
  --field-type=image \
  --field-widget=image_image \
  --is-required=0 \
  --cardinality=1
```

**Common field types:**
- `string` - Plain text
- `string_long` - Long text (textarea)
- `text_long` - Formatted text
- `text_with_summary` - Body field with summary
- `integer` - Whole numbers
- `decimal` - Decimal numbers
- `boolean` - Checkbox
- `datetime` - Date/time
- `email` - Email address
- `link` - URL
- `image` - Image upload
- `file` - File upload
- `entity_reference` - Reference to other entities
- `list_string` - Select list
- `telephone` - Phone number

**Common field widgets:**
- `string_textfield` - Single line text
- `string_textarea` - Multi-line text
- `text_textarea` - Formatted text area
- `text_textarea_with_summary` - Body with summary
- `number` - Number input
- `checkbox` - Single checkbox
- `options_select` - Select dropdown
- `options_buttons` - Radio buttons/checkboxes
- `datetime_default` - Date picker
- `email_default` - Email input
- `link_default` - URL input
- `image_image` - Image upload
- `file_generic` - File upload
- `entity_reference_autocomplete` - Autocomplete reference

#### Manage Fields

```bash
# List all fields on a content type
drush field:info node article

# List available field types
drush field:types

# List available field widgets
drush field:widgets

# List available field formatters
drush field:formatters

# Delete a field
drush field:delete node.article.field_subtitle
```

### Generate Module Scaffolding

```bash
# Generate a complete module
drush generate module
# Prompts for: module name, description, package, dependencies

# Generate a controller
drush generate controller
# Prompts for: module, class name, route path, services to inject

# Generate a simple form
drush generate form-simple
# Creates form with submit/validation, route, and menu link

# Generate a config form
drush generate form-config
# Creates settings form with automatic config storage

# Generate a block plugin
drush generate plugin:block
# Creates block plugin with dependency injection support

# Generate a service
drush generate service
# Creates service class and services.yml entry

# Generate a hook implementation
drush generate hook
# Creates hook in .module file or OOP hook class (D11)

# Generate an event subscriber
drush generate event-subscriber
# Creates subscriber class and services.yml entry
```

### Generate Entity Types

```bash
# Generate a custom content entity
drush generate entity:content
# Creates entity class, storage, access control, views integration

# Generate a config entity
drush generate entity:configuration
# Creates config entity with list builder and forms
```

### Generate Common Patterns

```bash
# Generate a plugin (various types)
drush generate plugin:field:formatter
drush generate plugin:field:widget
drush generate plugin:field:type
drush generate plugin:block
drush generate plugin:condition
drush generate plugin:filter

# Generate a Drush command
drush generate drush:command-file

# Generate a test
drush generate test:unit
drush generate test:kernel
drush generate test:browser
```

### Create Test Content

**Use Devel Generate for test data instead of manual entry:**

```bash
# Generate 50 nodes
drush devel-generate:content 50 --bundles=article,page --kill

# Generate taxonomy terms
drush devel-generate:terms 100 tags --kill

# Generate users
drush devel-generate:users 20

# Generate media entities
drush devel-generate:media 30 --bundles=image,document
```

### Workflow Best Practices

**1. Always start with generators:**
```bash
# Create module structure first
drush generate module

# Then generate specific components
drush generate controller
drush generate form-config
drush generate service
```

**2. Use field:create for all field additions:**
```bash
# Never manually create field config files
# Use drush field:create instead
drush field:create node article --field-name=field_subtitle
```

**3. Export configuration after CLI changes:**
```bash
# After creating fields/content types via CLI
drush config:export -y
```

**4. Document your scaffolding in README:**
```markdown
## Regenerating Module Structure

This module was scaffolded with:
- drush generate module
- drush generate controller
- drush field:create node article --field-name=field_custom
```

### Avoiding Common Mistakes

**DON'T manually create:**
- Content type config files (`node.type.*.yml`)
- Field config files (`field.field.*.yml`, `field.storage.*.yml`)
- View mode config (`core.entity_view_display.*.yml`)
- Form mode config (`core.entity_form_display.*.yml`)

**DO use CLI commands:**
- `drush generate` for code scaffolding
- `drush field:create` for fields
- `drush php:eval` for content types
- `drush config:export` to capture changes

### Integration with DDEV/Docker

```bash
# When using DDEV
ddev drush generate module
ddev drush field:create node article

# When using Docker Compose
docker compose exec php drush generate module
docker compose exec php drush field:create node article

# When using DDEV with custom commands
ddev exec drush generate controller
```

## Essential Drush Commands

```bash
drush cr                    # Clear cache
drush cex -y                # Export config
drush cim -y                # Import config
drush updb -y               # Run updates
drush en module_name        # Enable module
drush pmu module_name       # Uninstall module
drush ws --severity=error   # Watch logs
drush php:eval "code"       # Run PHP

# Code generation (see CLI-First Development above)
drush generate              # List all generators
drush gen module            # Generate module (gen is alias)
drush field:create          # Create field (fc is alias)
drush entity:create         # Create entity content
```

## Twig Best Practices

- Variables are auto-escaped (no need for `|escape`)
- Use `{% trans %}` for translatable strings
- Use `attach_library` for CSS/JS, never inline
- Enable Twig debugging in development
- Use `{{ dump(variable) }}` for debugging

```twig
{# Correct - uses translation #}
{% trans %}Hello {{ name }}{% endtrans %}

{# Attach library #}
{{ attach_library('my_module/my-library') }}

{# Safe markup (already sanitized) #}
{{ content|raw }}
```

## Before You Code Checklist

1. [ ] Searched drupal.org for existing modules?
2. [ ] Checked if a Recipe exists (Drupal 10.3+)?
3. [ ] Reviewed similar contrib modules for patterns?
4. [ ] Confirmed no suitable solution exists?
5. [ ] Planned test coverage?
6. [ ] Defined config schema for any custom config?
7. [ ] Using dependency injection (no static calls)?

## Drupal 10 to 11 Compatibility

### Key Differences

| Feature | Drupal 10 | Drupal 11 |
|---------|-----------|-----------|
| PHP Version | 8.1+ | 8.3+ |
| Symfony | 6.x | 7.x |
| Hooks | Procedural or OOP | OOP preferred (attributes) |
| Annotations | Supported | Deprecated (use attributes) |
| jQuery | Included | Optional |

### Writing Compatible Code (D10.3+ and D11)

**Use PHP attributes for plugins** (works in D10.2+, required style for D11):

```php
// Modern style (D10.2+, required for D11)
#[Block(
  id: 'my_block',
  admin_label: new TranslatableMarkup('My Block'),
)]
class MyBlock extends BlockBase {}

// Legacy style (still works but discouraged)
/**
 * @Block(
 *   id = "my_block",
 *   admin_label = @Translation("My Block"),
 * )
 */
```

**Use OOP hooks** (D10.3+):

```php
// Modern OOP hooks (D10.3+)
// src/Hook/MyModuleHooks.php
namespace Drupal\my_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;

final class MyModuleHooks {

  #[Hook('form_alter')]
  public function formAlter(&$form, FormStateInterface $form_state, $form_id): void {
    // ...
  }

  #[Hook('node_presave')]
  public function nodePresave(NodeInterface $node): void {
    // ...
  }

}
```

Register hooks class in services.yml:
```yaml
services:
  Drupal\my_module\Hook\MyModuleHooks:
    autowire: true
```

**Procedural hooks still work** but should be in `.module` file only for backward compatibility.

### Deprecated APIs to Avoid

```php
// DEPRECATED - don't use
drupal_set_message()           // Use messenger service
format_date()                  // Use date.formatter service
entity_load()                  // Use entity_type.manager
db_select()                    // Use database service
drupal_render()                // Use renderer service
\Drupal::l()                   // Use Link::fromTextAndUrl()
```

### Check Deprecations

```bash
# Run deprecation checks
./vendor/bin/drupal-check modules/custom/

# Or with PHPStan
./vendor/bin/phpstan analyze modules/custom/ --level=5
```

### info.yml Compatibility

```yaml
# Support both D10 and D11
core_version_requirement: ^10.3 || ^11

# D11 only
core_version_requirement: ^11
```

### Recipes (D10.3+)

Drupal Recipes provide reusable configuration packages:

```bash
# Apply a recipe
php core/scripts/drupal recipe core/recipes/standard

# Community recipes
composer require drupal/recipe_name
php core/scripts/drupal recipe recipes/contrib/recipe_name
```

When to use Recipes vs Modules:
- **Recipes**: Configuration-only, site building, content types, views
- **Modules**: Custom PHP code, new functionality, APIs

### Testing Compatibility

```bash
# Test against both versions in CI
jobs:
  test-d10:
    env:
      DRUPAL_CORE: ^10.3
  test-d11:
    env:
      DRUPAL_CORE: ^11
```

### Migration Planning

Before upgrading D10 → D11:
1. Run `drupal-check` for deprecations
2. Update all contrib modules to D11-compatible versions
3. Convert annotations to attributes
4. Consider moving hooks to OOP style
5. Test thoroughly in staging environment

## Pre-Commit Checks

**CRITICAL: Always run these checks locally BEFORE committing or pushing code.**

CI pipeline failures are embarrassing and waste time. Catch issues locally first.

### Required: Coding Standards (PHPCS)

```bash
# Check for coding standard violations
./vendor/bin/phpcs -p --colors modules/custom/

# Auto-fix what can be fixed
./vendor/bin/phpcbf modules/custom/

# Check specific file
./vendor/bin/phpcs path/to/MyClass.php
```

**Common PHPCS errors to watch for:**
- Missing trailing commas in multi-line function declarations
- Nullable parameters without `?` type hint
- Missing docblocks
- Incorrect spacing/indentation

### DDEV Shortcut

```bash
# Run inside DDEV
ddev exec ./vendor/bin/phpcs -p modules/custom/
ddev exec ./vendor/bin/phpcbf modules/custom/
```

### Recommended: Full Pre-Commit Checklist

```bash
# 1. Coding standards
./vendor/bin/phpcs -p modules/custom/

# 2. Static analysis (if configured)
./vendor/bin/phpstan analyze modules/custom/

# 3. Deprecation checks
./vendor/bin/drupal-check modules/custom/

# 4. Run tests
./vendor/bin/phpunit modules/custom/my_module/tests/
```

### Git Pre-Commit Hook (Optional)

Create `.git/hooks/pre-commit`:
```bash
#!/bin/bash
./vendor/bin/phpcs --standard=Drupal,DrupalPractice modules/custom/ || exit 1
```

Make executable: `chmod +x .git/hooks/pre-commit`

### Installing PHPCS with Drupal Standards

```bash
composer require --dev drupal/coder
./vendor/bin/phpcs --config-set installed_paths vendor/drupal/coder/coder_sniffer
```

## Sources

- [Drupal Testing Types](https://www.drupal.org/docs/develop/automated-testing/types-of-tests)
- [Services and Dependency Injection](https://www.drupal.org/docs/drupal-apis/services-and-dependency-injection)
- [Hooks vs Events](https://www.specbee.com/blogs/hooks-vs-events-in-drupal-making-informed-choice)
- [PHPUnit in Drupal](https://www.drupal.org/docs/develop/automated-testing/phpunit-in-drupal)
- [Drupal 11 Readiness](https://www.drupal.org/docs/upgrading-drupal/how-to-prepare-your-drupal-7-or-8-site-for-drupal-9/deprecation-checking-and-correction-tools)
- [OOP Hooks](https://www.drupal.org/docs/develop/creating-modules/implementing-hooks-in-drupal-11)
- [Drupal Recipes](https://www.drupal.org/docs/extending-drupal/drupal-recipes)
- [Drush Code Generators](https://drupalize.me/tutorial/develop-drupal-modules-faster-drush-code-generators)
- [Drush Generate Command](https://www.drush.org/11.x/commands/generate/)
- [Drush field:create](https://www.drush.org/13.x/commands/field_create/)
- [Scaffold Custom Content Entity with Drush](https://drupalize.me/tutorial/scaffold-custom-content-entity-type-drush-generators)
