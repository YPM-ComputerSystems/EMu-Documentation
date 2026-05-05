# `texexport` Command-Line Reference

Based on the authoritative KE Texpress documentation hosted by Axiell at
<https://emu.axiell.com/downloads/Texpress/texexport.pdf>. EMu's `etaxonomy` and
all other modules are Texpress tables, so this is the same utility EMu 9.0
ships. EMu installs may carry locally added export formats (configured via the
`~texpress/etc/export` file, see below) — those are extensions on top of what
follows, not changes to the base flags.

> **Important context.** `texexport` is not a standalone "dump a table" tool the
> way `mysqldump` is. It was designed to be invoked **by `texforms`** (the
> Texpress forms front-end) after the user has selected a record set and chosen
> an export format from the Report menu. The required `-o` argument is the
> *input* file of record offsets that `texforms` writes — it is not the output
> path. The exported data goes to **stdout**, redirected with `>` in the shell,
> or to a printer/file selected by `texforms` via the `+f` operator. This
> matters for any procedure that tries to call `texexport` directly.

## Synopsis

```
texexport [-ccollist] [-fstyle] [-ssortlist] [-mmodifiers] [-nr] -ofile dbname
```

Arguments are written with no space between flag and value (`-fdelimited`, not
`-f delimited`). Output is written to stdout. The database name (`dbname`) is
positional and required.

## Required arguments

### `-o<file>` — offsets file (input)

Path to a file containing internal pointers (offsets) to the records to export
within `dbname`. This file is **produced by `texforms`** when the user picks
**Export this match** or **Export all matches** from the Report menu. It is the
only mechanism by which `texexport` learns which subset of records to operate
on. There is no `-q` query argument and no "export everything" flag — the
record set is always defined by the offsets file.

### `dbname` — database name (positional, last)

The Texpress database (table) name, e.g. `etaxonomy`. Not a filesystem path —
the bare name. Texpress resolves it via its own configuration.

## Optional arguments

### `-c<collist>` — columns to export

Comma-separated list of column IDs, each optionally followed by `:fldno` to pick
a specific field within a multi-valued column.

```
-cClaScientificName,ClaAuthor,ClaKingdom
-caddr:1,addr:2,suburb,pc       # specific subfields of a multi-value column
```

When invoked from `texforms` via the `+c` operator, the column list is built
interactively by the user and substituted in. When invoked directly from a
shell, you must enumerate the columns explicitly.

> **What "all columns" means here.** The PDF does not document a "give me every
> column" shorthand. In the `texforms`-driven flow the user types `F10` in the
> column-picker to select all; that selection is then substituted via `+c`.
> From a direct shell invocation, the practical paths are: enumerate columns
> explicitly, drive it via `texforms`, or use `texpressdata` format (which dumps
> the underlying internal representation and does not respect `-c` the way the
> textual formats do).

### `-s<sortlist>` — sort columns

Same syntax as `-c`. Columns used for sorting need not be in the export
column list. Omitted means no sort.

### `-f<style>` — output format

Format identifier, no space. See the format table below for valid values and
their per-format modifiers.

### `-m<modifiers>` — format-specific modifiers

Style-specific options. The valid modifiers depend on `-f`. See the format
table.

### `-n` — notify on completion

Notify the user when the export finishes. Useful for backgrounded runs invoked
from `texforms`.

### `-r` — remove the offsets file on completion

Deletes the `-o` file when the export finishes. Set automatically by the
default `texforms` export commands so the temporary offsets file gets cleaned
up.

## Output formats (`-f`)

| Format | Description | Modifiers |
|---|---|---|
| `ids` | One field per line, `colid:fieldno=data`. Each record preceded by `rownum=nnn`, terminated by `###`. | `-ml`, `-mw<n>` |
| `idsbrief` | As `ids`, but lines for empty fields are omitted. | `-ml`, `-mw<n>` |
| `texpressdata` | KE Texpress internal "dave" data file format. Column order in output follows the order given in `-c`, which **may differ from the database's internal column order** — exercise caution; the resulting file can appear inconsistent if reloaded. | — |
| `texql` | KE Texql `INSERT` statements. Suitable for re-loading into another Texpress database. | — |
| `delimited` | Character-separated values. Default text delimiter `"`, default column separator tab. | `-md<ch>`, `-ms<ch>`, `-mc`, `-ml` |
| `fixedwidth` | Fixed-width, blank-padded columns. Width per column comes from the table's Insertion form definition. | — |
| `format` | Output controlled by an explicit format string. | `-mf<str>`, `-ml` |
| `formatbrief` | As `format`, but empty lines omitted. | `-mf<str>`, `-ml` |

### Modifier reference

- **`-ml`** — print Library (lookup-list) items as a single value rather than
  expanding them.
- **`-mw<n>`** — print column name in exactly `n` characters, right-padded with
  spaces. Applies to `ids` / `idsbrief`.
- **`-md<ch>`** — text qualifier character for `delimited`. Default `"`.
  Strings containing the qualifier are escaped by doubling.
- **`-ms<ch>`** — column separator character for `delimited`. Default tab. Use
  `-ms','` for CSV.
- **`-mc`** — for `delimited`, emit a header row of column names.
- **`-mf<str>`** — format string for `format` / `formatbrief`.

> **No `xml` in this reference.** The KE Texpress PDF documents no XML format
> for `texexport`. EMu's XML import/export workflow is typically driven through
> separate utilities and through the GUI's KE EMu XML export, not through
> `texexport -fxml`. If your local install accepts an `xml` style, it is a
> locally added format defined in `~texpress/etc/export` or
> `<dbdir>/export` — verify before relying on it.

## `texforms` substitution operators

These are not `texexport` flags — they are operators that appear in the
`~texpress/etc/export` configuration file and are interpreted by `texforms`
before it invokes `texexport`. Worth knowing because the example command lines
in the PDF and in `~texpress/etc/export` are written using them.

| Operator | Meaning |
|---|---|
| `+c` | Prompt the user to pick columns; substitute the selected list. |
| `+s` | Prompt the user to pick sort columns; substitute the selected list. |
| `+o` | Substitute the path of the temp offsets file `texforms` just wrote. |
| `+d` | Substitute the current database name. |
| `+f` | Prompt for output destination (file or printer); run in background. |
| `+i` | Run interactively in foreground (for export commands needing a TTY). |

## Worked examples (from the canonical `~texpress/etc/export`)

Comma-delimited, interactive column pick:

```
texexport -fdelimited -c+c -ms',' -s+s +f -r -o+o -n +d
```

Fixed-width:

```
texexport -ffixedwidth -c+c -s+s +f -r -o+o -n +d
```

`ids`:

```
texexport -fids -c+c -s+s +f -r -o+o -n +d
```

Texpress internal data:

```
texexport -ftexpressdata -c+c -s+s +f -r -o+o -n +d
```

A non-interactive mail-merge example with explicit columns and sort:

```
texexport -fdelimited -mc \
  -cfirstnam,surname,addr:1,addr:2,suburb,pc \
  -ssurname +i -r -o+o people > /tmp/mailmerge.txt
```

Note the `> /tmp/mailmerge.txt` — output goes to **stdout**, not to `-o`.

## Adding or inspecting export formats

Export formats live in two files, evaluated in this order:

1. `<dbdir>/export` — per-database, overrides global.
2. `~texpress/etc/export` — global to the Texpress install.

Each non-comment line is `label:command`. `label` is what shows up in the
format-picker; `command` is the shell command line, with the `+`-operators
above resolved at invocation time. To see what formats your install actually
offers — including any locally added ones such as XML or DwC — read these two
files.

---

## Implications for prior procedures (correction)

The procedures I wrote earlier in this conversation invoked `texexport` as a
standalone command-line dump tool. **Reading this reference, that framing was
wrong in three specific ways:**

1. **`-o` is the offsets *input* file, not the output path.** Output goes to
   stdout. My earlier examples that wrote `-o /exports/etaxonomy.xml` would
   either fail or be silently misinterpreted.
2. **There is no `-t /path/to/table` argument.** The table is the trailing
   positional `dbname`, resolved by Texpress configuration, not by filesystem
   path.
3. **There is no documented `xml` format in base Texpress and no "no query =
   all records" default.** The record set always comes from an offsets file
   `texforms` writes; there is no built-in "dump the whole table" mode driven
   from the command line alone.

Practically, this means a true full-table export from the shell goes through
one of:

- **Drive `texforms` from a script** so it produces the offsets file for "all
  matches" and then invokes `texexport` against it. This is how the registered
  export commands in `~texpress/etc/export` actually work.
- **Use a locally added export format** (check `~texpress/etc/export` and the
  per-database `export` file). Many EMu installs add XML and other formats here.
- **Use the EMu GUI's Tools > Export**, which orchestrates the offsets-file
  step internally.
- **Use `texdump`/`texrestore`** or other Texpress utilities for raw
  table-level dump/restore — these are the tools that actually correspond to
  "dump the whole table without a precomputed result set", though they produce
  Texpress-internal formats, not application-friendly XML/CSV.

I'd recommend confirming the exact mechanism your registry administrator uses
for full taxonomy dumps before scripting against it — and treating my earlier
procedures as needing revision rather than as ready to run.
