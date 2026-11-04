# Verilator Coverage Report & Explorer

A small, project-independent coverage viewer for raw Verilator `coverage.dat` files. It provides a browser-based dashboard for inspecting the coverage records that Verilator has already collected, plus command-line HTML and CSV report generation.

The same `cover_report` command can be used from different RTL, IP, SoC, and verification repositories. The viewer does not contain project-specific signal names, hierarchy paths, register maps, or protocol assumptions.

> **Current status:** usable as a general Verilator `.dat` report viewer, with FSM and functional-coverage views driven by the metadata present in the input file. Rich coverage-matrix views and more robust FSM analysis are follow-up work; see [Roadmap and additional work](#roadmap-and-additional-work).

## Contents

- [Background](#background)
- [What it supports](#what-it-supports)
- [What it does not do](#what-it-does-not-do)
- [Requirements](#requirements)
- [Installation](#installation)
- [Publishing the repository](#publishing-the-repository)
- [Verilator setup and coverage collection](#verilator-setup-and-coverage-collection)
- [Usage](#usage)
- [Using it across multiple projects](#using-it-across-multiple-projects)
- [FSM coverage troubleshooting](#fsm-coverage-troubleshooting)
- [Roadmap and additional work](#roadmap-and-additional-work)
- [Repository layout](#repository-layout)
- [Known limitations](#known-limitations)
- [References](#references)

## Background

Verilator can instrument a model with coverage counters and write the collected results into a raw `coverage.dat` file. The `verilator_coverage` utility can summarize, merge, filter, and annotate these records. This project adds a browser dashboard and convenient report exports around that data.

The viewer reads the raw Verilator coverage file directly. It does **not** need the RTL source tree, a project Makefile, an I2C/USB/AXI-specific environment, LCOV, or `genhtml` to parse and display the recorded points. Source filenames, line numbers, hierarchy strings, point names, bin labels, and hit counts are taken from the `.dat` metadata.

The tool separates two responsibilities:

1. **Coverage generation:** Verilator must be configured and the simulation must write a valid `coverage.dat`.
2. **Coverage review:** `cover_report` parses that file and presents the records it contains.

The viewer cannot add coverage points that were not instrumented or collected during simulation.

## What it supports

The parser recognizes the following Verilator coverage record types when they exist in the imported `.dat` file:

| Coverage record | Current viewer support |
|---|---|
| Line (`line`) | Summary and source-file/line-oriented records; expand a line to inspect its points. |
| Branch (`branch`) | Hit count, covered/uncovered status, filtering, and point exclusion. |
| Expression (`expr`) | Hit count, covered/uncovered status, filtering, and point exclusion. |
| Toggle (`toggle`) | Signal, bit index where available, `0->1` and `1->0` transition labels, hit count, and exclusion. |
| Native FSM state (`fsm_state`) | FSM state table, hit count and status, grouped using available hierarchy/file metadata. |
| Native FSM arc (`fsm_arc`) | FSM transition table; a graph is drawn only when transition labels contain a recognizable source-to-destination separator. |
| Functional coverage (`covergroup` / `funccov`) | Covergroup/coverpoint/bin records, bin hits, and covered/uncovered status. |
| Cross coverage | Cross-like bin combinations are displayed when the metadata identifies the cross and exposes separable dimensions. |
| Vector/value bins | Encoded vector literals are shown when they are present in the `.dat` metadata. |
| User/property (`user`) | User/property coverage records and their hit counts. |
| Unknown record types | Preserved as `other` coverage records where they are parsed as Verilator coverage records. |

### GUI features currently included

- Import a raw `.dat` by uploading it in the browser or by entering a path on the host running the server.
- Coverage summary cards by coverage type.
- Filters for source file, coverage type, and covered/uncovered/excluded status.
- Source-line records with expandable point details.
- FSM state and transition tables, with a best-effort graph from parseable arc labels.
- Functional covergroup/bin, cross-like combination, and vector-literal views.
- Exclude an individual point or all points on a selected source line/range.
- Undo a selected exclusion or clear all exclusions.
- Download a filtered copy of the `.dat`; the imported original file is not modified.
- Extract the current GUI report as CSV.

### Command-line report features

Without `-gui`, the command creates a standalone HTML report and a CSV report beside the input `.dat` file. This mode does not start the web server.

## What it does not do

- It does not run Verilator or execute a simulation.
- It does not instrument RTL or modify a simulator build.
- It does not infer an FSM from RTL source code or waveform activity.
- It does not create missing `fsm_state` or `fsm_arc` records.
- It does not reconstruct vector declarations, enum types, source text, or complete cross definitions when this information is absent from the `.dat` metadata.
- It is not currently a full cross-coverage Cartesian matrix or module/instance-by-coverage-type matrix tool. Current views are record tables and summaries; matrix work is tracked in the roadmap.
- It does not replace functional coverage planning, scoreboard checks, assertions, or verification sign-off.

## Requirements

- Linux is the primary supported environment for the current launcher and process-lifecycle management.
- Python **3.10 or newer**. The code uses Python 3.10 language syntax.
- A modern browser for GUI mode.
- No third-party Python packages are required; the server uses the Python standard library.
- Verilator and `verilator_coverage` are required only to generate or inspect coverage data, not to view an already-generated `.dat` file.

Check the Python installation:

```bash
python3 --version
```

## Installation

### Option A: clone the Git repository

Replace `<repository-url>` with the URL of the repository after you push it to Git hosting:

```bash
mkdir -p "$HOME/tools" "$HOME/bin"
git clone <repository-url> "$HOME/tools/general_verilator_coverage_explorer"
cd "$HOME/tools/general_verilator_coverage_explorer"
chmod +x cover_report verilator_coverage_explorer/server.py
ln -sfn "$HOME/tools/general_verilator_coverage_explorer/cover_report" "$HOME/bin/cover_report"
```

Add `$HOME/bin` to `PATH` (Bash):

```bash
grep -qxF 'export PATH="$HOME/bin:$PATH"' "$HOME/.bashrc" || \
  echo 'export PATH="$HOME/bin:$PATH"' >> "$HOME/.bashrc"
source "$HOME/.bashrc"
```

For Zsh, add the same `export PATH=...` line to `~/.zshrc` and reload it with `source ~/.zshrc`.

### Option B: install from a ZIP archive

Extract the release archive into a permanent tools directory, then create the same symlink:

```bash
mkdir -p "$HOME/tools" "$HOME/bin"
unzip <downloaded-archive>.zip -d "$HOME/tools"
chmod +x "$HOME/tools/general_verilator_coverage_explorer/cover_report"
ln -sfn "$HOME/tools/general_verilator_coverage_explorer/cover_report" "$HOME/bin/cover_report"
```

Ensure `$HOME/bin` is in `PATH` as shown above. Do not delete or move the extracted tool directory while the symlink is in use.

### Verify installation

```bash
command -v cover_report
cover_report --help
python3 -m py_compile \
  "$HOME/tools/general_verilator_coverage_explorer/cover_report" \
  "$HOME/tools/general_verilator_coverage_explorer/verilator_coverage_explorer/server.py"
```

If you installed by cloning to another directory, update the paths in the compile check accordingly.

Run a smoke test using the included synthetic coverage fixture:

```bash
cover_report "$HOME/tools/general_verilator_coverage_explorer/examples/sample_verilator_coverage.dat"
```

This should create a sample HTML report and CSV beside the fixture. The fixture includes representative line, branch, expression, toggle, FSM, covergroup/cross, vector-bin, and user records. It is a parser/UI test input, not coverage data from a real design.

## Publishing the repository

Before pushing, review the files that will be committed. Do not commit generated project coverage databases, imported files, or runtime state.

For a new repository:

```bash
cd "$HOME/tools/general_verilator_coverage_explorer"
git init
git add README.md .gitignore cover_report verilator_coverage_explorer/ examples/
git status --short
git commit -m "Add project-independent Verilator coverage explorer"
git branch -M main
git remote add origin <repository-url>
git push -u origin main
```

For an existing clone, skip `git init`, `git branch`, and `git remote add`; stage and commit the intended changes, then push the current branch:

```bash
git add README.md .gitignore
git status --short
git commit -m "Document Verilator coverage explorer"
git push
```

Keep `cover_report` and the `verilator_coverage_explorer/` directory together. The CLI resolves its server and UI files relative to its own installation path, so moving only the executable will break the installation.

## Verilator setup and coverage collection

`cover_report` consumes the `.dat` file; the following setup is performed in each design/project that you want to analyze.

### 1. Enable coverage during Verilation

Add coverage instrumentation to the existing Verilator build command and rebuild the model. The broad option is:

```bash
--coverage
```

Verilator documents this as enabling line, branch, expression, toggle, FSM, and user coverage. You can instead enable specific supported categories explicitly, for example:

```bash
--coverage-line \
--coverage-branch \
--coverage-expr \
--coverage-toggle \
--coverage-fsm \
--coverage-user
```

For native FSM state/arc counters in particular, make sure the model is Verilated with `--coverage` or `--coverage-fsm`. A clean rebuild is recommended after changing instrumentation options; then rerun the simulation to create a fresh coverage database.

For per-instance Verilator-inserted coverage counters, investigate `--coverage-per-instance` during Verilation. It must be selected when the model is built; the viewer cannot recover per-instance counters that were merged or not recorded.

### 2. Run the simulation and write the coverage file

For a model using Verilator's `--binary` or `--main` flow, the runtime coverage output path can be set using:

```bash
./obj_dir/V<top_module> \
  +verilator+coverage+file+coverage.dat
```

Replace `V<top_module>` with the executable generated by your build. With a custom C++ simulation wrapper, write the coverage data at the end of the run using the Verilator coverage API, for example:

```cpp
Verilated::threadContextp()->coveragep()->write("coverage.dat");
```

Use the method appropriate to your existing Verilator harness. The exact build command depends on your project, top module, file list, timing configuration, and C++/DPI setup.

### 3. Check the generated file

```bash
ls -lh /path/to/coverage.dat
head -5 /path/to/coverage.dat
```

The viewer expects a raw Verilator `.dat` file containing coverage records. An LCOV `.info` file is not accepted by this tool.

### 4. Optional: merge data from multiple runs

Run the simulations in separate output directories so each test writes its own coverage database. Merge compatible coverage files with Verilator before opening the result:

```bash
verilator_coverage --write merged_coverage.dat \
  run_001/coverage.dat \
  run_002/coverage.dat \
  run_003/coverage.dat
```

Use databases generated from the same compatible instrumented model/build. Then inspect `merged_coverage.dat` with `cover_report`.

## Usage

### Launch the GUI and choose a file explicitly

```bash
cover_report -gui /absolute/path/to/coverage.dat
```

This alternative argument order is also supported:

```bash
cover_report /absolute/path/to/coverage.dat -gui
```

### Launch the GUI and auto-discover a file

```bash
cd /path/to/your/project
cover_report -gui
```

When no path is supplied, the command checks `VERILATOR_COVERAGE_DAT`, common locations under the current directory, and then performs a shallow project-local search for `coverage.dat`. If multiple files are found, the most recently modified candidate is selected. If no file is found, the GUI opens and you can import a file from the browser.

You can make the default explicit with an environment variable:

```bash
export VERILATOR_COVERAGE_DAT="$HOME/work/project_a/sim/coverage.dat"
cover_report -gui
```

### Generate static HTML and CSV without a GUI

```bash
cover_report /absolute/path/to/coverage.dat
```

For an input named `coverage.dat`, the command creates:

```text
coverage_report.html
coverage_report.csv
```

The report files are written beside the input `.dat` file. This mode stops a previously launched `cover_report` GUI owned by the current user before generating the static reports.

### Import another file from the GUI

After launching the GUI, use either:

- **Upload & import** to upload a local file from the browser computer; or
- **Import server path** to provide a path that exists on the machine running `cover_report`.

The server has a configurable maximum upload size in `verilator_coverage_explorer/config.json` (500 MB in the supplied configuration).

### Remote Linux server usage

The HTTP server binds to `127.0.0.1`. When launched on a remote Linux host without a desktop, `cover_report` prints an SSH port-forwarding command. Run the printed command on your local computer, then open the forwarded `http://127.0.0.1:<port>` address in your local browser.

For example, using the port printed by the tool:

```bash
ssh -L <local-port>:127.0.0.1:<remote-port> <user>@<remote-host>
```

Keep the SSH session open while using the GUI.

### Server lifecycle and port handling

- GUI mode chooses an available localhost port automatically; no fixed port needs to be supplied.
- Starting a new `cover_report` invocation attempts to stop the previous explorer instance owned by the same user and matching the tool's recorded process identity.
- The launcher does not intentionally terminate unrelated processes or other users' servers.
- Closing the explorer browser tab requests server shutdown after a short grace period. A heartbeat timeout is also configured as a fallback if the browser disappears unexpectedly.
- Runtime state and logs are kept under `~/.cache/cover_report/`; imported files and exclusion working data are kept in the tool's configured workspace directory.

## FSM coverage troubleshooting

The FSM panel is populated from native Verilator records named `fsm_state` and `fsm_arc`. An empty panel or a `0/0` FSM summary means that the imported `.dat` does not contain native FSM records. The viewer cannot manufacture them.

Check the database directly:

```bash
grep -aE 'fsm_state|fsm_arc' /path/to/coverage.dat | head -50

verilator_coverage \
  --filter-type 'fsm_state|fsm_arc' \
  --report summary \
  /path/to/coverage.dat
```

If both commands show no FSM records, check the **coverage generation flow**, not only the GUI:

1. Confirm that the actual Verilator compile/Verilation command contains `--coverage` or `--coverage-fsm`.
2. Clean and rebuild the generated model after adding the option.
3. Run the rebuilt executable and ensure it writes a new `.dat` file at the path being inspected.
4. Check whether the RTL FSM fits the extraction patterns supported by your Verilator release. Native FSM extraction is conservative and does not recognize every coding style or every state-machine implementation.
5. Check the Verilator warnings and the documentation for the exact release used to build the model.

Verilator's FSM extraction documentation describes supported single-process and multi-process FSM coding forms and Verilator-specific FSM metacomments. Those metacomments may require RTL changes and should only be used when permitted by the design owner. The viewer itself does not require or perform RTL edits.

If `fsm_state` and `fsm_arc` records are present but the graph is missing, inspect the transition labels. The current graph renderer recognizes common separators such as `->`, `=>`, `→`, and `_TO_`. If labels do not encode source and destination states in a recognizable form, the transition table can still list the records, but the graph cannot reliably connect them.

## Roadmap and additional work

The following work is recommended before treating the tool as a complete coverage-analysis dashboard for multiple projects. Items below are **planned work**, not claims about the current implementation.

### Priority 0 — make FSM reporting dependable

- [ ] Build a small golden `.dat` fixture with multiple FSMs, repeated state labels, reset arcs, uncovered states, uncovered transitions, and hierarchy variations.
- [ ] Validate the parser against raw `.dat` files produced by each supported Verilator release (start with the team's installed 5.052 build, then add newer versions as needed).
- [ ] Confirm and document the exact metadata fields used for FSM identity, state names/encodings, transition labels, hierarchy, source file, and reset arcs.
- [ ] Group an FSM by a stable identity (not hierarchy alone); keep multiple FSMs in one module/instance separate.
- [ ] Parse state and arc records without relying only on the transition label text. Preserve unknown arc labels in a table rather than dropping them.
- [ ] Add a state-versus-next-state transition matrix: rows are source states, columns are destination states, and cells show hits/covered status.
- [ ] Display reset transitions separately and define whether they are included in the main transition denominator.
- [ ] Add clear diagnostics that distinguish: coverage option disabled, no FSM inferred by Verilator, no simulation hits, parser incompatibility, and graph-label parsing failure.
- [ ] Verify state, arc, graph, and matrix totals against `verilator_coverage` output, accounting for reset-arc reporting options.

### Priority 1 — add useful coverage matrices

- [ ] **Coverage-type matrix:** rows for source files/modules/instances and columns for line, branch, expression, toggle, FSM state, FSM arc, covergroup, and user/property coverage.
- [ ] **Hierarchy matrix:** roll up points by available `hier` metadata; allow drill-down from module to instance and point. Define behavior when hierarchy is absent or wildcard-collapsed.
- [ ] **Toggle matrix:** rows for signals and columns for individual bits, with separate `0->1` and `1->0` counts and uncovered directions.
- [ ] **Branch/expression matrix:** show each source line and its individual branch/expression outcomes, not just aggregate line counts.
- [ ] **Functional cross matrix:** derive real cross dimensions from raw metadata where possible, support N-dimensional crosses, retain complete bin names, and show uncovered combinations. Do not infer dimensions from `_x_` string splitting when reliable metadata is available.
- [ ] **Vector/value matrix:** preserve declared width, radix, value, range, transition sequence, and bin label when those fields exist in the database. Clearly mark metadata that the `.dat` does not provide.
- [ ] **Exclusion-aware calculations:** show original and remaining totals side by side; ensure all summaries, matrices, exports, and filtered `.dat` output use the same exclusion rules.
- [ ] Add CSV/JSON exports for each matrix and an integrated HTML report that preserves the dashboard's filters and selected scope.

### Priority 2 — multi-project and regression workflow

- [ ] Add regression comparison for two or more compatible `.dat` databases (newly covered, lost coverage, and hit-count changes).
- [ ] Add a project/run label to imports and exports so reports remain identifiable when several projects are analyzed from the same installation.
- [ ] Add automated parser tests for malformed records, missing metadata fields, duplicate labels, empty files, large files, and unknown coverage record types.
- [ ] Add CI tests that run the sample database and compare generated summary totals and CSV contents against expected values.
- [ ] Document performance and maximum supported input size using representative large project databases.

### Acceptance criteria for the next FSM release

- The same known FSM fixture produces identical state and transition totals in the GUI, CSV export, and command-line checks.
- Every FSM record remains visible even when the renderer cannot draw a graph for its label.
- Multiple FSMs with similar state names are not merged accidentally.
- Reset arcs are handled consistently and their denominator policy is visible.
- An input containing no native FSM records shows a clear explanation instead of implying that the RTL has no FSM.

## Repository layout

```text
general_verilator_coverage_explorer/
├── cover_report                       # CLI entry point
├── README.md                          # This file
├── .gitignore
├── examples/
│   └── sample_verilator_coverage.dat  # Small synthetic parser/UI fixture
└── verilator_coverage_explorer/
    ├── app.js                         # Browser UI logic
    ├── config.json                    # Server/workspace configuration
    ├── index.html                     # Dashboard markup
    ├── server.py                      # HTTP server, parser, import/export APIs
    └── style.css                      # Dashboard styling
```

Runtime files such as imported `.dat` copies, exclusion JSON, active import state, server PID files, and Python cache files should not be committed.

## Known limitations

- The parser is built around Verilator's raw `.dat` coverage-record format. Verilator may evolve record metadata between releases; test the parser against the precise tool version used by each project.
- Coverage categories are shown only when they were enabled, emitted, and collected.
- FSM extraction is a Verilator instrumentation concern. An RTL state machine may not be emitted as native FSM coverage if its coding structure is outside the supported extraction subset.
- Cross and vector presentation depends on the metadata emitted in the raw database. A viewer cannot recover source-level declarations that are not present there.
- FSM graphs are best-effort visualizations derived from state and arc labels; the tables are the more complete record-level view.
- The current release is a coverage record explorer, not yet a complete cross/hierarchy coverage-matrix product.

## References

- [Verilator command-line arguments](https://verilator.org/guide/latest/exe_verilator.html)
- [Verilator coverage collection and coverage analysis](https://verilator.org/guide/latest/simulating.html#coverage-analysis)
- [Verilator `verilator_coverage` command](https://verilator.org/guide/latest/exe_verilator_coverage.html)

Check the documentation matching the Verilator version installed in your environment, especially for experimental FSM extraction behavior and raw metadata details.

## License

No license is included in this repository yet. Before making the repository public or accepting external contributions, choose and add a license that matches the intended distribution and reuse terms.
