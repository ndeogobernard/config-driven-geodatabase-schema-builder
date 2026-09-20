# Config-Driven Geodatabase Schema Builder

Build a complete ArcGIS file geodatabase — feature datasets, feature classes, fields, domains,
subtypes, relationship classes, topology, and attribute rules — from **one YAML file**.

No clicking through Catalog. No undocumented database that only one person understands. The
schema is a reviewable text file, the build is one command, and a rebuild takes minutes.

```bash
python -c "import sys; sys.path.insert(0,'src'); from li import gdb; gdb.build_schema(overwrite=True)"
```

## Why

Geodatabase design is usually done by hand and then documented afterwards, if at all. The
documentation drifts, nobody can diff two versions, and rebuilding means repeating the clicks.

Making the schema the source of truth fixes all three:

- **Reviewable** — the schema is a YAML file, so a design change shows up in a pull request diff.
- **Reproducible** — one command rebuilds it identically on any machine.
- **Testable** — the schema can be validated *before* a build, catching the errors that would
  otherwise surface several minutes in.
- **Self-describing** — reference tables are seeded from config, so the delivered geodatabase
  explains itself.

## What it builds

From [`config/schema.yaml`](config/schema.yaml):

| Component | Supported |
|---|---|
| Feature datasets | ✅ with spatial reference |
| Feature classes | ✅ point / multipoint / polyline / polygon |
| Standalone tables | ✅ |
| Fields | ✅ types, lengths, aliases, nullability, defaults |
| Generated field families | ✅ e.g. `c01_raw … c11_raw` from one rule |
| Coded-value domains | ✅ |
| Range domains | ✅ |
| Subtypes | ✅ with per-subtype field defaults |
| Relationship classes | ✅ simple, 1:M and M:1 |
| Topology + rules | ✅ |
| Attribute rules | ✅ calculation and constraint (Arcade) |
| Global IDs | ✅ |
| Seeded reference tables | ✅ from config |

## Example

```yaml
domains:
  dm_ScreenStatus:
    type: coded
    field_type: TEXT
    values: {Pass: Pass, Fail: Fail, Review: Manual review required}

  rg_Score:
    type: range
    field_type: DOUBLE
    min: 0
    max: 100

feature_classes:
  CandidateSites:
    dataset: Analysis
    geometry: POLYGON
    fields:
      - {name: cand_id,       type: TEXT,   length: 64, nullable: false}
      - {name: acres,         type: DOUBLE, domain: rg_Acres}
      - {name: screen_status, type: TEXT,   length: 10, domain: dm_ScreenStatus}

attribute_rules:
  - name: calc_Parcel_Acres
    table: Parcels
    type: CALCULATION
    field: acres
    triggers: [INSERT, UPDATE]
    script: "return $feature.Shape_Area / 43560;"
```

## Validate before you build

The test suite checks the schema **without arcpy**, so it runs in CI and finishes in under a
second — catching the class of error that otherwise appears several minutes into a build:

```bash
python -m pytest tests/ -q
```

- Every field's domain is declared, and its type matches the field's type
- Relationship keys exist on **both** sides
- Subtype fields are integer — ArcGIS rejects text ones
- No duplicate field names; TEXT fields declare a length
- Topology participants live in the topology's own feature dataset
- Constraint rules declare an error number and message

## Requirements

- **ArcGIS Pro 3.x** (any licence level) for the build itself
- Python 3.11 with `pyyaml` — the stock `arcgispro-py3` environment already has it
- Validation alone needs only `pytest` and `pyyaml`; **no arcpy**

## Two things worth knowing

Both cost real debugging time to find:

- **Attribute rules require Global IDs.** Without them the tool fails with `ERROR 002710`. The
  builder adds Global IDs to every class before adding rules.
- **Subtype fields must be SHORT or LONG.** A text field cannot be a subtype field, so a text
  classification needs a paired integer field. The demonstration schema does exactly this with
  `land_use_class` / `land_use_st`.

## Demonstrated on

The example schema is the real one from a distribution-centre site-selection study: 10 feature
datasets, 29 feature classes, 11 tables, 11 domains, 10 relationship classes, a topology with
3 rules, and 5 attribute rules. It builds in about 3.5 minutes.

Nothing here is specific to that study — the builder reads whatever schema you give it.

## Status

The builder is complete and in use. The ERD, data dictionary, design-rationale write-up, and
the optional PostGIS DDL are **pending**: the parent study is reconciling its schema against
real source data, and publishing documentation that will shortly be wrong would be worse than
publishing none. They land here once that reconciliation is done.

## License

MIT — see [LICENSE](LICENSE).

---

A component of [dsg-dfw-site-selection](https://github.com/ndeogobernard/dsg-dfw-site-selection),
a DFW regional-DC site-selection system.
