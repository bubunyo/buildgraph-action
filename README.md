# buildgraph-action

GitHub Actions composite action for [BuildGraph](https://github.com/bubunyo/buildgraph) — intelligent selective rebuild for Go monorepos.

The action runs `buildgraph analyze` to detect which services changed, uploads an impact report, then runs `buildgraph generate` to save a fresh baseline for the next run. All in one step.

## Usage

```yaml
jobs:
  analyze:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.bg.outputs.services }}
      has_changes: ${{ steps.bg.outputs.has_changes }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run BuildGraph
        id: bg
        uses: bubunyo/buildgraph-action@v0.0.0-alpha

  build:
    needs: analyze
    if: needs.analyze.outputs.has_changes == 'true' && needs.analyze.outputs.services != '[]'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJson(needs.analyze.outputs.services) }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      - name: Build ${{ matrix.service }}
        run: go build -o bin/${{ matrix.service }} ./${{ matrix.service }}/...

      - name: Test ${{ matrix.service }}
        run: go test ./${{ matrix.service }}/...
```

## Inputs

| Input | Description | Default |
|---|---|---|
| `go-version-file` | Path to `go.mod` used to select the Go toolchain | `go.mod` |
| `baseline-artifact-name` | Artifact name used to persist the baseline between runs | `buildgraph-baseline` |
| `baseline-retention-days` | Days to retain the baseline artifact | `90` |
| `config` | Path to `buildgraph.yaml` (overrides default location) | _(empty)_ |
| `baseline` | Path to the baseline file (overrides `buildgraph.yaml`) | _(empty)_ |
| `working-directory` | Directory to run buildgraph from | `.` |

## Outputs

| Output | Description |
|---|---|
| `has_changes` | `"true"` if any functions changed, `"false"` otherwise |
| `services` | JSON array of relative service paths to rebuild, e.g. `["services/service-a","services/service-b"]` |
| `impact-file` | Path to the `impact.json` report written by `buildgraph analyze` |

## Examples

### Pin to a specific version

```yaml
- uses: bubunyo/buildgraph-action@v1
  with:
    version: v0.2.0
```

### Monorepo with a custom config path

```yaml
- uses: bubunyo/buildgraph-action@v1
  with:
    config: infra/buildgraph.yaml
```

### Services in a non-standard working directory

```yaml
- uses: bubunyo/buildgraph-action@v1
  with:
    working-directory: backend
```

### Share the baseline across multiple workflows

If you run BuildGraph in more than one workflow, give each a unique artifact name so their baselines don't collide:

```yaml
- uses: bubunyo/buildgraph-action@v1
  with:
    baseline-artifact-name: buildgraph-baseline-${{ github.workflow }}
```

### Read the impact report in a later step

```yaml
- name: Run BuildGraph
  id: bg
  uses: bubunyo/buildgraph-action@v1

- name: Print affected services
  run: cat ${{ steps.bg.outputs.impact-file }}
```

## How it works

1. Restores the baseline artifact from the previous run (`continue-on-error: true` so the first run is safe)
2. Installs the `buildgraph` CLI via `go install`
3. Runs `buildgraph analyze` and extracts `has_changes` and `services` into job outputs
4. Uploads the impact report as an artifact (always, even on failure)
5. Runs `buildgraph generate` to write a fresh baseline
6. Uploads the new baseline artifact for the next run

## License

MIT
