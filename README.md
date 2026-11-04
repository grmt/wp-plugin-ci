# wp-plugin-ci

Shared GitHub Actions checks for our own WordPress plugins. The PHP versions
live in one place, [`plugin-checks.yml`](.github/workflows/plugin-checks.yml):
`PHP_FLOOR` (the plugins' minimum and the servers' PHP, now 8.4) and
`PHP_NEWEST` (8.5). When the servers move up, change them there.

## Checks

| Job | What |
|---|---|
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

Inputs: `slug` (required), `artifact`, `ignore-codes` (one per line) and
`exclude-directories` (comma-separated), the latter two passed to Plugin Check.
