# artifact-repository

## Feature exercised

A Composer project that resolves a package exclusively from a `type: artifact` repository — a local directory of ZIP archives — exercising Mend SCA's ability to detect the `url` source type and record the artifact path rather than misclassifying the package as coming from Packagist (`registry`) or a path-type repository (`local`).

## How the artifact repository works

Composer's `artifact` repository type lets you declare a local directory of ZIP files as a package source. Each ZIP must contain a valid `composer.json` at its root. Composer scans the directory, reads the embedded `composer.json` files, and resolves packages from those archives exactly like it would from a real registry — but the resolved `dist.url` in the lockfile points at the local ZIP path rather than a remote URL.

The `artifacts/` directory here contains one archive:

```
artifacts/
└── acme-local-lib-1.0.0.zip   # contains composer.json for acme/local-lib 1.0.0
```

The root `composer.json` declares:

```json
{
  "repositories": [{"type": "artifact", "url": "./artifacts"}],
  "require": {"acme/local-lib": "^1.0"}
}
```

After `composer install`, the lockfile entry for `acme/local-lib` shows:

```json
{
  "dist": {
    "type": "zip",
    "url": "./artifacts/acme-local-lib-1.0.0.zip",
    "shasum": "ddb121b15dec27cda71fe13db1b1f63ae99f5507"
  }
}
```

## Source-type assertion contract

Mend MUST report `acme/local-lib` with `source = "url"`.

| Package | Version | Expected source | Must NOT report as |
|---|---|---|---|
| `acme/local-lib` | 1.0.0 | `url` | `registry`, `local` |

The `source_detail` must capture the artifact URL/path: `./artifacts/acme-local-lib-1.0.0.zip`.

## Expected dependency tree

### Direct dependencies (`require`)

| Package | Version | Source | dist.type |
|---|---|---|---|
| `acme/local-lib` | 1.0.0 | url | zip |

### Summary

- Total packages in lockfile: **1**
- Direct (`require`): **1**
- Transitive: **0**
- Dev packages (`require-dev`): **0**
- Source: **artifact repository (local ZIP)**

## Mend failure modes exercised

- **Source misclassified as `registry`**: Mend treats artifact-sourced packages as if they came from Packagist.
- **Package entirely missing**: Mend skips packages it cannot resolve via Packagist lookup.
- **Source misclassified as `local`**: Mend confuses artifact repositories with path-type repositories.
- **Artifact URL lost**: Mend detects the package but drops the `dist.url` from its output.

## Reproducing locally

```bash
# Requires Docker
docker run --rm -v "$PWD":/app -w /app composer:2.8 composer install --no-dev

# Verify the lockfile
jq '.packages[] | {name, version, dist}' composer.lock
```

Expected output:

```json
{
  "name": "acme/local-lib",
  "version": "1.0.0",
  "dist": {
    "type": "zip",
    "url": "./artifacts/acme-local-lib-1.0.0.zip",
    "shasum": "ddb121b15dec27cda71fe13db1b1f63ae99f5507"
  }
}
```

## Probe metadata

```
pattern:          artifact-repository
pm:               composer
lockfile:         composer.lock
php_minimum:      8.1
composer_minimum: 2.6
generated:        2026-04-29
target:           remote
repo:             https://github.com/mend-detection-qa/composer-artifact-repository-probe
```