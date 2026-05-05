# Full Export of Taxonomy Records and Fields from EMu 9.0

A procedure for exporting **all records** in the Taxonomy module (`etaxonomy`) with **all fields** populated. Common use cases: cold backup outside the EMu backup regime, migration, bulk transformation pipelines (DwC, Linked Art, RDF), full-text indexing.

This is a heavyweight operation. Plan capacity, choose the right tool, and verify the output before treating it as authoritative.

## Decide on tool: GUI vs `texexport`

| Concern | GUI Export | `texexport` |
|---|---|---|
| Record volume | Practical up to low hundreds of thousands | Preferred for any large table |
| Setup overhead | Low; no shell access needed | Requires server access and admin coordination |
| Reproducibility | Save search + field list | Script in version control |
| Field-level security | Honoured (may hide fields) | Bypassed (true full schema) |
| Output location | Client workstation | Server, then transferred |

For a faithful "all fields, all records" dump, `texexport` is the right tool. The GUI is acceptable when the table is modest and shell access is unavailable.

## Capacity planning

Before running:

1. **Record count** — open Taxonomy, press `F7` then `F8` with all fields blank, read the count from the status bar.
2. **Output size estimate** — XML output is typically 5–15× the average record's logical size. For half a million taxonomy records, plan for tens of GB.
3. **Disk headroom** — at least 50% free at the output location.
4. **Window** — schedule outside backup, reorganisation, and high-write periods.
5. **Coordination** — for production registries, notify the administrator.

## Procedure A — GUI export (XML, all fields, all records)

### 1. Open Taxonomy

Launch EMu, authenticate against the target environment, open the **Taxonomy** module. Confirm the environment in the title bar before proceeding.

### 2. Build an unrestricted result set

Press `F7` (Search mode). Leave every field blank. Press `F8`. EMu returns every record in `etaxonomy` visible to the current user. Confirm the count in the status bar against the figure from capacity planning.

> **Permissions caveat.** The GUI honours record- and field-level security. If true full coverage is needed, either run as a user with no restrictions, or switch to the `texexport` path, which operates below the EMu permission layer.

### 3. Sort by IRN ascending

**Tools > Sort** with `irn` ascending. Deterministic ordering makes the output diffable, resumable, and reproducible.

### 4. Open the Export dialog

**Tools > Export…**

### 5. Choose XML format

On the **General** tab, select **XML**. XML preserves nested table (`_tab`) fields, attached references, and grouped fields without the data loss CSV imposes on multi-value structure. CSV is not a sensible choice for a full-schema export of a taxonomy table.

### 6. Configure output

- **Output File** — full path. For large tables, target fast local storage rather than a network share.
- **Encoding** — UTF-8 (non-negotiable for taxonomic content).
- **Overwrite** — confirm intended behaviour.

### 7. Select every field

On the **Fields** tab:

1. Click **Add** to open the field picker.
2. Select every entry in the field list (`Ctrl+A`, or shift-click from first to last).
3. Click **OK** to add them all to the export field list.
4. **Do not expand reference fields.** For a faithful export, leave references as IRNs — they are round-trippable and avoid the runaway expansion that occurs through circular structures (parent → currently accepted → parent → …). Reference resolution belongs in a downstream transform, not in the dump itself.
5. **Save** the field list (e.g. `etaxonomy_all_fields_v1`) so the export is reproducible.

### 8. Execute

Click **OK**. Do not change the result set, switch modules, or close the client until the export completes. Runtime scales roughly linearly with record count; expect minutes to hours.

### 9. Verify

- Validate the XML is well-formed without loading the whole file: `xmllint --noout output.xml`.
- Count record-level elements with a streaming tool (`xmlstarlet sel -t -v "count(//tuple)"` or equivalent for your registry's element name) and compare to the result set count.
- Spot-check a handful of records — including ones with deeply nested `_tab` fields and multiple attached references — against the GUI.
- Compute and record a checksum: `sha256sum output.xml > output.xml.sha256`.

## Procedure B — Server-side `texexport`

### 1. Coordinate with the registry administrator

Confirm: server hostname, path to the `etaxonomy` table data directory, the system user under which `texexport` should run, the output location, and the maintenance window.

### 2. SSH to the EMu server and load the EMu environment

Authenticate as the agreed user. Source the EMu environment script so `texexport` resolves and `TEXAPI` / `TEXCONFIG` are set. The exact script path is install-specific (commonly under `/usr/local/ke/...`).

### 3. Run `texexport` with no query and no field list

With neither a query nor a field list specified, defaults are **all records** and **all fields** — exactly what is wanted here.

```sh
texexport \
  -f xml \
  -o /exports/etaxonomy_$(date +%Y%m%d).xml \
  -t /path/to/registry/data/etaxonomy
```

Verify the flag names against your local `texexport` — flag spellings have varied across Texpress releases. The conceptual arguments are stable: format, output, table.

### 4. Verify on the server

Same checks as Procedure A step 9: well-formed check, record count, spot checks, checksum.

### 5. Transfer off the server

`rsync` or `scp` to the destination. Re-checksum at the destination and compare. Compress (`zstd` or `xz`) before transfer for large outputs.

## Notes and gotchas

- **What "all fields" actually means.** EMu modules carry computed, admin (`Adm*`), security (`Sec*`), and sometimes hidden internal fields. The GUI field picker exposes what your role can see; `texexport` exposes the underlying schema. For migration-grade fidelity, `texexport` is closer to a faithful dump.
- **References stay as IRNs in the dump.** For consumer-facing exports (DwC, Linked Art, JSON-LD), resolve references downstream where the transform is reproducible and inspectable.
- **Multimedia is not in this export.** Binary attachments live in `emultimedia`. A full taxonomy export captures the IRN references; the binaries themselves need a separate `emultimedia` export.
- **Locking.** Long reads do not block writes in Texpress, but very long-held read positions interact poorly with reorganisation jobs. Schedule accordingly.
- **Auditability.** Full exports are sensitive. Log: who ran it, when, against which environment, where the output landed, and the checksum. Treat the artefact as restricted if any record carries sensitive locality, embargoed taxa, or otherwise protected content.
- **Verify against the Axiell documentation for your specific 9.0 build** — UI labels, menu paths, and `texexport` flag names can vary between point releases and locally customised registries.
