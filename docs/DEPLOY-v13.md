# Deploy runbook — spirit.org TYPO3 v12 → v13 (cPanel + SSH)

Production is shared **cPanel** hosting (not Cloudways) with **SSH/Terminal**, **Composer
on the host**, and `mysql` CLI available. Docroot is **`public_html/`** (not `public/`).
Production is still on TYPO3 **v12**, so this deploy runs the full **v12 → v13 upgrade on
the server** — not a small patch.

**Production is NOT a git repo yet** — its home dir is the app root (holds `composer.json`,
`config/`, `public_html/`, `vendor/`) with the running v12 install. This deploy *adopts*
that existing directory into git for the first time (step 3), then upgrades it.

**Site-package model — `path` repo, populated by the submodule.** The committed
`composer.json` requires only `eduardo-frank/bca13: "@dev"` via a **`path` repository**
(`packages/bca13`). So the install order on prod is: `git submodule update --init` *fills*
`packages/bca13` (the parent repo's `.gitmodules` supplies the URL), then `composer install`
reads the package from that path. A plain pull/checkout alone does NOT fetch submodule
contents.

Note: prod's *current* (v12) `composer.json` pulls the old `bca12` package via a **VCS** repo
into `vendor/`, and prod has **no `packages/` dir**. The v13 `composer.json` does not require
`bca12` at all — it's retired — so nothing on prod depends on it after the checkout.

**Safety — what adoption will NOT touch** (gitignored, so never in the committed tree):
`vendor/`, `var/`, `public_html/{_assets,typo3,typo3conf/ext,typo3temp,fileadmin}/`, and
`config/system/settings.php`. Prod's `.htaccess` and `config/system/additional.php` are
untracked-and-not-in-the-repo, so they're left alone too. The force-checkout only
overwrites **v12 tracked files** (index.php, composer.*, config.yaml, icons) with their v13
versions — which is the point.

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
cd ~                           # app root (has composer.json, public_html/, config/, vendor/)
php -v                         # must be PHP 8.2+ — switch via cPanel PHP selector if not
vendor/bin/typo3 --version     # confirm prod is v12
ls -a | grep -c '^\.git$'      # expect 0 — confirms prod is not yet a git repo
ls -d packages/* 2>/dev/null   # note existing v12 package dir(s), e.g. packages/bca12 (moved aside in step 3)
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

## 3. Deploy code — adopt the existing install into git, then check out v13

First-time adoption of the running directory. Do this from the app root (`cd ~`).

```bash
# 3a. Prod has no packages/ dir (the v12 bca12 package came from a Composer VCS repo into
#     vendor/). This guard is a no-op on prod; it only matters if a stray packages/ exists.
[ -d packages ] && mv packages packages.v12.bak

# 3b. Initialise git in place and point it at the remote (nothing is overwritten yet).
git init
git remote add origin https://github.com/eduardofrank/spiritorg.git
git fetch origin

# 3c. OPTIONAL safety inspect: see exactly what differs before any file is written.
git reset origin/main          # mixed: sets index to v13, leaves the working tree untouched
git status                     # gitignored env files (settings.php, fileadmin, vendor…) show as untracked = safe

# 3d. Check out v13 (force-overwrites only the v12 *tracked* files), then populate submodules.
git checkout -f -b main origin/main
git branch --set-upstream-to=origin/main main
git submodule update --init --recursive        # CRITICAL — clones bca13 at the pinned commit

# 3e. Build v13 dependencies (regenerates vendor/, drops fluid-styled-content, etc.)
composer install --no-dev --optimize-autoloader
```

> If 3d's `git checkout -f` reports an untracked file would be overwritten that you need to
> keep, stop and copy it aside first. The known env files are all gitignored, so this should
> only ever be plain v12 code being replaced. `packages.v12.bak/` is your rollback for the
> old site package; delete it only once prod is confirmed stable.

## 4. Import the local database (carries all migrations)

Because the DB is **ported** (not migrated in place), import it **before** the TYPO3
maintenance commands — running upgrade wizards on the old v12 DB right before overwriting it
is wasteful and can error. The dump already contains every v12→v13 DB change: the form
`EXT:bca12→bca13` `persistenceIdentifier` fix, the Disqus `list_type→CType` migration, the
`sys_template` `clear=3` cleanup, the map-link fix, and the upgrade-wizard "done" flags.

```bash
# local dump already exists (deploy.sql from `ddev export-db`). Upload it to the server, then:
mysql -u USER -p DBNAME < deploy.sql            # REPLACES the prod DB (backup taken in step 1)
```

Notes:
- `ddev export-db` dumps tables only (no CREATE DATABASE/USE), so it imports into the existing
  prod DB. For a fully clean slate you may drop+recreate the DB first, but it isn't required.
- The imported DB was used with the **local** encryption key; prod keeps its own
  `settings.php` (and key). Effect is harmless — BE sessions reset, so **log into the prod
  backend with your LOCAL admin credentials** afterwards. No DB-stored secrets depend on the key.

## 5. TYPO3 maintenance (against the now-v13 DB)

```bash
vendor/bin/typo3 extension:setup                      # register extensions against the v13 DB
vendor/bin/typo3 database:updateschema "*.add,*.change" # align schema drift — should report ~nothing
vendor/bin/typo3 cache:flush
```
Skip `upgrade:run` — the wizards already ran locally and their "done" flags are in the
imported DB. `fluid-styled-content` needs no manual step — it's already out of `composer.lock`.

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
