# JTOX JSON Schema Definitions

These schemas validate JTOX documents, settings files, and theme metadata.

## Files

| Schema | Validates |
|---|---|
| `node.schema.json` | A single JTOX node (recursive — validates the entire tree) |
| `bind.schema.json` | A `$bind` object (data binding expression) |
| `settings.schema.json` | A settings file (section or component settings) |
| `theme.schema.json` | A `theme.json` file (theme metadata and global settings) |

## Usage

These schemas follow JSON Schema draft-07. Use any JSON Schema validator:

**JavaScript (ajv):**
```js
const Ajv = require('ajv');
const ajv = new Ajv();
const validate = ajv.compile(require('./node.schema.json'));
const valid = validate(myJtoxDocument);
```

**Python (jsonschema):**
```python
import json, jsonschema
schema = json.load(open('node.schema.json'))
jsonschema.validate(instance=my_document, schema=schema)
```

**CLI (ajv-cli):**
```bash
ajv validate -s node.schema.json -d my-template.json
```

## Validation Levels

1. **Structural** — JSON Schema validates this (node.schema.json)
2. **Type** — JSON Schema validates this (conditional rules in node.schema.json)
3. **Binding** — JSON Schema validates syntax (bind.schema.json), path resolution requires theme context
4. **Composition** — Requires loading all theme files to verify $ref targets exist
5. **Theme** — Requires the complete theme package (theme.schema.json + all files)

Levels 1-3 can be done with JSON Schema alone. Levels 4-5 require a JTOX-aware validator.
