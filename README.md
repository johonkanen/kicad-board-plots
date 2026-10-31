# kicad-board-plots

A reusable GitHub Actions workflow that plots every copper layer of a KiCad board and shows the plots in the
project's README. Signal layers and planes are plotted flat (copper and board outline, no silkscreen), so inner
layers are visible too.

On each board change it:

1. Plots each copper layer with `kicad-cli pcb export svg` in the official `kicad/kicad` Docker image. Every layer,
   B.Cu too, is seen from the top (not mirrored), so the layers line up with each other.
2. Force-pushes the SVGs as a single commit to the `renders` branch, which only ever holds the latest set. The plots
   never enter the main branch's history.
3. Writes a table of the plots into `README.md` between `<!-- board-plots:start -->` and `<!-- board-plots:end -->`,
   with captions from the board's layer stack and zones (e.g. `In3.Cu: 1V1, VCORE, GND planes`). Without the
   markers, a `## Board` section is appended. The README is only committed when the table text changes, e.g. when a
   layer or plane net is added.

## Use

Add `.github/workflows/board-plots.yml` to the hardware repository:

```yaml
name: Board plots
on:
  push:
    branches: [main]
    paths: ['**.kicad_pcb', '.github/workflows/board-plots.yml']
  workflow_dispatch:
jobs:
  plots:
    uses: johonkanen/kicad-board-plots/.github/workflows/plots.yml@v1
    permissions:
      contents: write
```

Push it; the first run creates the `renders` branch and the README section. The repository must be public: the
README loads the images from `raw.githubusercontent.com`, which needs no login only for public repositories.

## Options

Pass them under `with:` in the calling job.

| Input | Default | |
|---|---|---|
| `boards` | `**/*.kicad_pcb` | board files or bash globs, space-separated |
| `kicad-version` | `10.0` | tag of the `kicad/kicad` image; use the KiCad version the board was saved with or newer |
| `branch` | `renders` | branch the plots are pushed to |
| `readme` | `README.md` | README to write the table into; `''` to skip |
| `image-width` | `400` | width of each image in the table, in pixels |
| `render-3d` | `false` | also render the assembled board in 3D, top and bottom (since `v1.1`) |

With `render-3d: true`, a parallel job raytraces the board with `kicad-cli pcb render` in the
`kicad/kicad:<kicad-version>-full` image, which has KiCad's 3D models (the plain image has none). The two PNGs go
to the same branch and lead the table. Footprints whose 3D model is missing or outside KiCad's library show as bare
pads.

Several boards get one table each, under a `### <board>` heading. Boards are told apart by file name, so two boards
with the same file name in different folders would overwrite each other's plots.

After an update, GitHub may show the previous images for a few minutes (image caching).
