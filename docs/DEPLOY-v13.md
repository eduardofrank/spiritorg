# Deploy runbook — spirit.org TYPO3 v12 → v13 (cPanel + SSH)

Production is shared **cPanel** hosting (not Cloudways) with **SSH/Terminal**, **Composer
on the host**, and `mysql` CLI available. Docroot is **`public_html/`** (not `public/`).
Production is still on TYPO3 **v12**, so this deploy runs the full **v12 → v13 upgrade on
the server** — not a small patch.

The site package `packages/bca13` is a **git submodule** (`eduardofrank/bca13`); a plain
`git pull` does NOT update it.

The local DB is **ported to production** (full `ddev export-db` → `mysql < dump`), so the
v12→v13 DB migrations travel with the dump — no per-row SQL on prod (see step 5).

> ⚠️ **Site base URL — change on the server before first frontend load** (same manual step
> as the Cloudways procedure). `config/sites/spirit/config.yaml` ships
> `base: 'https://spirit.ddev.site/'` (the DDEV URL); the base is **file-based, not in the
> DB**, so the DB port does not fix it. After deploying the code and importing the DB, edit
> on the server:
> ```yaml
> # config/sites/spirit/config.yaml
> base: 'https://www.spirit.org/'
> ```
> then flush caches, before the first frontend request. Note: `config.yaml` is git-tracked,
> so each subsequent `git pull` re-introduces the DDEV base — re-apply this edit after every
> deploy (or stash/restore it).

---

## 0. Pre-flight (read-only, do first)

Over SSH, in the site root (the dir containing `composer.json`, with `public_html/` as docroot):

```bash
php -v                         # must be PHP 8.2+ — switch via cPanel PHP selector if not
vendor/bin/typo3 --version     # confirm prod is v12
git status && git remote -v    # clean tree + remote = eduardofrank/spiritorg
```

Confirm prod DB matches local assumptions **before** the cleanup in step 5 (prod data may
differ from local):

```sql
-- Is the blog subtree deleted on prod too? (locally it was — uid 65 + children deleted)
SELECT uid, pid, title, deleted FROM pages WHERE uid=65 OR pid=65;

-- Identify the live root template record(s). Only the live root (rootPageId 11) should be touched.
SELECT uid, pid, title, clear, include_static_file FROM sys_template ORDER BY pid;
```

## 1. Backups (do not skip)

```bash
mysqldump -u USER -p DBNAME > ~/pre-v13-$(date +%Y%m%d).sql
tar czf ~/pre-v13-files-$(date +%Y%m%d).tgz public_html/fileadmin config/system/settings.php
```

`config/system/settings.php` is gitignored — it holds the prod DB credentials, SMTP
settings, and the encryption key. The deploy never touches it; just keep a backup.

## 2. Maintenance window

Put up a maintenance page (cPanel, or a temporary holding `index.html`). The upgrade has a
window where the site will not render.

## 3. Deploy code

```bash
git pull origin main
git submodule update --init --recursive        # CRITICAL — populates packages/bca13
composer install --no-dev --optimize-autoloader # upgrades core->13.4, BS->15, installs bca13, drops FSC
```

## 4. TYPO3 upgrade steps

```bash
vendor/bin/typo3 extension:setup
vendor/bin/typo3 database:updateschema "*.add,*.change"
vendor/bin/typo3 upgrade:run
vendor/bin/typo3 cache:flush
```

## 5. Import the local database (carries all migrations)

The local DB is being ported to production, so the v12→v13 DB changes are **already baked
into the dump** — no per-row SQL on prod. That dump already contains: the form
`EXT:bca12→bca13` `persistenceIdentifier` fix, the Disqus `list_type→CType` migration, the
`sys_template` `clear=3` cleanup, and the upgrade-wizard "done" flags.

```bash
# On local:
ddev export-db --gzip=false --file=spirit-prod.sql
# Upload it, then on the server — this REPLACES the prod DB (backup taken in step 1):
mysql -u USER -p DBNAME < spirit-prod.sql
```

`fluid-styled-content` needs no manual step — it is already out of `composer.lock`, so
`composer install` does not reinstall it.

> Because the dump is a full v13 DB at the same schema as the deployed code, the
> `database:updateschema` / `upgrade:run` in step 4 are belt-and-suspenders — run them, but
> they should report nothing to do.

## 6. Set production base URL, then final cache flush + asset rebuild

Apply the base-URL edit from the ⚠️ note above (`config/sites/spirit/config.yaml` →
`https://www.spirit.org/`) **before the first frontend request**, then:

```bash
vendor/bin/typo3 cache:flush
rm -rf var/cache/* public_html/typo3temp/assets/*

# Processed image variants are environment-specific (files live in fileadmin/_processed_,
# per server). The imported dump carries sys_file_processedfile records that reference the
# LOCAL machine's processed files, which don't exist on prod → broken images. Clear them so
# prod regenerates from its own originals:
vendor/bin/typo3 database:query "DELETE FROM sys_file_processedfile;" 2>/dev/null || \
  mysql -u USER -p DBNAME -e "DELETE FROM sys_file_processedfile;"
rm -rf public_html/fileadmin/_processed_/*
vendor/bin/typo3 cache:flush
```
> The original cause (stale **empty-identifier** `sys_file_processedfile` records from the DB
> import making large images serve the raw original, e.g. a `.tif`, → broken-image icon) is
> fully reset by the `DELETE` above.

## 7. Verify (then lift maintenance)

- Homepage renders; navbar logo present and menu right-aligned.
- Section background tints correct (`#F6F9FF` / `#F1F1E6`).
- `/prayer-requests` form **submits** (test the actual POST), no `EXT:bca12` error.
- Disqus thread loads on `/bca-news` and `/prayer-requests` (no "insert your DISQUS
  shortname" message).
- `tail public_html/typo3temp/var/log/typo3_*.log` (or `var/log/typo3_*.log`) is clean.

## 8. Rollback if needed

```bash
git checkout <previous-sha>
git submodule update --init --recursive
composer install --no-dev
mysql -u USER -p DBNAME < ~/pre-v13-YYYYMMDD.sql
```

---

### Notes / gotchas

- Switch the cPanel **PHP version to 8.2+ before** `composer install`, or the platform
  check will abort.
- Only clear the `sys_template` record(s) on the **live root tree** — prod's DB may differ
  from local; verify in step 0.
- The Disqus shortname is now baked into the site package
  (`plugin.tx_nsdisquscomments_comment.settings.ShortName`), so prod's old Extension
  Configuration value is no longer needed.
