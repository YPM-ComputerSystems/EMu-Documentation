# texexport — EMu Module Export Reference

## Binary Location

```
/export/home/emu/texpress/11.0/bin/texexport
```

## etaxonomy Data Directory

| Context | Path |
|---|---|
| Raw dump output (webdumper / general use) | `/tmp/dumped.etaxonomy` |
| Web pipeline input (read by `webbuild-tax.rex`) | `/ypm/webwork/dumped.etaxonomy` |
| Web pipeline output (processed by `webbuild-tax.rex`) | `/ypm/webwork/webinput.tax` |
| General fidsbrief dumps (`fidsdumper.rex`) | `/var/tmp/fidsdump.etaxonomy` |

---

## etaxonomy Export

Full export in fidsbrief format to a timestamped directory, with error log:

```bash
OUTDIR="/tmp/etaxonomy-$(date +%Y%m%d-%H%M%S)" && \
mkdir -p "$OUTDIR" && \
/export/home/emu/texpress/11.0/bin/texexport -fidsbrief -a etaxonomy \
  > "$OUTDIR/dumped.etaxonomy" \
  2> "$OUTDIR/texexport-etaxonomy.err"
```

Full export to `/var/tmp`:

```bash
OUTDIR="/var/tmp/etaxonomy-$(date +%Y%m%d-%H%M%S)" && \
mkdir -p "$OUTDIR" && \
/export/home/emu/texpress/11.0/bin/texexport -fidsbrief -a etaxonomy \
  > "$OUTDIR/dumped.etaxonomy" \
  2> "$OUTDIR/texexport-etaxonomy.err"
```

| File | Contents |
|---|---|
| `$OUTDIR/dumped.etaxonomy` | fidsbrief export data |
| `$OUTDIR/texexport-etaxonomy.err` | stderr / error log |

---

## ecatalogue Export — Suppress Blank Columns and Rows

Export only fields and records that contain data (`-md` drops empty fields, `-ms` drops empty rows):

### fidsbrief format

```bash
/export/home/emu/texpress/11.0/bin/texexport -fidsbrief -md -ms -a ecatalogue \
  > /var/tmp/etaxonomy-20260430/dumped.ecatalogue \
  2> /var/tmp/etaxonomy-20260430/texexport-ecatalogue.err
```

### Delimited format

```bash
/export/home/emu/texpress/11.0/bin/texexport -fdelimited -md -ms -a ecatalogue \
  > /var/tmp/etaxonomy-20260430/dumped.ecatalogue \
  2> /var/tmp/etaxonomy-20260430/texexport-ecatalogue.err
```

### Flag Reference

| Flag | Effect |
|---|---|
| `-fidsbrief` | fidsbrief format — one `field=value` per line, records separated by `###` |
| `-fdelimited` | delimited format — one row per record, tab-separated |
| `-md` | suppress empty/null fields — omits any field with no data |
| `-ms` | suppress empty rows — omits records with no populated data |
| `-a` | export all records in the module |
| `2>` | redirect stderr to a separate error log file |

---

## Detailed Logging

### Basic — stderr and stdout to separate files

```bash
OUTDIR="/var/tmp/ecatalogue-$(date +%Y%m%d-%H%M%S)" && \
mkdir -p "$OUTDIR" && \
/export/home/emu/texpress/11.0/bin/texexport -fidsbrief -md -ms -a ecatalogue \
  > "$OUTDIR/dumped.ecatalogue" \
  2> "$OUTDIR/texexport.err"
```

### Detailed — timestamped log with start/end and duration

```bash
OUTDIR="/var/tmp/ecatalogue-$(date +%Y%m%d-%H%M%S)" && \
mkdir -p "$OUTDIR" && \
LOGFILE="$OUTDIR/texexport.log" && \
echo "=== texexport ecatalogue ===" >> "$LOGFILE" && \
echo "Started: $(date)"            >> "$LOGFILE" && \
echo "Output:  $OUTDIR/dumped.ecatalogue" >> "$LOGFILE" && \
echo " "                           >> "$LOGFILE" && \
time /export/home/emu/texpress/11.0/bin/texexport -fidsbrief -md -ms -a ecatalogue \
  > "$OUTDIR/dumped.ecatalogue" \
  2>> "$LOGFILE" && \
echo " "                           >> "$LOGFILE" && \
echo "Finished: $(date)"           >> "$LOGFILE" && \
echo "Lines exported: $(wc -l < "$OUTDIR/dumped.ecatalogue")" >> "$LOGFILE"
```

### With `tee` — view output live and save log simultaneously

```bash
OUTDIR="/var/tmp/ecatalogue-$(date +%Y%m%d-%H%M%S)" && \
mkdir -p "$OUTDIR" && \
/export/home/emu/texpress/11.0/bin/texexport -fidsbrief -md -ms -a ecatalogue \
  > "$OUTDIR/dumped.ecatalogue" \
  2>&1 | tee "$OUTDIR/texexport.log"
```

### Log output file summary

| File | Contents |
|---|---|
| `dumped.ecatalogue` | fidsbrief export data |
| `texexport.log` | timestamped run log with start, finish, errors, line count |
| `texexport.err` | stderr only (errors/warnings from texexport) |

---

## Notes

- After exporting `etaxonomy`, copy `$OUTDIR/dumped.etaxonomy` to `/ypm/webwork/dumped.etaxonomy` before running `webbuild-tax.rex`
- The EMu Texpress database for each module lives at `/export/home/emu/ypmnh/server/data/<module>/`
- `texexport` reads directly from the live database — the `/data/` files are never accessed directly
