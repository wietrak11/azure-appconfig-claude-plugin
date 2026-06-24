# Azure App Configuration Migration — Claude Code Plugin

A Claude Code plugin that generates Azure App Configuration import files from Spring Boot YAML property changes. After completing a feature that adds or removes properties in `application*.yml`, run this skill to get a ready-to-import JSON file.

## What it does

1. Reads the project `artifactId` from `pom.xml` to use as the key prefix
2. Diffs `application*.yml` against `master` to detect added/removed properties
3. Flattens YAML to dot-notation keys and outputs one JSON file per changed config file
4. Reminds you to import via the Azure Portal and lists any keys to delete manually
5. Warns about secrets that belong in Azure Key Vault instead

## Output filename convention

| Source file | Generated file |
|---|---|
| `application.yml` | `propertiesMigration<YYYY-MM-DD>.json` |
| `application-local.yml` | `propertiesMigration<YYYY-MM-DD>-local.json` |
| `application-<env>.yml` | `propertiesMigration<YYYY-MM-DD>-<env>.json` |

## Output format

```json
{
  "/my-app/feature.enabled": "true",
  "/my-app/feature.max-size": "50MB"
}
```

Keys follow the pattern `/<artifactId>/<flattened.yaml.path>`. All values are strings.

## Installation

### 1. Add the marketplace

```bash
/plugin marketplace add wietrak11/azure-appconfig-claude-plugin
```

### 2. Install the plugin

```bash
/plugin install azure-appconfig-migration@wietrak11-plugins
```

### 3. Restart or reload

```bash
/reload-plugins
```

## Usage

After finishing a feature that touched Spring Boot config properties:

```
/azure-appconfig-migration:migrate
```

Claude will diff the config files and generate the import JSON. Import the file via the **Azure Portal — Configuration Explorer → Import**.

## Example output

Given a diff that adds a `reporting` section to `application-local.yml` in a project with `artifactId` `unified-log-parser-backend`:

**`propertiesMigration2026-06-24-local.json`**
```json
{
  "/unified-log-parser-backend/reporting.enabled": "true",
  "/unified-log-parser-backend/reporting.max-report-size": "50MB",
  "/unified-log-parser-backend/reporting.allowed-formats": "[\"pdf\",\"csv\"]"
}
```

Import this file via the Azure Portal — Configuration Explorer → Import.

## Hooks

The plugin installs two hooks automatically:

- **After editing `application*.yml`** — reminds you to run the migration skill before committing
- **Before `git commit`** — blocks the commit if config files are staged and no `propertiesMigration*.json` file exists yet

## Requirements

- Claude Code v2.1.128+
- Spring Boot project with `pom.xml` and `spring-cloud-azure-appconfiguration-config`
- Git repository with a `master` branch to diff against

## Uninstall

```bash
/plugin uninstall azure-appconfig-migration@wietrak11-plugins
```
