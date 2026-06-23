# karasu-action

A GitHub Action that renders [karasu](https://github.com/kompiro/karasu) `.krs`
architecture diagrams to SVG in your CI pipeline — a single `uses:` line instead
of copy-pasting workflow YAML.

karasu is a text-based architecture modeling tool inspired by the C4 model, with
its own vocabulary separating logical, physical, and organizational structure.

## Usage

```yaml
- uses: kompiro/karasu-action@v1
  with:
    input: docs/architecture.krs
    output: docs/architecture.svg
```

A typical workflow that re-renders the diagram on every push and commits the SVG
back to the repo:

```yaml
name: Render architecture diagram

on:
  push:
    branches: [main]
    paths: ["**/*.krs"]

permissions:
  contents: write

jobs:
  render:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: kompiro/karasu-action@v1
        with:
          input: docs/architecture.krs
          output: docs/architecture.svg

      - name: Commit the rendered SVG
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add docs/architecture.svg
          git diff --cached --quiet || git commit -m "docs: re-render architecture diagram [skip ci]"
          git push
```

The committed SVG renders natively in GitHub's file browser and Markdown previews,
so readers see up-to-date diagrams without installing anything.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `input` | ✅ | — | Path to the entry `.krs` file. `@import`s are resolved relative to it, so you only point at the top-level file. |
| `output` | ✅ | — | Path to write the SVG. |
| `version` | | `latest` | The `karasu` npm version to run (e.g. `0.1.0`). Pin it to avoid unexpected changes. |
| `view` | | _(all)_ | Render a single view: `system`, `deploy`, or `org`. Omit to render all views bundled into one SVG with tab navigation. |

### Rendering a single view

```yaml
- uses: kompiro/karasu-action@v1
  with:
    input: docs/index.krs
    output: docs/system.svg
    view: system
```

### Pinning the version

```yaml
- uses: kompiro/karasu-action@v1
  with:
    input: docs/index.krs
    output: docs/architecture.svg
    version: "0.1.0"
```

## Migrating from the workflow templates

This action replaces the copy-paste workflow templates previously published in
[`kompiro/karasu` → `examples/github-actions/`](https://github.com/kompiro/karasu/tree/main/examples/github-actions).
The render step is equivalent:

```yaml
# before — copied template
- run: npx --yes karasu@latest render docs/architecture.krs --output docs/architecture.svg

# after — this action
- uses: kompiro/karasu-action@v1
  with:
    input: docs/architecture.krs
    output: docs/architecture.svg
```

For the multi-file (matrix) template, keep your `strategy.matrix` and call the
action per entry:

```yaml
strategy:
  matrix:
    diagram:
      - { input: docs/system/index.krs, output: docs/system/architecture.svg }
      - { input: docs/deploy/index.krs, output: docs/deploy/architecture.svg }
steps:
  - uses: actions/checkout@v4
  - uses: kompiro/karasu-action@v1
    with:
      input: ${{ matrix.diagram.input }}
      output: ${{ matrix.diagram.output }}
```

## Requirements

- A GitHub-hosted runner (these ship with Node.js). On self-hosted runners,
  ensure Node.js (≥ 20) is available — the action invokes `karasu` via `npx`.
- `contents: write` permission only if your workflow commits the SVG back.

## How it works

This is a thin composite action: it creates the parent directory of `output` if
needed, then runs
`npx --yes karasu@<version> render <input> --output <output> [--view <view>]`.
Inputs are passed through the environment (not interpolated into the shell) so a
crafted value can't inject commands.

## License

[Apache-2.0](./LICENSE) — © 2026 Hiroki Kondo.
