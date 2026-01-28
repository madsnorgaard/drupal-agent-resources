# Agent Resources Roadmap

This document outlines planned improvements and new resources for the agent-resources repository, with focus on Drupal 11, PHP 8.4, and enhanced developer workflows.

## Current State Analysis

### Existing Resources

**Skills (5):**
- `drupal-expert` - Comprehensive D10/11 development
- `drupal-security` - Security patterns and vulnerability prevention
- `drupal-migration` - Migrate API and D7-to-D11 migrations
- `ddev-expert` - DDEV local development
- `docker-local` - Custom Docker Compose setups

**Commands (5):**
- `/drush-check` - Health checks
- `/module-scaffold` - Module generation
- `/config-export` - Configuration export workflow
- `/security-audit` - Security vulnerability scanning
- `/performance-check` - Performance analysis

**Agents (1):**
- `drupal-reviewer` - Code review for Drupal

### Identified Gaps

1. **PHP 8.4 Compatibility**: No dedicated resource for PHP 8.4 features, deprecations, and migration
2. **Drupal 11 Upgrade Path**: Limited D10→D11 upgrade guidance and readiness checking
3. **Deprecation Handling**: No automated deprecation detection and fixing workflow
4. **Testing**: No dedicated testing expertise (Unit, Kernel, Functional, E2E)
5. **Security Depth**: Could expand with specialized security agent and automated scanning
6. **Performance**: Basic coverage in command, but no dedicated performance skill

---

## Planned Resources

### Phase 1: Critical Drupal 11 & PHP 8.4 Support (High Priority)

#### 1.1 New Skill: `drupal-11-upgrade`

**File**: `.claude/skills/drupal-11-upgrade/SKILL.md`

**Purpose**: Specialized expertise for Drupal 10→11 upgrades and D11-specific development patterns

**Key Coverage**:
- PHP 8.3+ requirements (D11 minimum) and PHP 8.4 compatibility
- Symfony 7.x integration changes
- OOP Hooks migration (procedural hooks → attribute-based hooks)
- Annotation → Attribute conversion for all plugins
- jQuery removal strategies and vanilla JS alternatives
- CKEditor 4 → CKEditor 5 migration
- Automated upgrade tools (drupal-check, PHPStan, Rector)
- Drupal StarShot and Recipes ecosystem
- Backwards compatibility strategies during transition

**Example Patterns**:
```php
// Old D10 annotation style
/**
 * @Block(
 *   id = "example_block",
 *   admin_label = @Translation("Example block")
 * )
 */

// New D11 attribute style
#[Block(
  id: 'example_block',
  admin_label: new TranslatableMarkup('Example block')
)]
```

**Integration Points**:
- Extends `drupal-expert` with D11-specific guidance
- Works with `/drupal11-readiness` command

---

#### 1.2 New Skill: `php84-compat`

**File**: `.claude/skills/php84-compat/SKILL.md`

**Purpose**: PHP 8.4-specific features, deprecations, and Drupal compatibility guidance

**Key Coverage**:
- **New PHP 8.4 Features**:
  - Property hooks (get/set)
  - Asymmetric visibility (public get, private set)
  - New DOM HTML5 parsing API
  - PDO driver-specific subclasses
  - Array functions improvements
  - JIT compilation enhancements

- **PHP 8.4 Deprecations**:
  - Implicitly nullable parameters
  - GMP resource to object migration
  - hash_pbkdf2() parameter changes
  - FILTER_SANITIZE_* deprecations

- **Drupal Compatibility**:
  - Which Drupal versions support PHP 8.4
  - Common breaking changes in Drupal context
  - Migration strategies from PHP 8.2/8.3
  - Performance improvements for Drupal on PHP 8.4

**Example Patterns**:
```php
// PHP 8.4 property hooks
class ConfigEntity {
  private string $_name;

  public string $name {
    get => $this->_name;
    set => $this->_name = strtolower($value);
  }
}

// Asymmetric visibility (great for value objects)
class EntityId {
  public private(set) string $id;

  public function __construct(string $id) {
    $this->id = $id;  // Can set internally
  }
  // External code can read but not modify $id
}
```

**Integration Points**:
- Complements `drupal-11-upgrade` (D11 requires PHP 8.3+)
- Used by `/php-upgrade-check` command

---

#### 1.3 New Command: `/drupal11-readiness`

**File**: `.claude/commands/drupal11-readiness.md`

**Purpose**: Comprehensive Drupal 11 upgrade readiness assessment

**Workflow**:
1. **Check Current Version**
   ```bash
   drush status --fields=drupal-version,php-version
   ```

2. **Scan for Deprecations**
   ```bash
   # Install if needed
   composer require --dev mglaman/drupal-check

   # Scan custom modules
   vendor/bin/drupal-check modules/custom
   vendor/bin/drupal-check themes/custom
   ```

3. **Contrib Module Compatibility**
   ```bash
   # Check drupal.org project pages for D11 compatibility
   # Or use Upgrade Status module
   drush en upgrade_status
   drush upgrade_status:analyze
   ```

4. **Check for Annotations** (should be Attributes in D11)
   ```bash
   # Search for old annotation style
   grep -r "@Block\|@Plugin" modules/custom --include="*.php"
   ```

5. **Symfony 7 Compatibility Check**
   ```bash
   composer why symfony/symfony
   # Verify all Symfony components are 7.x compatible
   ```

6. **jQuery Dependency Scan**
   ```bash
   # Search for jQuery usage in custom code
   grep -r "jQuery\|Drupal.behaviors" modules/custom --include="*.js"
   ```

7. **CKEditor Version Check**
   ```bash
   drush pm:list --filter=ckeditor
   # Ensure using ckeditor5, not deprecated ckeditor
   ```

**Report Format**:
```
# Drupal 11 Readiness Report

## Summary
- Current Drupal Version: 10.3.1
- Current PHP Version: 8.2.14
- Readiness Score: 65% (NEEDS WORK)

## Critical Issues (Blockers)
- [ ] 12 contrib modules not yet D11 compatible
- [ ] 8 annotation-based plugins need attribute conversion
- [ ] Custom theme still uses jQuery for AJAX

## Warnings
- [ ] PHP 8.2 supported but PHP 8.3+ recommended for D11
- [ ] 3 deprecated function calls in custom modules
- [ ] CKEditor 4 fields need migration path planning

## Recommendations
1. Update to PHP 8.3 or 8.4
2. Run `/deprecation-fix` to auto-fix common deprecations
3. Convert plugin annotations to attributes
4. Plan jQuery removal strategy
5. Test contrib modules on D11 in staging environment

## Next Steps
1. Run: /deprecation-fix modules/custom
2. Run: /php-upgrade-check 8.3
3. Review contrib module alternatives for incompatible modules
```

**Integration Points**:
- Uses `drupal-11-upgrade` skill knowledge
- Suggests follow-up commands (`/deprecation-fix`, `/php-upgrade-check`)

---

#### 1.4 New Command: `/php-upgrade-check`

**File**: `.claude/commands/php-upgrade-check.md`

**Usage**: `/php-upgrade-check [target_version]`

**Purpose**: Analyze codebase for PHP version compatibility and upgrade readiness

**Workflow**:
1. **Detect Current PHP Version**
   ```bash
   php -v
   drush status --field=php-version
   ```

2. **Install Analysis Tools**
   ```bash
   composer require --dev phpstan/phpstan
   composer require --dev mglaman/drupal-check
   ```

3. **Scan for Deprecated Functions**
   ```bash
   # PHPStan with Drupal extension
   vendor/bin/phpstan analyse modules/custom themes/custom \
     --level=5 \
     --memory-limit=1G

   # Drupal-specific deprecations
   vendor/bin/drupal-check modules/custom
   ```

4. **Check Composer Dependencies**
   ```bash
   # Check if dependencies support target PHP version
   composer require php:^8.4 --dry-run

   # Update platform config to test
   composer config platform.php 8.4.0
   composer update --dry-run
   ```

5. **Common Breaking Changes Analysis**
   ```bash
   # Search for known PHP 8.4 breaking changes
   grep -rn "each\|create_function\|money_format" modules/custom
   ```

**Report Format**:
```
# PHP 8.4 Upgrade Readiness Report

## Current Environment
- PHP Version: 8.2.14
- Target Version: 8.4
- Drupal Version: 10.3.1

## Compatibility Status: READY (with minor fixes)

## Issues Found

### Errors (Must Fix)
1. **Deprecated: Implicitly nullable parameters** (3 occurrences)
   - modules/custom/example/src/Service.php:45
   - Fix: Add explicit `?` type or `null` default value

### Warnings (Should Fix)
1. **Usage of deprecated hash_pbkdf2() signature** (1 occurrence)
   - modules/custom/auth/src/Encryption.php:23
   - Fix: Update to new parameter order

### Info (Review)
1. Potential performance gains with PHP 8.4 JIT
2. Consider using new property hooks feature
3. DOM HTML5 parser available as replacement for DOMDocument

## Composer Dependencies
- ✓ symfony/symfony: 7.1 supports PHP 8.4
- ✓ drupal/core: 11.0 supports PHP 8.4
- ⚠ phpunit/phpunit: Update to 11.x for PHP 8.4 support

## Next Steps
1. Fix 3 implicitly nullable parameter warnings
2. Update hash_pbkdf2() usage
3. Update phpunit: composer require --dev phpunit/phpunit:^11
4. Test in PHP 8.4 environment before production upgrade
```

**Integration Points**:
- Uses `php84-compat` skill knowledge
- Can be called from `/drupal11-readiness`

---

#### 1.5 New Command: `/deprecation-fix`

**File**: `.claude/commands/deprecation-fix.md`

**Usage**: `/deprecation-fix [path]`

**Purpose**: Automated deprecation detection and fixing with Rector

**Workflow**:
1. **Install Rector and drupal-rector**
   ```bash
   composer require --dev rector/rector
   composer require --dev palantirnet/drupal-rector
   ```

2. **Create/Update rector.php Configuration**
   ```php
   // rector.php
   use Rector\Config\RectorConfig;
   use DrupalRector\Set\Drupal10SetList;

   return RectorConfig::configure()
     ->withPaths([
       __DIR__ . '/modules/custom',
       __DIR__ . '/themes/custom',
     ])
     ->withSets([
       Drupal10SetList::DRUPAL_10,
     ]);
   ```

3. **Dry Run - Preview Changes**
   ```bash
   vendor/bin/rector process --dry-run
   ```

4. **Show User Preview and Request Confirmation**
   ```
   Found 23 deprecations that can be auto-fixed:

   1. Replace deprecated entity_load() with entityTypeManager (8 files)
   2. Replace @Translation with TranslatableMarkup import (5 files)
   3. Update plugin annotations to attributes (10 plugins)

   Apply these fixes? [y/N]
   ```

5. **Apply Fixes** (if user confirms)
   ```bash
   vendor/bin/rector process
   ```

6. **Run PHPCS to Fix Code Style**
   ```bash
   vendor/bin/phpcs --standard=Drupal,DrupalPractice modules/custom
   vendor/bin/phpcbf --standard=Drupal modules/custom
   ```

7. **Generate Summary**
   ```bash
   git diff --stat
   ```

**Report Format**:
```
# Deprecation Fix Summary

## Changes Applied

### Files Modified: 15

### Fixes by Category:
1. **Entity API Updates** (8 files)
   - Replaced entity_load() → EntityTypeManager->load()
   - Replaced entity_create() → EntityTypeManager->create()

2. **Translation Functions** (5 files)
   - Added TranslatableMarkup imports
   - Replaced t() in annotations with TranslatableMarkup

3. **Plugin Syntax** (10 files)
   - Converted @Block annotations → #[Block] attributes
   - Converted @Plugin annotations → #[Plugin] attributes

### Lines Changed
- Additions: 45
- Deletions: 38
- Net: +7 lines

## Post-Fix Actions Needed
1. Review changes: git diff
2. Run tests: vendor/bin/phpunit
3. Manual review of complex changes in:
   - modules/custom/example/src/Plugin/Block/ComplexBlock.php
4. Commit changes: git add . && git commit -m "Fix deprecations"

## Remaining Manual Fixes (Cannot be automated)
1. jQuery dependencies in theme.js (5 files)
   - Requires rewrite with vanilla JS or modern framework
2. Complex service container usage (2 files)
   - Requires architectural review for proper DI
```

**Safety Features**:
- Always runs with --dry-run first
- Shows diff before applying
- Requires user confirmation
- Creates git checkpoint suggestion
- Runs code style fixes after

**Integration Points**:
- Called by `/drupal11-readiness` workflow
- Uses `drupal-11-upgrade` skill patterns

---

### Phase 2: Testing & Quality Assurance (Medium Priority)

#### 2.1 New Skill: `drupal-testing`

**File**: `.claude/skills/drupal-testing/SKILL.md`

**Purpose**: Comprehensive testing expertise for Drupal (Unit, Kernel, Functional, JS testing)

**Key Coverage**:

**Test Type Selection**:
```
Unit Test → Testing isolated logic, no Drupal services
Kernel Test → Testing with database, entities, services
Functional Test → Full Drupal install, browser testing
FunctionalJavascript → Testing AJAX, JavaScript behaviors
```

**PHPUnit Configuration**:
```xml
<!-- phpunit.xml -->
<phpunit bootstrap="web/core/tests/bootstrap.php">
  <testsuites>
    <testsuite name="unit">
      <directory>./modules/custom/*/tests/src/Unit</directory>
    </testsuite>
    <testsuite name="kernel">
      <directory>./modules/custom/*/tests/src/Kernel</directory>
    </testsuite>
    <testsuite name="functional">
      <directory>./modules/custom/*/tests/src/Functional</directory>
    </testsuite>
  </testsuites>
  <php>
    <env name="SIMPLETEST_BASE_URL" value="http://localhost"/>
    <env name="SIMPLETEST_DB" value="mysql://db:db@localhost/db"/>
    <env name="BROWSERTEST_OUTPUT_DIRECTORY" value="/tmp"/>
  </php>
</phpunit>
```

**Common Patterns**:

**Unit Test Example** (pure PHP logic):
```php
namespace Drupal\Tests\mymodule\Unit;

use Drupal\Tests\UnitTestCase;
use Drupal\mymodule\Calculator;

class CalculatorTest extends UnitTestCase {
  public function testAddition(): void {
    $calc = new Calculator();
    $this->assertEquals(4, $calc->add(2, 2));
  }
}
```

**Kernel Test Example** (with database, services):
```php
namespace Drupal\Tests\mymodule\Kernel;

use Drupal\KernelTests\KernelTestBase;

class MyServiceTest extends KernelTestBase {
  protected static $modules = ['system', 'user', 'mymodule'];

  public function testService(): void {
    $service = $this->container->get('mymodule.my_service');
    $result = $service->doSomething();
    $this->assertNotNull($result);
  }
}
```

**Functional Test Example** (full Drupal):
```php
namespace Drupal\Tests\mymodule\Functional;

use Drupal\Tests\BrowserTestBase;

class MyModuleTest extends BrowserTestBase {
  protected static $modules = ['mymodule', 'node'];
  protected $defaultTheme = 'stark';

  public function testPageAccess(): void {
    $user = $this->createUser(['access content']);
    $this->drupalLogin($user);
    $this->drupalGet('/mymodule/page');
    $this->assertSession()->statusCodeEquals(200);
  }
}
```

**Mocking Services**:
```php
// Mock a service in unit test
$logger = $this->createMock(LoggerInterface::class);
$logger->expects($this->once())
  ->method('error')
  ->with($this->stringContains('Error'));

// Use prophesy for complex mocking
$entityTypeManager = $this->prophesize(EntityTypeManagerInterface::class);
$storage = $this->prophesize(EntityStorageInterface::class);
$entityTypeManager->getStorage('node')->willReturn($storage);
```

**Test Coverage**:
```bash
# Generate code coverage report
vendor/bin/phpunit --coverage-html coverage/ \
  modules/custom/mymodule/tests

# View coverage report
open coverage/index.html
```

**CI/CD Integration**:
```yaml
# .gitlab-ci.yml example
test:
  stage: test
  script:
    - composer install
    - vendor/bin/phpunit --coverage-text --colors=never
```

**Integration Points**:
- Complements `drupal-expert` with testing focus
- Can suggest tests during code review (`drupal-reviewer` agent)

---

#### 2.2 New Command: `/security-scan`

**File**: `.claude/commands/security-scan.md`

**Purpose**: Enhanced automated security scanning beyond manual audit

**Workflow**:
1. **Composer Vulnerability Audit**
   ```bash
   # Built into Composer 2.4+
   composer audit

   # Or use Symfony security checker
   symfony security:check
   ```

2. **Drupal Core Security Updates**
   ```bash
   drush pm:security
   drush ups --security-only
   ```

3. **Security Review Module**
   ```bash
   drush en security_review
   drush security-review
   ```

4. **Scan for Exposed Secrets**
   ```bash
   # Check for API keys, passwords in code
   grep -rE "(api[_-]?key|password|secret|token).*=.*['\"][^'\"]{20,}" \
     modules/custom themes/custom --include="*.php" --include="*.yml"

   # Check for .env files in version control
   git ls-files | grep -E "\.env$|credentials"
   ```

5. **HTTPS and Security Headers Check**
   ```bash
   # Check HTTPS redirect
   curl -I http://example.com | grep -i location

   # Check security headers
   curl -I https://example.com | grep -iE "strict-transport|x-frame|x-content-type|content-security"
   ```

6. **File Permissions Check**
   ```bash
   # Settings.php should not be writable
   ls -la web/sites/default/settings.php
   # Should be: -r--r--r-- (444)

   # Files directory should not have .htaccess gaps
   ls -la web/sites/default/files/.htaccess
   ```

7. **Database Credential Exposure**
   ```bash
   # Ensure settings.php is not in git
   git ls-files | grep settings.php
   # Should return empty (use settings.local.php instead)
   ```

**Report Format**:
```
# Security Scan Report

## Critical Issues (Fix Immediately)
- ❌ 2 Composer packages with known security vulnerabilities
  - symfony/http-kernel: CVE-2023-xxxxx (Update to 6.4.2)
  - guzzlehttp/guzzle: CVE-2023-xxxxx (Update to 7.8.1)
- ❌ settings.php is world-writable (chmod 444 immediately)
- ❌ API key found in mymodule.module line 45

## High Priority
- ⚠️  HTTPS not enforced (no redirect from HTTP)
- ⚠️  Missing Content-Security-Policy header
- ⚠️  Missing X-Frame-Options header
- ⚠️  4 Drupal modules with security updates available

## Medium Priority
- ⚠️  security_review found 3 issues:
  - File permissions on files/ too permissive
  - Untrusted roles can use PHP filter
  - Failed login logging not enabled

## Passed Checks
- ✓ No .env files in version control
- ✓ Settings.php not in git
- ✓ Drupal core is up to date
- ✓ Strong Transport Security header present

## Immediate Actions
1. Update composer dependencies:
   composer require symfony/http-kernel:^6.4.2 guzzlehttp/guzzle:^7.8.1

2. Fix file permissions:
   chmod 444 web/sites/default/settings.php

3. Remove hardcoded API key from mymodule.module
   Move to settings.php: $settings['mymodule_api_key'] = getenv('MYMODULE_API_KEY');

4. Enable HTTPS redirect in .htaccess or settings.php

5. Apply security updates:
   composer update --with-dependencies $(drush pm:security --field=name)

## Follow-up Actions
- Configure security headers (CSP, X-Frame-Options)
- Review security_review suggestions
- Enable failed login logging
- Schedule regular security scans (monthly)
```

**Integration Points**:
- Extends `/security-audit` command with automation
- Uses `drupal-security` skill knowledge
- Could trigger `drupal-security-expert` agent for deep analysis

---

#### 2.3 New Agent: `drupal-security-expert`

**File**: `.claude/agents/drupal-security-expert.md`

**Purpose**: Specialized security review and penetration testing guidance

**Capabilities**:
- Deep vulnerability analysis (OWASP Top 10 for Drupal)
- Security regression testing planning
- Penetration testing guidance (ethical hacking approach)
- Security patch impact analysis
- GDPR/privacy compliance checking
- Accessibility security (preventing screen reader exploits)

**Tools**: All tools except Task, Edit, Write, NotebookEdit (analysis/research only)

**Model**: Sonnet (balance of capability and cost)

**Usage Pattern**:
```
User: /security-scan
[Command runs]
Claude: Found critical API key exposure. Launching drupal-security-expert for deep analysis.
[Agent analyzes code patterns, suggests fixes, checks for similar issues]
```

**Example Analysis**:
```
# Deep Security Analysis: API Key Exposure

## Vulnerability Details
- **Type**: Hardcoded credentials (CWE-798)
- **Severity**: CRITICAL
- **OWASP**: A07:2021 - Identification and Authentication Failures
- **Location**: modules/custom/mymodule/mymodule.module:45

## Exploitation Scenario
1. Attacker accesses public Git repository
2. Finds API key in commit history (even if removed later)
3. Uses key to access external service
4. Potential data breach or service abuse

## Remediation Steps
1. **Immediate**: Revoke compromised API key from provider
2. **Generate new key** with minimal necessary permissions
3. **Store securely** using environment variables:
   ```php
   // settings.php
   $settings['mymodule_api_key'] = getenv('MYMODULE_API_KEY');

   // In module
   $api_key = \Drupal::settings()->get('mymodule_api_key');
   ```
4. **Update .gitignore** to prevent future commits:
   ```
   web/sites/*/settings.local.php
   web/sites/*/settings.secret.php
   .env
   ```
5. **Audit git history** and remove from all commits:
   ```bash
   # Use BFG Repo-Cleaner or git filter-branch
   bfg --replace-text passwords.txt repo.git
   ```

## Similar Vulnerabilities Check
Scanning for other hardcoded credentials...
- Found 2 additional issues:
  1. Database credentials in old migration script (archive/migrate.php)
  2. SMTP password in config file (config/sync/smtp.settings.yml)

## Prevention Recommendations
1. Add pre-commit hook to detect secrets
2. Use secret scanning tools (truffleHog, git-secrets)
3. Implement key rotation policy
4. Use secret management service (Vault, AWS Secrets Manager)
5. Train team on secure credential handling
```

---

### Phase 3: Performance & Advanced Features (Lower Priority)

#### 3.1 New Skill: `drupal-performance`

**File**: `.claude/skills/drupal-performance/SKILL.md`

**Purpose**: Deep performance optimization beyond basic caching

**Key Coverage**:
- Cache strategies (Redis, Memcached, Varnish, CDN)
- Database query optimization (indexes, views, aggregation)
- BigPipe and lazy loading patterns
- Image optimization (WebP, lazy loading, responsive images)
- Frontend performance (Critical CSS, JS optimization, bundling)
- Profiling tools (Blackfire, Xdebug, New Relic)
- Load testing (Apache Bench, K6, JMeter)
- Performance monitoring and alerting

**Example Patterns**:
```php
// Cache expensive operations
$cid = 'mymodule:complex_data:' . $entity_id;
$cached = \Drupal::cache()->get($cid);
if ($cached) {
  return $cached->data;
}

$data = $this->expensiveCalculation($entity_id);

\Drupal::cache()->set($cid, $data, Cache::PERMANENT, [
  'entity:node:' . $entity_id,
]);

return $data;
```

**Redis Integration**:
```php
// settings.php
$settings['redis.connection']['interface'] = 'PhpRedis';
$settings['redis.connection']['host'] = 'redis';
$settings['cache']['default'] = 'cache.backend.redis';
```

**Database Query Optimization**:
```php
// Bad: N+1 query problem
foreach ($nodes as $node) {
  $author = $node->getOwner(); // Queries for each!
}

// Good: Eager loading
$nodes = \Drupal::entityTypeManager()
  ->getStorage('node')
  ->loadMultiple($nids);
\Drupal::service('entity.repository')
  ->loadMultiple('user', array_column($nodes, 'uid'));
```

**Integration Points**:
- Builds on `/performance-check` command
- Complements `drupal-expert` with performance focus

---

#### 3.2 New Command: `/api-documentation`

**File**: `.claude/commands/api-documentation.md`

**Purpose**: Generate comprehensive API documentation for custom modules

**Workflow**:
1. Scan module for classes, services, hooks
2. Extract PHPDoc comments
3. Generate markdown documentation
4. Create usage examples
5. Document service dependencies
6. Create service dependency graph diagram

**Example Output**:
```markdown
# MyModule API Documentation

## Services

### mymodule.my_service

**Class**: `Drupal\mymodule\MyService`
**File**: `src/MyService.php`

**Dependencies**:
- entity_type.manager
- database
- logger.channel.mymodule

**Methods**:

#### doSomething(string $param): array
Does something useful with the parameter.

**Parameters**:
- `$param` (string): Description of parameter

**Returns**: Array of results

**Example**:
```php
$service = \Drupal::service('mymodule.my_service');
$results = $service->doSomething('example');
```

## Hooks

### hook_mymodule_data_alter(&$data, $context)
Allows modules to alter data before processing.

**Parameters**:
- `&$data` (array): Data to be altered
- `$context` (array): Context information

**Example**:
```php
function myothermodule_mymodule_data_alter(&$data, $context) {
  if ($context['type'] === 'special') {
    $data['custom_field'] = 'value';
  }
}
```
```

---

## Implementation Priority

### Immediate (Next Release)
1. ✅ **Installation Documentation** (Linux/WSL steps) - DONE
2. **drupal-11-upgrade skill** - Critical for D11 adoption
3. **php84-compat skill** - PHP 8.4 released Nov 2024
4. **/drupal11-readiness command** - High user value
5. **/php-upgrade-check command** - Complements readiness

### Short Term (1-2 months)
6. **/deprecation-fix command** - High automation value
7. **drupal-testing skill** - Quality improvement
8. **/security-scan command** - Security priority

### Medium Term (3-6 months)
9. **drupal-security-expert agent** - Deep security analysis
10. **drupal-performance skill** - Performance optimization
11. **/api-documentation command** - Developer productivity

---

## Research Needed

### PHP 8.4 Resources
- [ ] Official PHP 8.4 migration guide: https://www.php.net/migration84
- [ ] PHP 8.4 deprecated features list
- [ ] Drupal core PHP 8.4 compatibility testing results
- [ ] Performance benchmarks: PHP 8.4 vs 8.3 for Drupal

### Drupal 11 Resources
- [ ] Drupal 11 change records: https://www.drupal.org/list-changes/drupal?to_branch=11.x
- [ ] OOP hooks complete documentation
- [ ] Annotation to Attribute migration guide
- [ ] StarShot initiative current status
- [ ] Recipes system documentation

### Testing Resources
- [ ] Drupal core PHPUnit setup documentation
- [ ] Nightwatch.js for Drupal documentation
- [ ] Testing trait best practices
- [ ] CI/CD integration patterns for Drupal

### Security Resources
- [ ] OWASP Top 10 2021 Drupal mapping
- [ ] Drupal Security Team advisories archive
- [ ] Security module documentation (security_review, paranoia)
- [ ] Automated security scanning tools comparison

---

## Quality Standards for New Resources

All new resources must meet these standards:

### Code Examples
- ✓ PHP 8.2+ syntax (constructor promotion, typed properties)
- ✓ Attributes not annotations (D11 standard)
- ✓ Dependency injection (no `\Drupal::service()` in classes)
- ✓ Proper error handling and logging
- ✓ Security best practices (CSRF, XSS prevention, access control)

### Documentation
- ✓ Clear activation criteria / usage instructions
- ✓ Real-world examples (not abstract)
- ✓ Troubleshooting sections
- ✓ Links to official documentation
- ✓ Cross-references to related resources

### Testing
- ✓ YAML frontmatter validates
- ✓ Code examples syntax-check
- ✓ Bash commands tested on Linux/WSL
- ✓ Installation tested via `agr add`
- ✓ Resources load correctly in Claude Code

---

## Success Metrics

**User Adoption**:
- Track `agr add` downloads per resource
- Monitor GitHub stars/forks
- Collect user feedback via issues

**Quality Indicators**:
- Code generated follows Drupal standards
- Security issues prevented
- Time saved vs manual implementation
- D11 upgrade success rates

**Community Impact**:
- Contributions from other developers
- Resources shared in Drupal community
- Integration with other tools/workflows

---

## Contributing

To implement any of these planned resources:

1. Create feature branch: `feature/resource-name`
2. Follow resource templates in CLAUDE.md
3. Test thoroughly (see Quality Standards)
4. Submit PR with:
   - Resource file(s)
   - Update to this ROADMAP.md (move to "Completed")
   - Update README.md (add to Available Resources table)
   - Example usage in PR description

---

## Completed

- ✅ Linux/WSL Installation Documentation (2026-01-28)

---

## Future Considerations

**Drupal 12 / PHP 9.x**:
- Monitor Drupal 12 planning (likely 2027+)
- Track PHP 9.0 development
- Plan migration resources when roadmap solidifies

**AI-Assisted Development**:
- AI code review integration
- Automated refactoring suggestions
- Natural language to Drush commands

**Multi-site Management**:
- Multi-site deployment patterns
- Configuration synchronization across sites
- Shared module/theme management

**Cloud Platform Integration**:
- Platform.sh specific patterns
- Acquia Cloud integration
- Pantheon workflow optimization
- AWS/Azure Drupal hosting patterns

**Accessibility**:
- WCAG 2.1 AA/AAA compliance checking
- Automated accessibility testing
- Screen reader compatibility patterns
