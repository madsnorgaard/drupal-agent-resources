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

### Blocker 1: `__PHP_Incomplete_Class` in `DefaultTableMapping`

**Symptom:**

```
PHP Fatal error: Uncaught Error: Cannot use __PHP_Incomplete_Class as array
... in Drupal\Core\Entity\Sql\DefaultTableMapping
```

Often surfaces during `commerce_stripe_update_8102` or `admin_toolbar_update_8003`, but the underlying corruption is independent of either hook and will keep breaking later updates until it is cleared.

**Cause:** The `key_value` row under `entity.definitions.installed` / `commerce_payment_method.field_storage_definitions` holds a serialized entity definition snapshot from the pre-upgrade codebase. During the D10 update run Drupal tries to unserialize it, but the referenced class has been removed or renamed, producing `__PHP_Incomplete_Class` objects that `DefaultTableMapping` then tries to dereference as an array. The site looks broken at the Commerce layer, but the root cause is stale cached entity metadata.

**Recovery:**

```bash
# Backup the row before deleting it
ddev mysql -e "
CREATE TABLE IF NOT EXISTS key_value_backup LIKE key_value;
INSERT INTO key_value_backup
SELECT * FROM key_value
WHERE collection='entity.definitions.installed'
  AND name='commerce_payment_method.field_storage_definitions';
DELETE FROM key_value
WHERE collection='entity.definitions.installed'
  AND name='commerce_payment_method.field_storage_definitions';
"

# Re-run updates; Drupal rebuilds the entity definition from live class code
ddev drush updb -y
```

**Verification:**

```bash
ddev drush updatedb-status
ddev drush php:eval "var_export(\Drupal::keyValue('entity.definitions.installed')->get('commerce_payment_method.field_storage_definitions') !== NULL);"
# Expected: true (a fresh definition has been rebuilt)
```

If `__PHP_Incomplete_Class` errors persist, check for additional entity types with corrupt `field_storage_definitions` rows. The same pattern can affect `commerce_order`, `commerce_product_variation`, and other Commerce entities. Widen the query and repeat the backup-and-delete for each affected row.

---

### Blocker 2: `commerce_stripe_update_8102` — Column Already Exists

**Symptom:**

```
[error] SQLSTATE[42S21]: Column already exists: 1060 Duplicate column name 'stripe_customer_id'
```

The update hook tries to add a `stripe_customer_id` column to `commerce_payment_method`, but the column was already created by a previous partial run or a schema mismatch.

**Cause:** The schema update ran partially (column was created) but the schema version was not recorded as complete. Re-running `updatedb` tries to add the column again.

**Recovery:**

Verify the column is already in place:

```sql
SHOW COLUMNS FROM commerce_payment_method LIKE 'stripe_customer_id';
```

If it exists, advance the schema version past `8102` via the keyValue service. This serializes the value correctly as an integer (`i:8102;`) and avoids the brittle hand-written PHP string format:

```bash
ddev drush php:eval "\Drupal::keyValue('system.schema')->set('commerce_stripe', 8102);"
```

Raw SQL alternative if `drush` is unavailable:

```sql
UPDATE key_value
SET value = 'i:8102;'
WHERE collection = 'system.schema'
  AND name = 'commerce_stripe';
```

**Verification:**

```bash
ddev drush updatedb-status
ddev drush php:eval "echo \Drupal::keyValue('system.schema')->get('commerce_stripe'), PHP_EOL;"
# commerce_stripe should no longer list 8102 as pending and should report 8102 or higher
```

---

### Blocker 3: `commerce_stripe_update_8104` — Hook-Level Error on Missing Payment Method Metadata

**Symptom:**

```
[error] Undefined array key "method_id"
[error] Call to a member function getType() on null
```

Hit during `ddev drush updb -y` while the site is running `commerce_stripe_update_8104`. The hook dereferences payment-method metadata that is missing or shaped differently than the hook expects.

**Cause:** A PHP-level bug inside the `commerce_stripe` update hook when it encounters payment methods whose stored metadata does not match the hook's expected structure. This is not a missing-table or missing-column issue; the hook itself raises a fatal error mid-run.

**Recovery (last-resort workaround, requires follow-up, see below):**

Back up the current `system.schema` row for `commerce_stripe`, then advance the schema version past `8104` so the remaining update hooks and `post_update` steps can run. This skips the failed hook rather than repairing it.

```bash
ddev mysql -e "
CREATE TABLE IF NOT EXISTS key_value_backup LIKE key_value;
INSERT INTO key_value_backup
SELECT * FROM key_value
WHERE collection='system.schema'
  AND name='commerce_stripe';
"
ddev drush php:eval "\Drupal::keyValue('system.schema')->set('commerce_stripe', 8106);"
ddev drush updb -y
```

**Verification:**

```bash
ddev drush updatedb-status
ddev drush php:eval "echo \Drupal::keyValue('system.schema')->get('commerce_stripe'), PHP_EOL;"
```

**Required follow-up before production rollout:**

Skipping `8104` means the data shape the hook was meant to update has not actually been migrated. Before the next production deploy, do the following on a restored pre-skip backup of the database:

1. Read the hook source. Look up `commerce_stripe_update_8104` in `modules/contrib/commerce_stripe/commerce_stripe.install` or on drupal.org. Understand what data it reads and writes.
2. Decide whether the target data shape is in use on this site. If the site does not use the Stripe payment methods or features the hook touches, the skip is safe. Document this in writing.
3. If the data is in use, repair the inputs and re-run the hook rather than keeping the skip. Reset `system.schema` back to the pre-skip value from `key_value_backup` and let `updatedb` execute `8104` against cleaned inputs.
4. Re-verify payments. Run a live Stripe transaction in test mode, confirm `commerce_payment_method` rows look correct, and check that saved customer payment methods still authorise.

Do not ship the skip to production unsupervised.

---

### Blocker 4: `dblog_update_10100` — `ALTER TABLE watchdog` Exceeds MySQL Timeout

**Symptom:**

```
[error] SQLSTATE[HY000]: General error: 2006 MySQL server has gone away
```

Hit during `ddev drush updb -y` while `dblog_update_10100` runs `ALTER TABLE watchdog ... BIGINT`. Less commonly, on sites whose `wid` values have exceeded the old `INT` range, the alteration instead fails with `SQLSTATE[22003]: Numeric value out of range`.

**Cause:** D10 widens the `wid` column in the `watchdog` table to `BIGINT`. On sites with a large watchdog table the in-place `ALTER TABLE` runs longer than MySQL's connection-idle limit (`wait_timeout`, or the local DDEV `max_allowed_packet`), and MySQL drops the connection mid-run. The numeric-overflow variant has the same root shape (the old column can't hold values during the conversion) and clears with the same recovery.

**Recovery (fastest, what we ran in production):**

Uninstall `dblog` before running updates so the alteration is never attempted, then re-enable it afterwards. This is safe: `watchdog` holds operational logging only.

```bash
ddev drush pm:uninstall dblog -y
ddev drush updb -y
ddev drush en dblog -y
ddev drush cr
```

**Alternative (keep dblog installed, shrink the table first):**

If you need to preserve recent watchdog entries, prune the table so the `ALTER TABLE` stays under the MySQL timeout window.

```sql
SELECT COUNT(*) FROM watchdog;

-- Keep only the last 100,000 entries
CREATE TABLE watchdog_backup AS SELECT * FROM watchdog ORDER BY wid DESC LIMIT 100000;
TRUNCATE TABLE watchdog;
INSERT INTO watchdog SELECT * FROM watchdog_backup;
DROP TABLE watchdog_backup;
```

Then re-run:

```bash
ddev drush updb -y
```

**Verification:**

```bash
ddev drush updatedb-status
# dblog should no longer list 10100 as pending
ddev drush watchdog:show --count=5
```

---

### Blocker 5: Orphaned `system.schema` Entries

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

When skipping a failed update hook, prefer the keyValue service. It serializes the integer value correctly and avoids hand-written serialization bugs:

```bash
ddev drush php:eval "\Drupal::keyValue('system.schema')->set('commerce_stripe', 8106);"
ddev drush php:eval "\Drupal::keyValue('system.schema')->set('dblog', 10101);"
```

Raw SQL alternative. Schema versions are stored as serialized integers (`i:<version>;`), not strings. Writing `s:<length>:"<version>";` is brittle, because a miscounted length prefix silently corrupts `system.schema`:

```sql
UPDATE key_value SET value = 'i:8102;' WHERE collection = 'system.schema' AND name = 'commerce_stripe';
UPDATE key_value SET value = 'i:8106;' WHERE collection = 'system.schema' AND name = 'commerce_stripe';
UPDATE key_value SET value = 'i:10101;' WHERE collection = 'system.schema' AND name = 'dblog';
```

**Warning:** Only skip an update hook when you have confirmed the schema change it applies either already exists or is not needed by your site. Blindly skipping updates will cause data loss or runtime errors. For hooks you have not diagnosed, treat the skip as a last-resort workaround and plan a follow-up repair before production rollout (see Blocker 3 for a worked example).

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
