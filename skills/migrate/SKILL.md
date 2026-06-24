---
name: migrate
description: Use when a Spring Boot feature is complete and new or removed properties in application*.yml need to be reflected in Azure App Configuration. Trigger phrases: "generate app config import", "properties migration", "azure config sync", "what properties changed", "create import file for azure".
---

# Azure App Configuration Properties Migration

Generates a JSON import file for Azure App Configuration based on changed Spring Boot YAML properties. Works with any Spring Boot project that uses `spring-cloud-azure-appconfiguration-config`.

## Step 1: Find Changed Config Files

Run:
```bash
git diff master -- $(find . -path "*/src/main/resources/application*.yml" | tr '\n' ' ')
```

If the branch has no changes relative to master yet, try:
```bash
git diff HEAD~1 -- $(find . -path "*/src/main/resources/application*.yml" | tr '\n' ' ')
```

If multiple files have changes, ask the user which one to use. If no changes are found, tell the user and stop.

## Step 2: Ask for Label

Ask:
> "What Azure App Configuration label should be applied to these properties? (e.g. `production`, `staging`, `online`)"

## Step 3: Parse the Diff

From the diff output:
- Lines starting with `+` (not `+++`) = **added** properties
- Lines starting with `-` (not `---`) = **removed** properties
- Use indentation context from surrounding unchanged lines to reconstruct full YAML key paths

**Flattening rules:**

| YAML | Flat key | Value |
|------|----------|-------|
| `parent:\n  child: value` | `parent.child` | `"value"` |
| `feature:\n  enabled: true` | `feature.enabled` | `"true"` |
| `size: 50MB` | `size` | `"50MB"` |
| `list:\n  - a\n  - b` | `list` | `"[\"a\",\"b\"]"` with `content_type: application/json` |

Rules:
- All values are strings (Azure App Config stores everything as strings)
- Booleans and numbers: convert to string (`true` → `"true"`, `100` → `"100"`)
- `${var}` Spring expressions: keep as-is
- **Passwords/secrets** (keys containing `password`, `secret`, `token`, `credential`, `key`): **omit and warn** — these belong in Azure Key Vault

## Step 4: Generate the Import JSON

Create file: `az-appconfig-<feature-name>-<YYYY-MM-DD>.json` at the project root.

Use the Azure App Configuration `appconfig` format:

```json
[
  {
    "key": "feature.property-name",
    "value": "property-value",
    "label": "production",
    "content_type": null,
    "tags": {}
  }
]
```

Set `content_type` to `"application/json"` for list values, `null` for everything else.

## Step 5: Handle Removals

Removed properties cannot go in the import file — list the delete commands separately:

```
# Properties to DELETE from Azure App Configuration:
az appconfig kv delete --name <app-config-name> --key "parent.removed-key" --label "<label>" --yes
```

## Step 6: Print Summary

Output:
1. Path to the generated JSON file
2. Import command:
   ```bash
   az appconfig kv import --name <app-config-name> --source file --path <file> --format appconfig --yes
   ```
3. Delete commands for any removed properties
4. Warnings for any secrets skipped (Key Vault recommendation)
5. Any values flagged for manual review (complex YAML, anchors, multiline strings)

## Example

Diff in `application-local.yml`:
```diff
+reporting:
+  enabled: true
+  max-report-size: 50MB
+  allowed-formats:
+    - pdf
+    - csv
```

Generated `az-appconfig-reporting-2026-06-23.json`:
```json
[
  {
    "key": "reporting.enabled",
    "value": "true",
    "label": "production",
    "content_type": null,
    "tags": {}
  },
  {
    "key": "reporting.max-report-size",
    "value": "50MB",
    "label": "production",
    "content_type": null,
    "tags": {}
  },
  {
    "key": "reporting.allowed-formats",
    "value": "[\"pdf\",\"csv\"]",
    "label": "production",
    "content_type": "application/json",
    "tags": {}
  }
]
```

Import command:
```bash
az appconfig kv import --name my-app-config --source file --path az-appconfig-reporting-2026-06-23.json --format appconfig --yes
```

## Edge Cases

- **No diff found**: Inform user — no migration needed
- **Only comments/whitespace changed**: Skip — no key-value impact
- **Complex YAML** (anchors, aliases, multiline strings): Flag for manual review, don't attempt to flatten
- **Properties already in main `application.yml`**: Note them as "verify if already present in App Configuration"
