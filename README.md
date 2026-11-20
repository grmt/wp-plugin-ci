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
| PHPStan | Static analysis with `szepeviktor/phpstan-wordpress` at `phpstan-level` (default `8`) on the floor PHP |
| Plugin check | [WordPress Plugin Check](https://github.com/WordPress/plugin-check-action) on the floor PHP |
| Repo conventions | `.github/dependabot.yml` exists. It has an entry for each ecosystem the repo uses (`github-actions`, plus `composer` and `npm` when `composer.json` / `package.json` exist), every entry is `weekly`, and each groups `minor` and `patch` updates into one PR (majors stay separate). |

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
- `phpstan-level`: the PHPStan rule level (`0`–`10`), default `8`. Pass `""` to skip the PHPStan check.
- `extra-plugins`: other public plugin repos (`owner/repo`, main branch)
  installed next to this one, e.g. a plugin this one depends on.

## Tests and quality

[`quality.yml`](.github/workflows/quality.yml) is a second shared workflow:

| Job | What |
|---|---|
| PHPUnit | `vendor/bin/phpunit` from the repo's own `composer.json` and `phpunit.xml.dist`, on the floor and the newest PHP. Coverage (pcov, Clover) on the floor, uploaded to Codecov over OIDC. A failed upload doesn't fail CI. |
| WordPress tests | Only with `wordpress-tests: true`. `phpunit-wordpress.xml.dist` from the repo runs against WordPress and its test suite, with MySQL 8.4 as a service and PHPUnit 9.6 with the PHPUnit Polyfills: the WordPress test suite doesn't support newer PHPUnit. These are installed apart from the repo's `vendor/`, so its unit tests can stay on a newer PHPUnit. The matrix is PHP floor and newest × WordPress: the newest patch release of `WP_FLOOR`, and the newest release when that is a different version. Coverage comes from PHP floor with the floor WordPress, and is uploaded to Codecov with the flag `wordpress` and handed to SonarQube Cloud. The job sets `WP_TESTS_DIR`, `WP_CORE_DIR` and `WP_TESTS_PHPUNIT_POLYFILLS_PATH`. The repo's bootstrap loads `$WP_TESTS_DIR/includes/functions.php`, hooks its plugin into `muplugins_loaded` and requires `$WP_TESTS_DIR/includes/bootstrap.php`. |
| SonarQube Cloud | CI-based analysis, with the coverage when PHPUnit ran. Project key `<owner>_<repo>`, name `<repo>`, organisation `<owner>`. After a repo rename, change the key in SonarQube Cloud (project Administration → Update Key); the name follows on the next analysis. Skipped, with a notice, when the repo has no `SONAR_TOKEN` secret. That includes forks and Dependabot's pull requests, which get no secrets. |

It is a separate file because Codecov's OIDC upload needs `id-token: write`. GitHub checks a called workflow's permissions when the run starts, skipped jobs included. Asking for that permission in `plugin-checks.yml` would fail every caller that doesn't grant it.

```yaml
jobs:
  quality:
    uses: grmt/wp-plugin-ci/.github/workflows/quality.yml@main
    permissions:
      contents: read
      id-token: write
    secrets: inherit
```

Inputs:

- `phpunit` (default `true`): set `false` for a repo without PHPUnit tests. SonarQube Cloud still runs.
- `wordpress-tests` (default `false`): also run the WordPress integration tests (see above).
- `tests-directory` (default `tests`): analysed as test code when it exists.
- `sonar-exclusions`: comma-separated paths to skip, on top of `vendor`, `node_modules`, `dist` and the tests.
- A `.sonarcloud.properties` in the repo (the file Automatic Analysis reads) is added after the generated settings, so rule exceptions (`sonar.issue.ignore.multicriteria…`) and test inclusions carry over when a repo moves to CI-based analysis. A key set there replaces the generated one: a `sonar.exclusions` there must list `vendor/**` itself.

A repo also needs:

- **Codecov:** the repo active in Codecov. After a rename, use Resync in Codecov. A `codecov.yml` is optional; [avpvh-gallery's](https://github.com/grmt/avpvh-gallery/blob/main/codecov.yml) makes the statuses informational.
- **SonarQube Cloud:** the project created in SonarQube Cloud, with **Automatic Analysis turned off**. SonarQube Cloud refuses CI-based analysis while it is on. You also need a `SONAR_TOKEN` secret. `grmt` is a user account, so there are no organisation secrets: set it per repo with `bin/set-sonar-token`. Public repos fall under the free plan for open source.

`PHP_FLOOR` and `PHP_NEWEST` appear in both workflows; change them together. `WP_FLOOR` (the minimum WordPress) is in `quality.yml`.

## Repository settings

Some settings need an admin token, so a workflow can't check them:

- `bin/check-repo-settings` lists, for every repo that calls these workflows, whether Dependabot alerts and security updates are on, whether it calls `quality.yml`, and whether it has `SONAR_TOKEN`. `--fix` turns the Dependabot settings on; it never touches secrets.
- `bin/set-sonar-token [owner/repo ...]` asks for the token without echoing it and stores it as an Actions secret. Without arguments it stores it in every calling repo.
- `bin/repos` prints the repos that call these workflows.
