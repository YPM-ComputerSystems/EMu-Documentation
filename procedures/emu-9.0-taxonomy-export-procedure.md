# Exporting Taxonomy Records from EMu 9.0

A stepwise procedure for exporting records from the Taxonomy module (`etaxonomy`) in Axiell EMu 9.0. Covers the GUI workflow with notes on server-side alternatives.

## Prerequisites

- EMu 9.0 client installed and configured against the target registry
- User account with read access to the Taxonomy module and the Export operation enabled in the Operations security profile
- Write access to the target output location (local filesystem or mapped network share)
- A clearly defined record set: known search criteria, a saved search, or a saved group

## Procedure

### 1. Launch EMu and open the Taxonomy module

Authenticate against the appropriate environment (production, training, or test). From the **Modules** menu or the launcher, open **Taxonomy**. Confirm in the title bar that you are in the intended environment before exporting.

### 2. Build the result set

Use one of the following to identify the records to export:

1. **Search** — Press `F7` (or **Edit > Search**) to enter Search mode. Populate criteria such as `ClaScientificName`, `ClaRank`, `ClaKingdom`, or `ClaFamily`. Execute with `F8`.
2. **Saved search** — **Tools > Searches > Load** to retrieve a previously saved query.
3. **Saved group** — **Tools > Groups > Load** to retrieve a static set of IRNs.
4. **Hierarchy** — Use the Classification/hierarchy browser to expand a clade and select descendants.

Confirm the record count in the status bar matches expectations before proceeding.

### 3. (Optional) Sort and trim the result set

- **Sort:** **Tools > Sort** — define a sort key (e.g., `ClaKingdom`, then `ClaFamily`, then `ClaScientificName`) so the export is deterministically ordered.
- **Trim:** Select unwanted rows and use **Edit > Delete from Result Set**. This removes them from the working set only; the records remain in the database.

### 4. Open the Export dialog

**Tools > Export…** opens the Export Properties dialog. If the menu item is greyed out, the user lacks the Export operation in their security profile.

### 5. Choose the export format

On the **General** tab, select a format:

- **CSV** — comma-separated, flat. Best for spreadsheet and downstream ETL where hierarchy can be encoded in a few columns.
- **Tab-delimited** — same shape as CSV with tab as the delimiter; safer when fields contain commas.
- **XML** — KE/Axiell EMu XML. Preserves nested table fields (`_tab`) and attached references as structured elements. Use this when synonyms, common names, or reference attachments must round-trip with structure intact.
- **Custom report** — uses a configured report template (Crystal Reports, Stylesheet, or similar) defined under **Tools > Reports**.
- **Darwin Core / DwC-A** — only available if a local DwC export profile or report has been configured. Not part of the stock EMu install; verify with your registry administrator.

For taxonomy work with attached references (currently accepted name, parent, synonyms, common names, citations), XML is usually the right default. CSV requires you to flatten and accept the data loss.

### 6. Specify output destination

Still on the **General** tab:

- **Output File** — full path and filename. Use a stable naming convention (e.g., `etaxonomy_<scope>_<YYYYMMDD>.csv`).
- **Encoding** — set to **UTF-8**. Taxonomy data routinely contains diacritics in author names (`Linné`, `Müller`, `Lamarck ex De Candolle`); legacy encodings will mangle these.
- **Overwrite** — confirm the desired behaviour if the target file already exists.

### 7. Select fields

Switch to the **Fields** tab.

1. Click **Add** to open the field picker. Pick from the Taxonomy schema. Typical selections:
   - **Identifiers:** `irn`, `AdmGUIDValue_tab`, `ClaPreferredFlag`
   - **Name:** `ClaScientificName`, `ClaRank`, `ClaAuthor`, `ClaPublishedYear`, `ClaScientificNameAuthorship`
   - **Hierarchy (as recorded):** `ClaKingdom`, `ClaPhylum`, `ClaClass`, `ClaOrder`, `ClaFamily`, `ClaGenus`, `ClaSpecies`, `ClaSubspecies`
   - **Higher classification (resolved):** `ClaHigherClassification_tab` (if populated by your registry)
   - **Status:** `ClaTaxaStatus_tab`, `ClaCurrentlyAcceptedNameRef`, `ClaCurrentNameRef`
   - **Common names / synonyms:** `ComName_tab`, `ComStatus_tab`, plus the synonym attachment fields used locally
   - **Audit:** `AdmDateModified`, `AdmTimeModified`, `AdmModifiedBy`
2. For attachment (reference) fields, double-click to drill into the linked record and select sub-fields (e.g., `ClaCurrentlyAcceptedNameRef.ClaScientificName`). Without expansion, only the IRN of the linked record is exported.
3. To reuse a field set, **Save** the list (right pane), or **Load** a previously saved one.

### 8. Configure delimiter and quoting options (CSV / Tab only)

On the **Options** tab:

- **Field delimiter** — comma or tab.
- **Text qualifier** — `"` (double quote) is standard.
- **Include column headers** — enable so the first row carries field names.
- **Multi-value separator** — the character used to join values inside a `_tab` field (e.g., `;`). Pick something that does not occur in your data.
- **Line terminator** — match the consumer system (LF for Unix pipelines, CRLF for Windows-bound deliverables).

### 9. Execute the export

Click **OK** (or **Export**). EMu shows progress in the status bar; large result sets may surface a progress dialog. Do not close the module or change the result set until completion.

### 10. Verify the output

Open the output file and check:

- Row count equals the result set count (subtract 1 for the header row in CSV/Tab).
- Diacritics render correctly — confirms UTF-8 was honoured end to end.
- `_tab` fields contain the expected multi-value separator and no truncation.
- Reference fields contain resolved values, not bare IRNs (unless that was intentional).
- For XML, validate against the EMu XML schema (`xmllint --noout`) before downstream ingest.

### 11. (Optional) Persist the configuration

For repeatable exports:

- In the Export dialog, save the field list.
- **Tools > Searches > Save** to persist the query.
- Document the format, encoding, delimiter choices, and field list alongside the saved search so the export is reproducible by other staff.

## Server-side alternative: `texexport`

For large, scheduled, or pipeline-driven exports, run `texexport` directly against the `etaxonomy` table on the EMu server. This bypasses the GUI and is the right tool for automated workflows. It requires shell access to the server, knowledge of Texpress query syntax, and coordination with the registry administrator. The output formats and field-selection semantics map closely to the GUI dialog.

A typical invocation looks like:

```sh
texexport -t etaxonomy -f csv -o /exports/etaxonomy_$(date +%Y%m%d).csv \
  -F field_list.txt -q 'ClaKingdom=Animalia'
```

Field lists and queries should be version-controlled with the rest of your ETL.

## Notes and gotchas

- **Reference fields export IRNs by default.** If you need names rather than internal numeric keys, expand the reference and select the relevant sub-fields explicitly.
- **Hierarchy fields are stored values, not traversed.** `ClaKingdom`…`ClaSpecies` reflect what was entered on the record. To get a hierarchy derived by walking parent references, either populate `ClaHigherClassification_tab` via a registry job or post-process from `ClaParentRef` chains.
- **Field-level security is honoured.** Restricted fields appear blank in the output without warning. If counts look right but values are missing, check the user's record- and field-level access.
- **Synonyms and common names** sit on attached records or in `_tab` fields depending on local configuration. Confirm the registry's modelling before assuming a single field carries the data you need.
- **Attachments (multimedia, citations)** are not embedded in CSV/Tab exports. XML exports include attachment payloads only when those fields are explicitly selected.
- **Long-running exports** can be interrupted by client timeouts on slow links. Run heavy exports from a workstation on the same network segment as the EMu server, or use `texexport` server-side.
- **Verify against the Axiell documentation** for your specific 9.0 build — minor UI labels and menu paths can differ between point releases and locally customised registries.
