---
name: migrate
description: Use when a Spring Boot feature is complete and new or removed properties in application*.yml need to be reflected in Azure App Configuration. Trigger phrases: "generate app config import", "properties migration", "azure config sync", "what properties changed", "create import file for azure".
---

# Azure App Configuration Properties Migration

Generates a JSON import file for Azure App Configuration based on changed Spring Boot YAML properties. Works with any Spring Boot project that uses `spring-cloud-azure-appconfiguration-config`.

## Step 1: Read App Name from pom.xml

Find the project's own `artifactId` in `pom.xml` — the one that is NOT a parent or dependency. It is typically the second `<artifactId>` in the file, immediately after `<parent>`.

```bash
grep "<artifactId>" pom.xml | head -5
```

Use this value as `<app-name>` in all generated keys.

## Step 2: Find Changed Config Files

Run:
```bash
git diff master -- $(find . -path "*/src/main/resources/application*.yml" | tr '\n' ' ')
```

If the branch has no changes relative to master yet, try:
```bash
git diff HEAD~1 -- $(find . -path "*/src/main/resources/application*.yml" | tr '\n' ' ')
```

If multiple files have changes, process **each file separately** — one output JSON per file. If no changes are found, tell the user and stop.

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
| `list:\n  - a\n  - b` | `list` | `"[\"a\",\"b\"]"` |

Rules:
- All values are strings (Azure App Config stores everything as strings)
- Booleans and numbers: convert to string (`true` → `"true"`, `100` → `"100"`)
- `${var}` Spring expressions: keep as-is
- **Passwords/secrets** (keys containing `password`, `secret`, `token`, `credential`, `key`): **omit and warn** — these belong in Azure Key Vault

## Step 4: Generate the Import JSON

Determine the output filename from the source yml filename (use today's date as `YYYY-MM-DD`):

| Source file | Output file |
|---|---|
| `application.yml` | `propertiesMigration<YYYY-MM-DD>.json` |
| `application-local.yml` | `propertiesMigration<YYYY-MM-DD>-local.json` |
| `application-<env>.yml` | `propertiesMigration<YYYY-MM-DD>-<env>.json` |

Create the file at the project root.

The format is a flat JSON object. Each key is `/<app-name>/<flattened.property.path>`, the value is the property value as a string:

```json
{
  "/<app-name>/parent.child": "value",
  "/<app-name>/feature.enabled": "true"
}
```

## Step 5: Handle Removals

Removed properties cannot go in the import file. List each one so the user can delete them manually via the Azure Portal:

```
# Properties to DELETE from Azure App Configuration (delete via Azure Portal):
/<app-name>/parent.removed-key
/<app-name>/another.removed-key
```

## Step 6: Print Summary

Output:
1. Path to each generated JSON file
2. Reminder: **"Import the file via the Azure Portal — Configuration Explorer → Import."**
3. List of keys to delete manually (if any removed properties)
4. Warnings for any secrets skipped (Key Vault recommendation)
5. Any values flagged for manual review (complex YAML, anchors, multiline strings)

## Example

`artifactId` in `pom.xml`: `unified-log-parser-backend`

Diff in `application-local.yml`:
```diff
+reporting:
+  enabled: true
+  max-report-size: 50MB
+  allowed-formats:
+    - pdf
+    - csv
```

Generated `propertiesMigration2026-06-24-local.json`:
```json
{
  "/unified-log-parser-backend/reporting.enabled": "true",
  "/unified-log-parser-backend/reporting.max-report-size": "50MB",
  "/unified-log-parser-backend/reporting.allowed-formats": "[\"pdf\",\"csv\"]"
}
```

Import via the Azure Portal — Configuration Explorer → Import — using `propertiesMigration2026-06-24-local.json`.

## Edge Cases

- **No diff found**: Inform user — no migration needed
- **Only comments/whitespace changed**: Skip — no key-value impact
- **Complex YAML** (anchors, aliases, multiline strings): Flag for manual review, don't attempt to flatten
- **Properties already in main `application.yml`**: Note them as "verify if already present in App Configuration"
