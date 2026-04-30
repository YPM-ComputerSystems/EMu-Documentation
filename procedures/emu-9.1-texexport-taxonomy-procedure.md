# Exporting All Taxonomy Data with `texexport` from EMu 9.1

A stepwise procedure for a full-table, full-schema export of `etaxonomy` from the EMu server using the Texpress `texexport` utility. This is the recommended path for backup, migration, and bulk transformation pipelines on any non-trivial registry.

`texexport` is faster than the GUI, scriptable, and operates below EMu's permission layer — it produces a true full-schema dump, not a security-filtered view. That is a feature for migration work and a footgun for routine reporting; treat the output accordingly.

## Prerequisites

- Shell access to the EMu server hosting the target registry
- Permission to assume the OS user that owns the registry data (typically `emu` or a service account — confirm locally)
- Sufficient disk on the output volume — plan for roughly the on-disk size of the `etaxonomy` table, with 50% headroom; XML output is verbose but compresses 10:1 or better
- The registry's data path (commonly `…/<env>/data/etaxonomy`, but installation-specific)
- A maintenance window for production registries
- Familiarity with the local `texexport` build — flag spellings have drifted across Texpress releases. Verify with `man texexport` before scripting.

## Step 1 — Coordinate and announce

Notify the registry administrator and any active integrators. A full export does not block writes, but it ties up I/O bandwidth and creates a long-lived read position that interacts poorly with reorganisation jobs. For production, announce the window in the team channel.

## Step 2 — SSH to the EMu server and become the registry user

Authenticate as your normal user, then switch to the EMu service account. The export must run as a user with read access to the table data files; running as your own user typically fails on permissions.

```sh
ssh emu-server.example.org
sudo -iu emu
```

## Step 3 — Source the EMu environment

The Texpress utilities depend on `TEXAPI`, `TEXCONFIG`, `LD_LIBRARY_PATH`, and `PATH` being set. These come from the registry's environment script — exact path is install-specific.

```sh
. /usr/local/ke/etc/<env>.profile

which texexport
texexport -V
```

If `texexport` does not resolve or does not print a version, the environment is not set correctly. Stop and fix before continuing.

## Step 4 — Locate the `etaxonomy` table

```sh
echo "$TEXCONFIG"
ls "$TEXCONFIG"/../data/etaxonomy
```

Confirm the directory exists and contains the expected Texpress files. Adjust the path if your install uses a different layout.

## Step 5 — Capture the source record count

Capture the count from the source before exporting; you will need it for verification.

```sh
texquery -t "$TEXCONFIG"/../data/etaxonomy -c '*' | wc -l
```

The exact `texquery` invocation varies by version — the point is to get a verifiable number, not the specific command. Save it: `echo <count> > "$OUTDIR/source_count.txt"` once `OUTDIR` is set in the next step.

## Step 6 — Prepare the output location

Create a dated output directory on a volume with sufficient free space. **Do not** target the same filesystem as the live registry data — concurrent heavy I/O on the same spindle slows both the export and live operations.

```sh
OUTDIR=/exports/etaxonomy/$(date +%Y%m%d)
mkdir -p "$OUTDIR"
df -h "$OUTDIR"
```

## Step 7 — Run the export

For a full-schema dump, use KE EMu XML. With no query and no field list, `texexport` defaults to **all records** and **all fields** — which is what is wanted here.

Foreground (small registries, attended runs):

```sh
texexport \
  -f xml \
  -o "$OUTDIR/etaxonomy.xml" \
  -t "$TEXCONFIG"/../data/etaxonomy \
  2> "$OUTDIR/texexport.stderr.log"
```

Backgrounded under `nohup` (large registries, unattended runs):

```sh
nohup texexport \
  -f xml \
  -o "$OUTDIR/etaxonomy.xml" \
  -t "$TEXCONFIG"/../data/etaxonomy \
  >  "$OUTDIR/texexport.stdout.log" \
  2> "$OUTDIR/texexport.stderr.log" &
echo $! > "$OUTDIR/texexport.pid"
```

Verify the flags against `man texexport` on your system. The conceptual arguments — format, output path, table — are stable; flag spellings (`-f` vs `--format`, `-t` vs `-T`, accepted format names) have not been.

## Step 8 — Monitor progress

For backgrounded runs:

```sh
tail -f "$OUTDIR/texexport.stderr.log"
ps -p "$(cat "$OUTDIR/texexport.pid")"
ls -lh "$OUTDIR/etaxonomy.xml"   # confirm output is growing
```

For foreground runs, hold the session under `tmux` or `screen` so a dropped SSH connection does not kill the export.

## Step 9 — Verify the output

After the process exits cleanly:

```sh
# 1. Well-formed XML
xmllint --noout "$OUTDIR/etaxonomy.xml"

# 2. Record count matches source
xmlstarlet sel -t -v "count(/table/tuple)" "$OUTDIR/etaxonomy.xml"
# Adjust the XPath to match your registry's XML schema:
# the root and record element names depend on the local config.

# 3. Checksum the artefact
sha256sum "$OUTDIR/etaxonomy.xml" > "$OUTDIR/etaxonomy.xml.sha256"
```

Compare the count from step 9 against the source count from step 5. They should match exactly. Spot-check a few records — including ones with deeply nested `_tab` fields and multiple attached references — against the GUI for fidelity.

## Step 10 — Compress and transfer

XML compresses extremely well.

```sh
zstd -19 "$OUTDIR/etaxonomy.xml" -o "$OUTDIR/etaxonomy.xml.zst"
sha256sum "$OUTDIR/etaxonomy.xml.zst" >> "$OUTDIR/etaxonomy.xml.sha256"
```

Transfer to the destination via `rsync` or `scp`. Re-checksum at the destination and compare. Do not delete the source artefact on the server until destination integrity is confirmed.

## Step 11 — Document and clean up

Record in a run log:

- Operator, date, environment (production / training / test)
- Source data path and record count
- `texexport` version and exact command line
- Output path, size, and SHA-256
- Destination of the transferred copy

Once the export is verified and archived in its destination, remove the working copy from the EMu server in line with retention policy. Full exports are sensitive artefacts; handle them with the same discipline as a database backup, including access logging and at-rest encryption where required.

## Notes

- **References stay as IRNs.** `texexport` does not resolve attached references, and you do not want it to — circular structures (parent ↔ currently accepted name, etc.) make resolution-on-export a footgun. Resolve downstream where the transform is reproducible and inspectable.
- **Multimedia is separate.** Taxonomy records reference media in `emultimedia`; the binaries themselves require a parallel `emultimedia` export.
- **Subset exports.** To export a filtered subset, add a `-q` (or local equivalent) Texpress query argument. Document the filter alongside the output so the export is reproducible.
- **Format alternatives.** CSV and tab-delimited outputs are available but lossy for `_tab` fields and references. They are appropriate for narrow projections, not full-schema dumps.
- **Permission layer.** `texexport` bypasses EMu's record- and field-level security. This is correct for migration-grade dumps and unsafe for ad hoc reporting; the artefact must be handled accordingly.
- **Verify against the Axiell / Texpress documentation for your specific 9.1 build.** Flag names, default format identifiers, and environment-script paths can differ between point releases and locally customised installs.
