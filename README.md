# Equilibrium Prototype Knowledge Graph

An editable version of the knowledge graph from RGC proposal 27628327, *Form-Finding Based on Equilibrium Prototypes*. It is drawn in the notation of the proposal's Figure 3:

- **Arch-cable** is the core prototype, and **Triangular Arch** and **Triangular Cable** are the original prototypes that combine into it.
- Eight adjusted prototypes open into **32 In-Between States**.
- Each In-Between State is materialised as one representative **built case**. All 78 collected cases are included, either as the representative or as an alternate that can be swapped in.

**Open it:** https://gabrielle-guo.github.io/knowledge-graph/   The address works once GitHub Pages is switched on for this repository.

## What's here

| Path | What it is |
|---|---|
| `index.html` | The graph editor: drag circles, change links and commands, swap cases, export PNG, SVG or JSON |
| `knowledge-graph.html` | A blank knowledge graph to build yourself, with a drawing board for each circle's form and force diagram |
| `photos/sources.json` | Where each case's photo comes from: the Miro board's chosen Commons file, or a Commons search |
| `scripts/fetch_photos.py` | Downloads the photos (run by GitHub Actions) |
| `photos/manifest.json` | Written by the script; the editor reads it to fill the photo rings |
| `CREDITS.md` | Author, licence and source page of every photo |
| `lab/index.html` | The Equilibrium Case Lab: form and force diagrams for up to ten cases, equilibrium checks, and the rules that link them |
| `lab/cases.json` | The cases, drawings and rules the lab opens with |

## Reviewing the graph

- Click any circle or link to see its details.
- Photos picked by *Wikimedia Commons search* in `CREDITS.md` were chosen automatically, so check them before use.
- To change a photo, edit its entry in `photos/sources.json` (a Commons file name or a search). The **Fetch case photos** action then runs again.
- Edits made in the browser stay in that browser. Use **Export → JSON** to share a changed layout.

## Equilibrium Case Lab

**Open it:** https://gabrielle-guo.github.io/knowledge-graph/lab/

Each case runs along the same chain as the graph: **2D prototype → first-step transformation → In-Between State → Materialize → built case**. At every stage you can draw a form diagram and a force diagram, with blue for compression, red for tension and green for loads, or use an image instead.

- **Check equilibrium** solves the force in every member from the loads and supports, and flags any member whose colour disagrees with the result.
- **Transformation commands** (mirror, scale, rotate with the loads, array) apply to a whole drawing and record themselves as that stage's command.
- **Compare and rules** lines the cases up, lists what they share, and keeps the rules you are testing.

The working copy is the Claude version of the lab, which saves every change. To update this public copy, use **Export → JSON** there and upload the file here as `lab/cases.json`, replacing the old one.

## Blank knowledge graph

**Open it:** https://gabrielle-guo.github.io/knowledge-graph/knowledge-graph.html

The same buttons as the graph editor, starting with no circles. Add prototypes, In-Between States and built cases, connect them, and add photos. Select any circle and click **Draw** to draft its form and force diagram, with blue for compression, red for tension and green for loads, then check its equilibrium. To share a version here, use **Submit to GitHub**; it is saved as `knowledge-graph.json`.
