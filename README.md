# Azure App Configuration Migration — Claude Code Plugin

A Claude Code plugin that generates Azure App Configuration import files from Spring Boot YAML property changes. After completing a feature that adds or removes properties in `application*.yml`, run this skill to get a ready-to-import JSON file and the exact `az appconfig kv import` command.

## What it does

1. Diffs `application*.yml` against `master` to detect added/removed properties
2. Asks which Azure App Configuration label to use (e.g. `production`, `staging`)
3. Flattens YAML to dot-notation keys and outputs an `appconfig`-format JSON file
4. Prints the `az appconfig kv import` command and any `az appconfig kv delete` commands for removed properties
5. Warns about secrets that belong in Azure Key Vault instead

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

Claude will diff the config files, ask for a label, and generate the import JSON.

## Example output

Given a diff that adds a `reporting` section to `application-local.yml`, the skill generates:

**`az-appconfig-reporting-2026-06-23.json`**
```json
[
  { "key": "reporting.enabled", "value": "true", "label": "production", "content_type": null, "tags": {} },
  { "key": "reporting.max-report-size", "value": "50MB", "label": "production", "content_type": null, "tags": {} }
]
```

**Import command**
```bash
az appconfig kv import --name my-app-config --source file --path az-appconfig-reporting-2026-06-23.json --format appconfig --yes
```

## Requirements

- Claude Code v2.1.128+
- Spring Boot project with `spring-cloud-azure-appconfiguration-config`
- Git repository with a `master` branch to diff against

## Uninstall

```bash
/plugin uninstall azure-appconfig-migration@qiagen-claude-plugins
```
