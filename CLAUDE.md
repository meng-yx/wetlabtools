# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fork of `ngoldb/wetlabtools` (Nicolas Goldbach). Only `wetlabtools/cloning/tecanfluent.py` is
actively modified here; the other subpackages (`spark`, `tecan`, `chirascan`, `plot`, `protein`)
are upstream's and carry their own readmes.

`cloning.tecanfluent.make_cloning_worklists_from_twist()` turns a plate map into the files a
Tecan Fluent method consumes. There is no test suite — correctness is established by
regenerating a known run and diffing (see **Verifying a change**).

## The FluentControl contract — read before changing anything

The generated files are **inputs to a FluentControl method that validates them**. The method is
not in this repo and cannot be read from here, so these constraints are invisible from the code
alone. Breaking one aborts the run on the instrument, not in Python.

**`<barcode>_reactions.csv` is robot-facing, not a report.** The method does:

```
Set Variable     TwistID = GetAttribute("TwistPlate[001]", "barcode")
Import Variable  20 variables from ...\LPDI_Twist_worklists\<TwistID>\<TwistID>_reactions.csv
If               worklisting_version <> 0.4
Raise Error      NonRecoverable: "Incompatible Version. Update worklist creation script"
```

- `worklisting_version` is gated by **exact equality**. Changing it aborts every run at startup,
  right after the Twist plate barcode is read. It describes the *file format* the method
  consumes, not the behaviour of this package — do not treat it as a changelog. It moves only
  together with the method's `If` condition.
- The method imports **exactly 20 variables**, so the column set and count are frozen:
  4 counts + 7 vectors + 8 strains + the version.

**Names and locations are fixed.** `TwistID` is read from the physical plate barcode, so the
output directory `<gwl_location>/<Plate ID>/` and the `<Plate ID>_*` filename prefix are what the
method resolves paths with. Do not rename generated files.

**Labware labels are worktable positions, not identities.** Every label written into a `.gwl` —
`TwistPlate[001]`, `96 Well PCR GGA`, `96 Well PCR Gibson`, `CUSTOM_VECTOR_1..3`,
`96 Well PCR Transform 1`/`2`, and the strain names — must exist on the method's worktable and be
reachable by the pipetting arm. `ALLOWED_VECTORS` and `TRANSFORMATION_STRAINS` are closed sets for
that reason. A slot carries no plasmid or strain identity; that mapping is per-run and lives in
the experiment's own notebook, nowhere here.

**What the `.gwl` does and does not control.** It is an itinerary: labware label, position index,
volume, and optionally a liquid class *name*. Everything else is the method's — labware geometry
(including whether Transform 2 is a deep-well plate, which only the `cloning/readme.md` states as
intent), liquid class definitions, arm and tip selection, wash stations, incubation, plate moves
and control flow. A `Load Worklist` step supplies defaults for arm, liquid class, tips and wash;
a liquid class named in the file overrides that default. Most records here leave it empty.

Record layouts, as written and parsed by this package:

```
A/D   11 fields   A;RackLabel;RackID;RackType;Position;TubeID;Volume;LiquidClass;TipType;TipMask;ForcedRackType
R     16+ fields  R;SrcRackLabel;SrcRackID;SrcRackType;SrcPosStart;SrcPosEnd;
                    DestRackLabel;DestRackID;DestRackType;DestPosStart;DestPosEnd;
                    Volume;LiquidClass;NoOfDitiReuses;NoOfMultiDisp;Direction[;ExcludedDestWell...]
W     tip change / wash
```

Positions are **column-major** (`1=A1, 8=H1, 9=A2`). Field 17 onward of an `R` record is the
excluded-well list, which is how a reagent reaches a subset of a contiguous range.

**The master mix is never written.** `gga_settings["mm_gwl_file"]` and `mm_volume` exist, but
`make_cloning_worklists` only uses `mm_gwl_file` to derive the output directory for a print.
No `*_mm.gwl` is ever produced — the method dispenses master mix itself.

## Plate layout is a function of the data

`_sort_reactions()` orders reactions by `(src_numerical, Vector)` with a stable sort, and
`ALLOWED_VECTORS` supplies the canonical vector order.

This matters because one source well can feed several reactions — the same insert cloned into two
vectors — which makes `src_numerical` alone a tied key. Those ties were previously broken by
pandas' default unstable quicksort, and the two sorts (in `make_cloning_worklists` and on
`cloning_df`) broke them differently, so the assembly and transformation plates could disagree
about which vector sat in which well. On one real 46-reaction plate that silently swapped 8 of 46
wells, confirmed by Sanger sequencing. Keep both the explicit `Vector` key and `kind='mergesort'`:
the sort must not depend on the input plate map's row order.

A transformation plate is re-indexed over only the wells being transformed, so **its layout need
not match the assembly plate**. That is by design (partial transformation, or a mixed GGA/Gibson
run drawing from two assembly plates). The transformation plate maps therefore carry `Vector`,
`Source Plate` and `Source Well`, so a transformed well stays traceable when the layouts diverge.

## Plate map sheets

`Plate.to_plate_map_dataframes()` defaults its `properties` argument to a `set()`, so sheet order
varies between processes. Pass an explicit ordered list to `to_plate_map_excel()` whenever writing
a plate map, or anything reading "the first sheet" becomes a coin flip.

## Verifying a change

There are no tests. Regenerate a known run and compare:

```python
import wetlabtools.cloning.tecanfluent as tf
tf.make_cloning_worklists_from_twist(plate_map, out_dir, cautios=False)
```

Then byte-compare the four `.gwl` files against the previous output. Changes that leave them
identical cannot affect the robot's pipetting; changes that alter them need justification.
Check `reactions.csv` separately — it is robot-facing and easy to forget.

Worth exercising more than the happy path: partial transformation (only some wells get a strain),
duplicated source wells across vectors, a mixed GGA/Gibson plate map, and a run using
`Transform_2`. These exercise the re-indexing and tie-breaking paths that ordinary single-vector
runs do not.

`cautios=True` (the default) refuses to overwrite an existing output directory; callers that
regenerate in place pass `cautios=False`. Note that regenerating never *deletes* files it no
longer produces — narrowing a run leaves stale plate maps behind that describe plates which are
no longer part of it.
