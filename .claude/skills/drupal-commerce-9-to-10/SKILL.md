---
name: drupal-commerce-9-to-10
description: Drupal Commerce 9-to-10 upgrade expertise. Use when upgrading Drupal Commerce sites from Drupal 9 to Drupal 10, resolving Commerce-specific update hooks, or recovering from failed database updates.
---

# Drupal Commerce 9→10 Upgrade Expert

You are an expert in upgrading Drupal Commerce sites from Drupal 9 to Drupal 10, including diagnosing and recovering from Commerce-specific update hook failures.

## When This Activates

- Upgrading a Drupal 9 Commerce site to Drupal 10
- Resolving failed `drush updatedb` runs involving Commerce modules
- Debugging `commerce_stripe`, `dblog`, or `system.schema` update failures
- Planning a major-version Drupal upgrade on a site with Commerce, Stripe, or payment modules

## Upgrade Phases

The upgrade follows four phases: **Inventory → Composer Upgrade → Database Updates & Recovery → Cutover**. Expect blockers in Phase 3 — the recipes below are first-class recovery procedures, not edge cases.

---

## Phase 1: Inventory

Before touching Composer, audit the current state.

### Check Drupal and PHP Versions

```bash
ddev drush status --fields=drupal-version,php-version,db-driver
```

### List Installed Modules and Versions

```bash
ddev drush pm:list --status=enabled --format=table
ddev composer show drupal/* --format=table
```

### Identify Deprecated or Incompatible Modules

```bash
# Check for Drupal 10 compatibility
ddev exec vendor/bin/drupal-check -ad modules/contrib/
ddev exec vendor/bin/drupal-check -ad modules/custom/
```

If `drupal-check` is not installed:

```bash
ddev composer require --dev mglaman/drupal-check
```

### Export Current Configuration

```bash
ddev drush config:export -y
```

Commit the config export — this is your rollback baseline.

### Audit Migration Modules and Groups

Commerce sites frequently have active migration modules — from initial data imports, ongoing feeds, or a prior D7→D9 migration. These must be inventoried before upgrading.

```bash
# List all enabled migration modules
ddev drush pm:list --status=enabled --type=module | grep -i migrat

# List all registered migration groups
ddev drush migrate:status --group=all 2>/dev/null || ddev drush migrate:status
```

If `migrate_tools` is installed, list every migration group and its state:

```bash
ddev drush sqlq "SELECT id, label FROM config WHERE name LIKE 'migrate_plus.migration_group.%';" | cat
```

Check for Commerce-specific migration modules:

| Module | Purpose | D10 Action |
|--------|---------|------------|
| `commerce_migrate` | Ubercart/Commerce 1.x → Commerce 2.x | Check for D10-compatible release; remove if migration is complete |
| `migrate_drupal_commerce` | D7 Commerce → D9/D10 Commerce | Verify compatibility; may need patch for D10 |
| `commerce_feeds` | Product feed imports | Check D10 release; consider replacement with `migrate_plus` |
| `migrate_plus` | Config-based migrations, groups | Update to D10-compatible version (`^6`) |
| `migrate_tools` | Drush commands for migrations | Update to D10-compatible version (`^6`) |
| `migrate_file` | File migration handling | Update to D10-compatible version |

#### Check for In-Progress or Stuck Migrations

Migrations left in an "Importing" state will block `updatedb`:

```bash
# Show migration status — look for "Importing" or "Stopping" states
ddev drush migrate:status 2>/dev/null

# Reset any stuck migrations before upgrading
ddev drush migrate:reset-status <migration_id>
```

If you have many stuck migrations:

```sql
-- Find all migrations in a non-idle state
SELECT m.name, m.value FROM key_value m
WHERE m.collection = 'migrate_status'
  AND m.value != 'i:0;';

-- Reset all to idle (value 0)
UPDATE key_value
SET value = 'i:0;'
WHERE collection = 'migrate_status';
```

#### Decide: Keep or Remove Migration Modules

**If the original migration is complete** (all data is in D9 and verified):

```bash
# Uninstall migration modules cleanly before upgrading
ddev drush pm:uninstall commerce_migrate migrate_drupal migrate_drupal_ui -y
ddev drush cr
```

Uninstalling before the D10 upgrade avoids update hook failures from migration modules that lack D10 releases.

**If migrations are still active** (ongoing feeds, periodic imports):

Keep the modules but ensure every one has a D10-compatible release. Check each on drupal.org:
- `migrate_plus` ≥ 6.0 for D10
- `migrate_tools` ≥ 6.0 for D10
- `commerce_migrate` — check issue queue for D10 status

#### Export Migration Group Configuration

If you are keeping migration modules, export their config so you can restore after upgrade:

```bash
# Export migration group and migration configs
ddev drush config:export --destination=/tmp/migration-config -y
ls /tmp/migration-config/migrate_plus.migration_group.* 2>/dev/null
ls /tmp/migration-config/migrate_plus.migration.* 2>/dev/null
```

Back these up separately — `updatedb` can alter or delete migration configuration.

### Record Schema Versions

```bash
ddev drush php:eval '$schemas = \Drupal::keyValue("system.schema")->getAll(); ksort($schemas); foreach ($schemas as $name => $version) { echo $name . "\t" . $version . PHP_EOL; }' | cat
```

Save this output. You will need it to diagnose orphaned schema entries in Phase 3.

---

## Phase 2: Composer Upgrade

### Create a Database Snapshot Before Starting

```bash
ddev snapshot --name=pre-d10-upgrade
```

### Update `composer.json` Constraints

The core constraint change:

```bash
ddev composer require drupal/core-recommended:^10 drupal/core-composer-scaffold:^10 drupal/core-project-message:^10 --no-update
```

### Update Commerce and Payment Modules

```bash
ddev composer require drupal/commerce:^2 drupal/commerce_stripe:^1 --no-update
```

Adjust module constraints based on your `composer show` output from Phase 1. Every contrib module needs a D10-compatible release.

### Run the Full Update

```bash
ddev composer update -W
```

The `-W` (or `--with-all-dependencies`) flag is essential — it allows Composer to resolve the full dependency tree including transitive dependencies.

**If Composer fails**, resolve one constraint at a time:
- Remove modules that have no D10 release (`ddev composer remove drupal/module_name`)
- Check drupal.org for D10-compatible releases or patches
- For custom modules, update `core_version_requirement` in `.info.yml` files to `^9 || ^10`

### Verify Codebase State

```bash
ddev drush cr
ddev drush status
```

The site may return errors at this point — that is expected. The database schema has not been updated yet.

---

## Phase 3: Database Updates & Recovery

This is where Commerce sites diverge from simple Drupal upgrades. Run `updatedb` and expect failures:

```bash
ddev drush updatedb -y
```

If it completes cleanly, skip to Phase 4. If it fails, identify the failing update hook from the error output and apply the matching recipe below.

---

### Blocker 1: `commerce_stripe_update_8102` — Column Already Exists

**Symptom:**

```
[error] SQLSTATE[42S21]: Column already exists: 1060 Duplicate column name 'stripe_customer_id'
```

The update hook tries to add a `stripe_customer_id` column to `commerce_payment_method`, but the column was already created by a previous partial run or a schema mismatch.

**Cause:** The schema update ran partially (column was created) but the schema version was not recorded as complete. Re-running `updatedb` tries to add the column again.

**Recovery:**

```sql
-- Verify the column already exists
SHOW COLUMNS FROM commerce_payment_method LIKE 'stripe_customer_id';

-- If the column exists, skip the update by setting the schema version past it
UPDATE key_value
SET value = 's:4:"8102";'
WHERE collection = 'system.schema'
  AND name = 'commerce_stripe';
```

**Verification:**

```bash
ddev drush updatedb-status
# commerce_stripe should no longer list 8102 as pending
```

---

### Blocker 2: `commerce_stripe_update_8104` — Table or Column Missing

**Symptom:**

```
[error] SQLSTATE[42S02]: Base table or view not found: 1146 Table 'db.commerce_stripe_checkout_session' doesn't exist
```

Or a similar error referencing a missing column that `8104` expects to exist from a prior update.

**Cause:** The `8104` hook assumes `8102` and `8103` completed cleanly. If you skipped or partially applied earlier updates, the prerequisite schema changes are missing.

**Recovery — Option A (table is genuinely missing):**

```sql
-- Create the expected table structure
CREATE TABLE IF NOT EXISTS commerce_stripe_checkout_session (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  checkout_session_id VARCHAR(255) NOT NULL,
  order_id INT UNSIGNED NOT NULL,
  created INT NOT NULL DEFAULT 0,
  UNIQUE KEY checkout_session_id (checkout_session_id),
  KEY order_id (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Then re-run: `ddev drush updatedb -y`

**Recovery — Option B (skip if your site does not use Stripe Checkout Sessions):**

```sql
UPDATE key_value
SET value = 's:4:"8104";'
WHERE collection = 'system.schema'
  AND name = 'commerce_stripe';
```

**Verification:**

```bash
ddev drush updatedb-status
ddev drush sqlq "SELECT schema_version FROM key_value WHERE collection = 'system.schema' AND name = 'commerce_stripe';" | cat
```

---

### Blocker 3: `dblog_update_10100` — Long to Int Conversion Failure

**Symptom:**

```
[error] Failed to convert field 'wid' to type 'int' in table 'watchdog'
```

Or:

```
[error] SQLSTATE[22003]: Numeric value out of range
```

**Cause:** Drupal 10 changes the `wid` column in the `watchdog` table from `BIGINT`/`SERIAL` to `INT`. On sites with a large watchdog table, existing `wid` values may exceed the `INT` range (2,147,483,647), or the table may be too large for an in-place `ALTER TABLE` to succeed.

**Recovery:**

```sql
-- Check current max wid
SELECT MAX(wid) FROM watchdog;

-- If max(wid) exceeds 2147483647, truncate or prune first
-- WARNING: This deletes log entries. This is safe — watchdog is operational logging.
TRUNCATE TABLE watchdog;

-- Now the column alteration will succeed on re-run
```

Then re-run:

```bash
ddev drush updatedb -y
```

**Alternative (keep recent logs):**

```sql
-- Keep only the last 100,000 entries and reset wid
CREATE TABLE watchdog_backup AS SELECT * FROM watchdog ORDER BY wid DESC LIMIT 100000;
TRUNCATE TABLE watchdog;
INSERT INTO watchdog SELECT * FROM watchdog_backup;
DROP TABLE watchdog_backup;
```

**Verification:**

```bash
ddev drush updatedb-status
# dblog should no longer list 10100 as pending
ddev drush watchdog:show --count=5
```

---

### Blocker 4: Orphaned `system.schema` Entries

**Symptom:**

```
[error] Module 'some_removed_module' not found.
```

Or `updatedb` crashes because it tries to run update hooks for a module that no longer exists in the codebase.

**Cause:** A module was removed via Composer (or deleted from `modules/`) without first being uninstalled through Drupal. The `key_value` table still has a `system.schema` entry for it, so Drupal thinks the module is installed and tries to update it.

**Recovery:**

```sql
-- List all orphaned schema entries (modules in schema but not in codebase)
SELECT name FROM key_value
WHERE collection = 'system.schema'
  AND name NOT IN ('system')
ORDER BY name;
-- Compare this list against: ddev drush pm:list --status=enabled --field=machine_name

-- Remove specific orphaned entries
DELETE FROM key_value
WHERE collection = 'system.schema'
  AND name = 'some_removed_module';
```

Also clean up `core.extension` config so Drupal does not try to load the module:

```bash
# Check if the module is still listed in core.extension
ddev drush config:get core.extension module.some_removed_module 2>/dev/null

# If listed, remove it by editing core.extension directly
ddev drush php:eval "\$config = \Drupal::configFactory()->getEditable('core.extension'); \$modules = \$config->get('module'); unset(\$modules['some_removed_module']); \$config->set('module', \$modules)->save();"
ddev drush cr
```

**Verification:**

```bash
ddev drush updatedb-status
# No errors about missing modules
ddev drush cr
```

---

### Post-Update: Verify Migration Module Group Integrity

After all update hooks pass, verify that migration modules and their groups survived the upgrade intact.

```bash
# Check migration module status
ddev drush pm:list --status=enabled | grep -i migrat

# List all migration groups
ddev drush migrate:status 2>/dev/null
```

**If migration config was deleted by `updatedb`:**

Re-import the migration group configuration you backed up in Phase 1:

```bash
ddev drush config:import --partial --source=/tmp/migration-config -y
ddev drush cr
ddev drush migrate:status
```

**If `migrate_plus` or `migrate_tools` update hooks changed group structure:**

```bash
# List current migration group configs
ddev drush config:list | grep migrate_plus.migration_group

# Verify each group still contains its expected migrations
ddev drush migrate:status --group=<group_name>
```

**Common issue — migration map tables renamed or dropped:**

D10 schema changes can affect `migrate_map_*` and `migrate_message_*` tables. Check that map tables still exist for active migrations:

```sql
SHOW TABLES LIKE 'migrate_map_%';
SHOW TABLES LIKE 'migrate_message_%';
```

If a map table is missing, the migration will behave as though it has never been run. For ongoing imports this means duplicate content on next run — re-create the map table from a backup or run a rollback-and-reimport.

---

## Phase 4: Cutover

After all update hooks pass:

### Re-export Configuration

```bash
ddev drush config:export -y
```

Review the diff — every change should be explainable as a D9→D10 schema or default update.

### Run Full Status Check

```bash
ddev drush status
ddev drush core:requirements --severity=2
```

### Verify Commerce Functionality

```bash
# Check that Commerce entity types are intact
ddev drush entity:updates
ddev drush sqlq "SELECT COUNT(*) FROM commerce_order;" | cat
ddev drush sqlq "SELECT COUNT(*) FROM commerce_product;" | cat
ddev drush sqlq "SELECT COUNT(*) FROM commerce_payment_method;" | cat
```

### Clear All Caches and Test

```bash
ddev drush cr
ddev launch
```

Walk through critical Commerce paths manually:
- Product display pages
- Add to cart → checkout flow
- Payment processing (use Stripe test mode)
- Order admin pages (`/admin/commerce/orders`)

### Commit the Final State

```bash
ddev drush config:export -y
git add -A
git commit -m "Drupal 10 upgrade complete"
```

---

## Quick Reference: Schema Version Overrides

When skipping a failed update hook, match the `value` format exactly. Drupal stores schema versions as serialized PHP strings:

```sql
-- Pattern: 's:<length>:"<version>";'
-- Examples:
UPDATE key_value SET value = 's:4:"8102";' WHERE collection = 'system.schema' AND name = 'commerce_stripe';
UPDATE key_value SET value = 's:4:"8104";' WHERE collection = 'system.schema' AND name = 'commerce_stripe';
UPDATE key_value SET value = 's:5:"10100";' WHERE collection = 'system.schema' AND name = 'dblog';
```

**Warning:** Only skip an update hook when you have confirmed the schema change it applies either already exists or is not needed by your site. Blindly skipping updates will cause data loss or runtime errors.

## Pre-Upgrade Checklist

- [ ] Database snapshot taken (`ddev snapshot`)
- [ ] Configuration exported and committed
- [ ] `system.schema` versions recorded
- [ ] `drupal-check` run against all custom and contrib modules
- [ ] Contrib modules verified for D10-compatible releases on drupal.org
- [ ] Custom module `.info.yml` files updated with `core_version_requirement: ^9 || ^10`
- [ ] Staging/dev environment — never run this on production first

## Sources

- [Drupal 9 to 10 Upgrade Guide](https://www.drupal.org/docs/upgrading-drupal/upgrading-from-drupal-9-to-drupal-10)
- [Drupal Commerce Documentation](https://docs.drupalcommerce.org/)
- [Commerce Stripe Module](https://www.drupal.org/project/commerce_stripe)
- [Drupal Update API](https://www.drupal.org/docs/drupal-apis/update-api)
