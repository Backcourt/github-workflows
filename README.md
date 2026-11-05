# github-workflows

Shared GitHub Actions release workflows for Backcourt plugins.

Each plugin repo keeps a short `release.yml` that calls one of the workflows in this repo when you push a version tag. All the real work (building, checking, packaging and deploying) lives here, so a fix made once applies to every plugin.

There are three release types, one for each place a plugin can be hosted:

| Workflow | Use for | Where the plugin goes |
| --- | --- | --- |
| `release-woocommerce.yml` | Premium plugins sold on WooCommerce.com | GitHub release → WooCommerce.com → GlotPress (translate.backcourt.io) |
| `release-github.yml` | Free plugins served from GitHub through Git Updater | GitHub release → Git Updater cache flush on backcourt.io |
| `release-wporg.yml` | Free plugins on WordPress.org | GitHub release → WordPress.org SVN |

---

## How it works

Every release runs the same shared stage (`build.yml`), then one deploy job that is specific to the release type.

```
 git push origin 1.4.0
        │
        ▼
 plugin repo: .github/workflows/release.yml
        │  uses: Backcourt/github-workflows/.github/workflows/release-<type>.yml@v1
        ▼
 release-<type>.yml
        │  uses: ./.github/workflows/build.yml   (same repo, same commit)
        ▼
┌──────────────────────── SHARED: build.yml ─────────────────────────┐
│ job "build"                                                        │
│  1. Check out the repo (full history, for commit changelogs)       │
│  2. Read name/version from package.json                            │
│  3. Check versions: tag = package.json = plugin header             │
│  4. Detect pre-release (-alpha / -beta / -rc)                      │
│  5. Set up PHP + Composer + WP-CLI (cached)                        │
│  6. composer install --no-dev                                      │
│  7. Translate: generate languages/<textdomain>.pot                 │
│  8. Check vendor-prefixed/ exists (if prefixed)                    │
│  9. Set up Node (cached) and install npm packages                  │
│ 10. Build: npm run release (or npm run betarelease)                │
│ 11. Check the zip contents                                         │
│ 12. Write release notes (changelog.txt or commit log)              │
│ 13. Upload zip + notes + .pot as a workflow artifact               │
│                                                                    │
│ job "github-release"                                               │
│ 14. Create the GitHub release with <slug>-<version>.zip            │
└────────────────────────────────────────────────────────────────────┘
        │
        ▼
 DEPLOY jobs, one set per type:
   woocommerce : WooCommerce.com deploy (as <slug>.zip)  →  GlotPress full sync   (stable)
                 GlotPress additive import                                      (pre-release)
   github      : Git Updater cache flush                                        (stable)
   wporg       : readme.txt Stable tag check  →  WordPress.org SVN deploy       (stable)
```

Pre-releases (`1.4.0-beta.1`, `2.0.0-rc.1`, and so on) still build and still get a GitHub release marked as a pre-release. They are **never** sent to WooCommerce.com, WordPress.org or Git Updater. The one exception is GlotPress (woocommerce only): see [Translations and beta releases](#translations-and-beta-releases).

The deploy jobs never rebuild. They download the exact zip the build job checked, so what lands on WooCommerce.com or WordPress.org is byte-for-byte what is attached to the GitHub release.

### What is shared (`build.yml`)

| Step | Notes |
| --- | --- |
| Checkout, PHP, Composer, WP-CLI, Node, caching | PHP version is an input. Node uses the `node-version` input, then `.nvmrc`, then `24`. Composer steps are skipped if there is no `composer.json`. `npm ci` is used if there is a `package-lock.json`, otherwise `npm install`. |
| Version checking | Fails if the git tag, `package.json` version and `Version:` plugin header don't all match. Reports every mismatch in one run. |
| Pre-release detection | Any version containing `-alpha`, `-beta` or `-rc`. |
| Translate (`.pot`) | Writes `languages/<textdomain>.pot`, where the text domain is `package.json` `textdomain`, then `name`. Runs the plugin's `composer makepot` script if it has one, otherwise `wp i18n make-pot`. Turn off with `makepot: false`. |
| Build | Runs the plugin's own `npm run release`. Pre-releases run `npm run betarelease` if that script exists. |
| Zip verification | Zip exists; `languages/<textdomain>.pot` is in it (if `makepot`); `vendor-prefixed/autoload.php` is in it (if `prefixed`); no `node_modules/`, `strauss.phar` or (if `forbid-vendor`) `vendor/`. |
| Release notes | `changelog: file` reads the `changelog.txt` entry for this version. `changelog: commits` lists commit subjects since the previous tag, skipping version-bump commits. |
| GitHub release | Attaches `<slug>-<version>.zip` with the release notes as the body. |

### What differs

| Type | Deploy jobs | Secrets |
| --- | --- | --- |
| **woocommerce** | 1. Copies the zip to `<slug>.zip` (WooCommerce.com wants no version in the name) and POSTs it to the WooCommerce.com submission runner.<br>2. Uploads `<textdomain>.pot` to `https://translate.backcourt.io/wp-json/gp-language-pack-server/v1/projects/<project>/import`. Stable releases do a full sync after the WooCommerce.com deploy succeeds. Pre-releases do an additive import. A GlotPress failure shows as a warning but doesn't fail the release. | `WCCOM_SUBMISSIONS_USERNAME`, `WCCOM_SUBMISSIONS_PASSWORD`, `GLOTPRESS_USER`, `GLOTPRESS_APP_PASS` |
| **github** | Tells Git Updater on backcourt.io to flush its cached repo data, so sites see the update on their next check. The endpoint returns HTTP 200 even with a bad key, so the job checks the response for `"success":true`. | `BACKCOURT_GITUPDATER_KEY` |
| **wporg** | Unzips the release, checks `readme.txt` `Stable tag` matches the version, then commits it to SVN `trunk` and `tags/<version>`, plus the `.wordpress-org/` assets folder (via `10up/action-wordpress-plugin-deploy`). Translations come from translate.wordpress.org, so there's no GlotPress step. | `SVN_USERNAME`, `SVN_PASSWORD` |

### Translations and beta releases

Premium plugins send their `.pot` file to GlotPress on every release, so translators can work on new strings:

- **Stable release (`1.4.0`):** full sync. Runs after the WooCommerce.com deploy succeeds. New strings are added and strings that are no longer in the code are marked obsolete.
- **Pre-release (`1.4.0-beta.1`):** additive import (`additive=1`). New strings are added, but nothing is marked obsolete, because stable sites still use those strings.

So a beta tag gives translators a head start, and translations for new strings can be ready when the stable version ships. Language packs built in the meantime still include every string the stable version uses.

The upload is the same request the old `grunt upload-pot` task made. The credentials go to curl on stdin (not on the command line or in the URL), and redirects aren't followed. Plugins no longer need the `upload-pot` Grunt task.

---

## Repo layout

```
github-workflows/
├── .github/workflows/
│   ├── build.yml             # shared build + GitHub release (called by the three below)
│   ├── release-woocommerce.yml   # build → WooCommerce.com → GlotPress
│   ├── release-github.yml    # build → Git Updater flush
│   └── release-wporg.yml     # build → WordPress.org SVN
└── README.md
```

GitHub only lets other repos call reusable workflows from a repo's `.github/workflows/` folder, so that is where they live. When a `release-*.yml` file calls `./.github/workflows/build.yml`, GitHub loads `build.yml` from this repo at the same commit. So `@v1` and `@my-branch` both get a matching `build.yml`.

---

## Adding a release to a plugin

### 1. Make sure the plugin follows the conventions

- **`package.json`** with `name` (the plugin slug) and `version`.
- **`npm run release`** writes the zip to **`./deploy/<slug>-<version>.zip`**, with a single top-level `<slug>/` folder inside. An optional **`npm run betarelease`** does the same for pre-releases.
  - Builds that still write `./deploy/<slug>.zip` are accepted with a warning, until every plugin uses the shared Gruntfile tasks.
- A main plugin file **`<slug>.php`** with a `Version:` header. If the file has a different name, set the `main-file` input.
- Translations: either a `composer makepot` script that writes `languages/<textdomain>.pot`, or nothing (the workflow runs `wp i18n make-pot` for you). Set `"textdomain"` in `package.json` if it differs from `name`. Set `makepot: false` to skip translations.
- For woocommerce: the GlotPress project slug, from the `glotpress-project` input, then `"glotpressProject"` in `package.json`, then `name`.
- If dependencies are prefixed with Strauss, set `prefixed: true` so the zip is checked for `vendor-prefixed/autoload.php`.
- If the zip legitimately ships a `vendor/` folder, set `forbid-vendor: false`.
- For `changelog: file`: a `changelog.txt` with headings like `2026-11-05 - version 1.4.0`.
- For wporg: `readme.txt` in the zip with `Stable tag:` set to the new version, and banners/icons/screenshots in `.wordpress-org/`.

### 2. Add the secrets

Add these under **Settings → Secrets and variables → Actions**. It's easier to set them once as **organization secrets** on the Backcourt org and share them with the repos that need them.

| Secret | woocommerce | github | wporg |
| --- | :-: | :-: | :-: |
| `WCCOM_SUBMISSIONS_USERNAME` | ✓ | | |
| `WCCOM_SUBMISSIONS_PASSWORD` | ✓ | | |
| `GLOTPRESS_USER` | ✓ | | |
| `GLOTPRESS_APP_PASS` | ✓ | | |
| `BACKCOURT_GITUPDATER_KEY` | | ✓ | |
| `SVN_USERNAME` | | | ✓ |
| `SVN_PASSWORD` | | | ✓ |

`GITHUB_TOKEN` is provided automatically. Never commit secret values to any repo, including this one.

### 3. Add `.github/workflows/release.yml` to the plugin

Pick the one that matches where the plugin is hosted. `secrets: inherit` passes the plugin repo's secrets through, so you don't have to list them. `permissions: contents: write` lets the workflow create the GitHub release.

#### WooCommerce.com (premium plugins)

```yaml
name: Release

on:
  push:
    tags:
      - '[0-9]+.[0-9]+.[0-9]+*'

permissions:
  contents: write

jobs:
  release:
    uses: Backcourt/github-workflows/.github/workflows/release-woocommerce.yml@v1
    with:
      woo-product-id: '853021'
      prefixed: true
    secrets: inherit
```

#### Public (GitHub + Git Updater)

```yaml
name: Release

on:
  push:
    tags:
      - '[0-9]+.[0-9]+.[0-9]+*'

permissions:
  contents: write

jobs:
  release:
    uses: Backcourt/github-workflows/.github/workflows/release-github.yml@v1
    with:
      prefixed: true
    secrets: inherit
```

The Git Updater slug comes from the `git-updater-slug` input, then `"git-updater-slug"` in `package.json`, then `package.json` `name`.

#### WordPress.org

```yaml
name: Release

on:
  push:
    tags:
      - '[0-9]+.[0-9]+.[0-9]+*'

permissions:
  contents: write

jobs:
  release:
    uses: Backcourt/github-workflows/.github/workflows/release-wporg.yml@v1
    with:
      # Only needed if the WordPress.org slug differs from package.json "name".
      wporg-slug: my-plugin
    secrets: inherit
```

### 4. Release

1. Bump the version in `package.json` and the plugin header (and `readme.txt` `Stable tag` for wporg).
2. Add the changelog entry (if `changelog: file`).
3. Commit, then tag and push:

```bash
git tag 1.4.0 && git push origin 1.4.0
```

Watch the run under the plugin repo's **Actions** tab. If the version check fails, fix the mismatch, then delete and re-push the tag:

```bash
git tag -d 1.4.0 && git push origin :refs/tags/1.4.0
```

---

## Inputs reference

### Shared (all three workflows)

| Input | Default | What it does |
| --- | --- | --- |
| `php-version` | `8.3` | PHP version for Composer and WP-CLI. |
| `node-version` | *(empty)* | Node version. Empty uses `.nvmrc`, then `24`. |
| `main-file` | `<slug>.php` | Plugin file whose `Version:` header is checked. |
| `makepot` | `true` | Generate the `.pot` file and require it in the zip. |
| `prefixed` | `false` | Require `vendor-prefixed/autoload.php` (Strauss) in the zip. |
| `forbid-vendor` | `true` | Fail if the zip contains `vendor/`. |
| `changelog` | `commits` (github)<br>`file` (woocommerce, wporg) | Release notes source: `file` = `changelog.txt`, `commits` = commit subjects since the previous tag. |

### Per type

| Workflow | Input | Default | What it does |
| --- | --- | --- | --- |
| woocommerce | `woo-product-id` | *required* | WooCommerce.com product ID. |
| woocommerce | `glotpress` | `true` | Upload the `.pot` to GlotPress. |
| woocommerce | `glotpress-server` | `https://translate.backcourt.io` | GlotPress site running gp-language-pack-server. |
| woocommerce | `glotpress-project` | `package.json` `glotpressProject`, then `<slug>` | GlotPress project slug. |
| github | `git-updater-slug` | `package.json` `git-updater-slug`, then `<slug>` | Repo slug sent to Git Updater. |
| github | `git-updater-url` | backcourt.io flush endpoint | Git Updater `flush-repo-cache` URL. |
| wporg | `wporg-slug` | `<slug>` | WordPress.org plugin slug. |
| wporg | `assets-dir` | `.wordpress-org` | Folder with banners, icons and screenshots. |
| wporg | `dry-run` | `false` | Run the SVN deploy without committing. |

---

## Versioning this repo

Plugins call these workflows by tag (`@v1`), not `@trunk`, so a change here can't break every plugin's release at once.

- Bug fixes and new optional inputs: move the `v1` tag forward.
  ```bash
  git tag -f v1 && git push -f origin v1
  ```
- Breaking changes (renamed inputs, new required conventions): tag `v2` and move plugins over one at a time.

To try a change before moving `v1`, point one plugin's `release.yml` at a branch (`@my-branch`) and push a pre-release tag like `1.4.1-beta.1`. Pre-releases build and create a GitHub release, but never deploy anywhere else.

This repo must be **public**. If it's private, share it with the other Backcourt repos under **Settings → Actions → General → Access** so they're allowed to call its workflows.

---

## To do

- **Beta translations:** pre-release tags do an additive GlotPress import (see [Translations and beta releases](#translations-and-beta-releases)). Decide whether that is enough, or whether betas need their own GlotPress project.
- **Gruntfile:** move the shared Grunt build tasks into this repo, so every plugin builds `./deploy/<slug>-<version>.zip` the same way. Then remove the `<slug>.zip` fallback from `build.yml`.
