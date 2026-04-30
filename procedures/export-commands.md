# EMu Export Commands
20260430

Commands run from the command line (bash) to export data from EMu.
See texexport-etaxonomy.md for more detailed information including additional commands and advanced exports

$ mkdir /var/tmp/20260430

Exported entire modules in a delimited format (one record/row) with empty rows (fields) and rows supressed 
$ /export/home/emu/texpress/11.0/bin/texexport -fdelimited -md -ms -a ecatalogue > /var/tmp/20260430/dumped.ecatalogue-delimited 2> /var/tmp/20260430/texexport-ecatalogue.err
$ /export/home/emu/texpress/11.0/bin/texexport -fdelimited -md -ms -a ecollectionevents > /var/tmp/20260430/dumped.ecollectionevents-delimited 2> /var/tmp/20260430/texexport-ecollectionevents.err
$ /export/home/emu/texpress/11.0/bin/texexport -fdelimited -md -ms -a etaxonomy > /var/tmp/20260430/dumped.etaxonomy-delimited 2> /var/tmp/20260430/texexport-etaxonomy.err
$ /export/home/emu/texpress/11.0/bin/texexport -fdelimited -md -ms -a eparties > /var/tmp/20260430/dumped.eparties-delimited 2> /var/tmp/20260430/texexport-eparties.err
