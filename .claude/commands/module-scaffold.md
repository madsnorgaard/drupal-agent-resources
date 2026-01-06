---
name: module-scaffold
description: Generate a new Drupal module with best-practice structure
---

# Drupal Module Scaffolding

Create a new Drupal module following best practices.

## Step 1: Research First

**Before creating a custom module, ask the developer:**

1. "What functionality do you need?"
2. "Have you searched drupal.org for existing modules that provide this?"
3. "Would you like me to search for contrib alternatives first?"

**If developer hasn't researched:**
- Offer to search drupal.org/project/project_module
- Look for modules with: security coverage, active maintenance, D10/11 compatibility
- Only proceed with custom module if no suitable contrib exists

## Step 2: Gather Requirements

Ask the developer for:

1. **Module machine name** (e.g., `my_module`)
   - Must be lowercase, underscores only, start with letter
2. **Module human name** (e.g., "My Module")
3. **Module description** (one sentence)
4. **What the module needs:**
   - [ ] Custom service?
   - [ ] Custom block plugin?
   - [ ] Custom form (config or regular)?
   - [ ] Custom controller/page?
   - [ ] Custom permissions?
   - [ ] Event subscriber?
   - [ ] Custom entity? (complex - confirm needed)

## Step 3: Generate Structure

### Always Create

**my_module.info.yml**
```yaml
name: 'Module Name'
type: module
description: 'Module description.'
core_version_requirement: ^10 || ^11
package: Custom
```

**my_module.module**
```php
<?php

/**
 * @file
 * Primary module hooks for My Module.
 */

declare(strict_types=1);
```

### If Service Needed

**my_module.services.yml**
```yaml
services:
  my_module.example:
    class: Drupal\my_module\Service\ExampleService
    arguments: ['@entity_type.manager', '@current_user']
```

**src/Service/ExampleService.php**
```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Session\AccountProxyInterface;

/**
 * Example service with dependency injection.
 */
final class ExampleService {

  public function __construct(
    protected EntityTypeManagerInterface $entityTypeManager,
    protected AccountProxyInterface $currentUser,
  ) {}

  /**
   * Example method.
   */
  public function doSomething(): void {
    // Implementation here.
  }

}
```

### If Block Plugin Needed

**src/Plugin/Block/ExampleBlock.php**
```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Plugin\Block;

use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Provides an example block.
 */
#[Block(
  id: 'my_module_example',
  admin_label: new TranslatableMarkup('Example Block'),
  category: new TranslatableMarkup('Custom'),
)]
final class ExampleBlock extends BlockBase implements ContainerFactoryPluginInterface {

  public static function create(
    ContainerInterface $container,
    array $configuration,
    $plugin_id,
    $plugin_definition,
  ): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
    );
  }

  /**
   * {@inheritdoc}
   */
  public function build(): array {
    return [
      '#markup' => $this->t('Hello world!'),
      '#cache' => [
        'contexts' => ['user'],
        'tags' => [],
        'max-age' => 3600,
      ],
    ];
  }

}
```

### If Config Form Needed

**my_module.routing.yml**
```yaml
my_module.settings:
  path: '/admin/config/my-module'
  defaults:
    _form: '\Drupal\my_module\Form\SettingsForm'
    _title: 'My Module Settings'
  requirements:
    _permission: 'administer site configuration'
```

**my_module.links.menu.yml**
```yaml
my_module.settings:
  title: 'My Module'
  description: 'Configure My Module settings.'
  route_name: my_module.settings
  parent: system.admin_config
  weight: 100
```

**config/install/my_module.settings.yml**
```yaml
enabled: true
limit: 10
```

**config/schema/my_module.schema.yml**
```yaml
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

**src/Form/SettingsForm.php**
```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Form;

use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Configuration form for My Module.
 */
final class SettingsForm extends ConfigFormBase {

  /**
   * {@inheritdoc}
   */
  public function getFormId(): string {
    return 'my_module_settings';
  }

  /**
   * {@inheritdoc}
   */
  protected function getEditableConfigNames(): array {
    return ['my_module.settings'];
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state): array {
    $config = $this->config('my_module.settings');

    $form['enabled'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('Enabled'),
      '#default_value' => $config->get('enabled'),
    ];

    $form['limit'] = [
      '#type' => 'number',
      '#title' => $this->t('Limit'),
      '#default_value' => $config->get('limit'),
      '#min' => 1,
      '#max' => 100,
    ];

    return parent::buildForm($form, $form_state);
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->config('my_module.settings')
      ->set('enabled', $form_state->getValue('enabled'))
      ->set('limit', $form_state->getValue('limit'))
      ->save();

    parent::submitForm($form, $form_state);
  }

}
```

### If Permissions Needed

**my_module.permissions.yml**
```yaml
administer my_module:
  title: 'Administer My Module'
  description: 'Configure My Module settings.'
  restrict access: true
```

### Always Create Test Structure

**tests/src/Unit/.gitkeep**
**tests/src/Kernel/.gitkeep**
**tests/src/Functional/.gitkeep**

**Example Kernel Test (if service exists):**
```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\my_module\Kernel;

use Drupal\KernelTests\KernelTestBase;
use Drupal\my_module\Service\ExampleService;

/**
 * Tests the ExampleService.
 *
 * @group my_module
 */
final class ExampleServiceTest extends KernelTestBase {

  protected static $modules = ['my_module'];

  /**
   * Tests that the service can be instantiated.
   */
  public function testServiceExists(): void {
    $service = $this->container->get('my_module.example');
    $this->assertInstanceOf(ExampleService::class, $service);
  }

}
```

## Step 4: Post-Generation

After generating the module:

```bash
# Enable the module
drush en my_module

# Clear cache
drush cr

# Verify module is enabled
drush pm:list --filter=my_module

# Run tests (if created)
./vendor/bin/phpunit modules/custom/my_module
```

## Final Structure

```
my_module/
├── my_module.info.yml
├── my_module.module
├── my_module.services.yml
├── my_module.routing.yml
├── my_module.permissions.yml
├── my_module.links.menu.yml
├── config/
│   ├── install/
│   │   └── my_module.settings.yml
│   └── schema/
│       └── my_module.schema.yml
├── src/
│   ├── Controller/
│   ├── Form/
│   │   └── SettingsForm.php
│   ├── Plugin/
│   │   └── Block/
│   │       └── ExampleBlock.php
│   └── Service/
│       └── ExampleService.php
└── tests/
    └── src/
        ├── Unit/
        ├── Kernel/
        │   └── ExampleServiceTest.php
        └── Functional/
```

## Reminders

- Config schema is **required** for all custom configuration
- All classes use **dependency injection** (no `\Drupal::service()`)
- Test structure created from the start
- Use PHP 8.2+ features (constructor promotion, typed properties)
- Use PHP attributes for plugins (Drupal 11 style)
