# wp-plugin-ci

Shared GitHub Actions checks for our own WordPress plugins. The PHP versions
live in one place, [`plugin-checks.yml`](.github/workflows/plugin-checks.yml):
`PHP_FLOOR` (the plugins' minimum and the servers' PHP, now 8.4) and
`PHP_NEWEST` (8.5). When the servers move up, change them there.

## Checks

| Job | What |
|---|---|
| Declared PHP floor | The plugin header's `Requires PHP` (required), `readme.txt`'s and `composer.json`'s `require.php` / `config.platform.php` (when present) all state `PHP_FLOOR`. Checked on the build when there is one. |
| Global prefix | Only with the `prefixes` input: WordPress' `PrefixAllGlobals` sniff, so functions, classes, constants, namespaces and hook names start with one of the prefixes (global variables excepted). Checked on the build when there is one. |
| Naming style | Functions, methods and properties follow WordPress (`snake_case`) or PSR (`camelCase`) style, classes `Capitalised_Words` or `PascalCase`, never a mix within one name. Magic methods and a property named `ID` (WordPress's own spelling, as in `WP_Post::$ID`) exempt; `vendor/`, `node_modules/`, `tests/`, `.phan/` skipped. Checked on the build when there is one. |
| Dependencies on PHP floor | `composer check-platform-reqs --no-dev --lock`, if the plugin has a `composer.lock` |
| PHP syntax | `php -l` on every PHP file, on the floor and the newest PHP |
| PHP compatibility | PHPCS with PHPCompatibilityWP, `testVersion` floor and later |
| Plugin check | [WordPress Plugin Check](https://github.com/WordPress/plugin-check-action) on the floor PHP |

`vendor/`, `node_modules/` and `dist/` are skipped by the syntax and
compatibility checks.

## Use

A plugin whose repository is the plugin:

```yaml
jobs:
  plugin-checks:
    uses: grmt/wp-plugin-ci/.github/workflows/plugin-checks.yml@main
    with:
      slug: my-plugin
```

A plugin that is built first (pass the build's artifact; the plugin check
then runs on the build, the other checks on the source):

```yaml
jobs:
  build:
    # ... uploads the built plugin as artifact "build-artifact"
  plugin-checks:
    needs: build
    uses: grmt/wp-plugin-ci/.github/workflows/plugin-checks.yml@main
    with:
      slug: my-plugin
      artifact: build-artifact
      ignore-codes: |
        plugin_updater_detected
```

Inputs:

- `slug` (required): the plugin folder name.
- `artifact`: build artifact holding the plugin files at its root. Without
  it the repo itself is the plugin; its runtime Composer dependencies
  (`composer install --no-dev`) are installed first.
- `ignore-codes`, `exclude-checks`, `exclude-files`, `exclude-directories`:
  passed to Plugin Check, one per line.
- `prefixes`: the prefixes global names must carry, one per line, e.g. `avpvh_flst` (see the naming convention in the plugins' shared CLAUDE.md). Without it the prefix check is skipped.
- `extra-plugins`: other public plugin repos (`owner/repo`, main branch)
  installed next to this one, e.g. a plugin this one depends on.
