# texexport — etaxonomy Full Module Export

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

## Export Commands

Full export of the entire `etaxonomy` module in fidsbrief format to `/tmp`:

```bash
/export/home/emu/texpress/11.0/bin/texexport -fidsbrief -a etaxonomy > /tmp/dumped.etaxonomy
```

Full export to `/var/tmp` (as used by `fidsdumper.rex`):

```bash
/export/home/emu/texpress/11.0/bin/texexport -fidsbrief -a etaxonomy > /var/tmp/dumped.etaxonomy
```

## Notes

- `-fidsbrief` — standard fidsbrief format used across all EMu dump scripts
- `-a etaxonomy` — exports all records from the `etaxonomy` module
- After writing to `/tmp`, the file is typically copied to `/ypm/webwork/dumped.etaxonomy` before `webbuild-tax.rex` is run
