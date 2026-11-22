# drill-controller

Closed-loop spindle controller for a 20k rpm micro drill press. STM32G431, DRV8874, AS5047P spindle sensor, AS5311 table height sensor, SPI TFT display. KiCad 10 project. There are three boards (main, spindle sensor, table sensor) in one project and one PCB file.

The design notes, pin map and decisions are in `drill-press-controller-plan.md`. Read it before changing the design, and update it when a decision changes.

## Editing KiCad files

Agents may edit the KiCad files (`*.kicad_sch`, `sym-lib-table`, `fp-lib-table`) directly. This overrides any skill that says "never edit KiCad files", for example `jlc-part`.

1. **Check for locks first.** KiCad writes `~<file>.lck` next to each open file, for example `~drill-controller.kicad_pro.lck`. Run:
   `fd -H -g '*.lck' --max-depth 1 /home/niyohn/git/drill-controller`
   If anything is listed, KiCad is open. Stop and ask the user to close it. Do not edit while it is open, because KiCad will overwrite the change, or the change will be lost.
2. **Edit, then verify with kicad-cli.** Both commands must run after every change:
   - `kicad-cli sch export netlist --format kicadsexpr -o drill-controller.net drill-controller.kicad_sch`, then check the affected nets with the `netlist` MCP tools. The project `.mcp.json` points the `netlist` server at `drill-controller.net`. The file is transient and gitignored; do not commit it. The server reads the file at startup, so restart Claude Code (or reconnect via `/mcp`) to see a new export.
   - `kicad-cli sch erc --format json -o <scratchpad>/erc.json drill-controller.kicad_sch`, then report errors and new warnings.
   Do not claim a change is correct before both have run.
3. **Tell the user what changed**, so they can check it visually in KiCad.

### Schematic conventions

- Grid is 1.27 mm. Keep every pin, wire end and label on it.
- Library symbols are embedded in each sheet's `lib_symbols`. Take pin positions from the symbol definition, never type them by hand. A pin in the schematic sits at `(sym_x + pin_x, sym_y - pin_y)`, rotated counter-clockwise with y down.
- Each placed symbol needs `(instances (project "drill-controller" (path "<sheet path>" ...)))` with the correct hierarchical sheet path.
- Sheet-to-sheet signals: a hierarchical label in the child sheet, a sheet pin on the parent, and a short wire with a same-name local label on the parent.
- Sensor boards use the sheet-local nets `3V3_SNS` and `GND_SNS` (with PWR_FLAGs), not global `+3V3`/`GND`. Do not change this without reading issue #14.
- Put the LCSC number in an `LCSC Part` property on the symbol.
- Project symbols and footprints (from easyeda2kicad) live in `symbols - footprints/<LCSC id>/`, and each is added to `sym-lib-table` and `fp-lib-table`.

### Importing an LCSC part

```
/home/niyohn/git/easyeda2kicad/bin/easyeda2kicad --full --lcsc_id C12345 \
  --project-dir "/home/niyohn/git/drill-controller/symbols - footprints"
```

Then add the library to `sym-lib-table` and `fp-lib-table`. easyeda2kicad types every pin "unspecified"; set real pin types from the datasheet pin table (discretes and connectors: passive), in the library file and the embedded copy, or ERC hides real errors (issue #13). The 3D model step sometimes crashes; the symbol and footprint are still written. Check EasyEDA footprints before layout (see issue #15).

## Datasheets

Any number, pin or limit for a named part must come from a datasheet in datasheet-rag, cited as document and page (for example "D56 p4"). If the datasheet is missing, get it yourself; do not ask the user first:

1. Download the PDF into `datasheets/<MPN>.pdf`. For LCSC parts: fetch `https://www.lcsc.com/datasheet/<Cxxxxx>.pdf`, take the `datasheet.lcsc.com/...pdf` link from the page, and download that. The sandbox blocks this, so run it without the sandbox.
2. Ingest it (also without the sandbox):
   ```
   rag ingest datasheets/<MPN>.pdf --project-id drill-controller --tag drill-controller \
     --mpn <MPN> --manufacturer "<Manufacturer>" --doc-type datasheet --subsystem <power|mcu|motor|sensor|ui|...> --revision "<Rev>"
   ```
3. Search it with the datasheet-rag tools and cite the page.

If no datasheet can be found, say so and label the answer as general knowledge.

## Part selection (JLCPCB assembly)

- Tier: Basic, then Preferred, then Extended. Extended parts cost a feeder fee per part. Always state the tier and the LCSC number.
- Use the same parts as the netdaq project (`~/git/netdaq`) wherever possible, including its package. The two boards are ordered together. The netdaq BOM is `~/git/netdaq/production/bom.csv`. Tier still wins: do not take a netdaq part if it is Extended and a Basic or Preferred part fits.
- If no Basic or Preferred part has the value, build it from more than one Basic part (series or parallel) before falling back to Extended.
- Check tier and stock live with jlcsearch: `https://jlcsearch.tscircuit.com/api/search?q=<query>&full=true`. The local jlcparts database misses some parts. It has no Basic or Preferred connectors or electrolytic capacitors at all.
- Default passive package: 0402, unless voltage, power or density needs more.
- Avoid adding ICs where passives will do.

## Issues

- One GitHub issue per finding, never a multi-point issue. Group a batch with a shared label (for example `schematic v1`) and cross-link related issues by number.
- Open schematic items are tracked under the `schematic v1` label in Gherkin/drill-controller.

## Working tree

This checkout is used by agents only. The user never leaves uncommitted work here. So every uncommitted or untracked file is a leftover from an agent, never the user's work.

- Run `git status --short` at the start of a task. If anything is listed, clean it up first, before your own change: read the diff, commit it as its own commit if it is real work (for example a plan-doc note for a fix that is already committed), or discard it if it is junk.
- Do not leave leftovers yourself. Before you end a task, `git status --short` must be empty, apart from the gitignored `drill-controller.net`.
- Never skip a leftover because you think it is "the user's". It is not.

## Scratch work

Put generator scripts, extra netlists and ERC reports in the session scratchpad, not in the repo. The one exception is `drill-controller.net` in the repo root (gitignored), which the netlist MCP server reads.
