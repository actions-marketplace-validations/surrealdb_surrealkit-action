# surrealkit-action

GitHub Action for [SurrealKit](https://github.com/surrealdb/surrealkit) - install the CLI and run migrations, tests, sync, or seed against a [SurrealDB](https://surrealdb.com) instance in CI.

Supports Linux x64, macOS ARM64, and Windows x64 runners.

## Quick start

### Run the test suite against an ephemeral SurrealDB

```yaml
name: tests
on: [push, pull_request]

jobs:
  surrealkit-test:
    runs-on: ubuntu-latest
    services:
      surrealdb:
        image: surrealdb/surrealdb:latest
        ports: ['8000:8000']
        options: >-
          --health-cmd "/surreal is-ready"
          --health-interval 2s
          --health-timeout 5s
          --health-retries 10
        env:
          SURREAL_USER: root
          SURREAL_PASS: root
        # surrealdb start requires args; use a command override:
        # (GitHub service containers pass env, but not CLI args - if you need args,
        # run surrealdb in a step instead; see "Without service containers" below.)
    steps:
      - uses: actions/checkout@v4
      - uses: surrealdb/surrealkit-action@v1
        with:
          command: test
          host: http://localhost:8000
          user: root
          pass: root
```

### Without service containers (more flexible)

```yaml
jobs:
  surrealkit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Start SurrealDB
        run: |
          curl -sSf https://install.surrealdb.com | sh
          surreal start --user root --pass root memory &
          for i in {1..20}; do
            curl -fsS http://localhost:8000/health && break
            sleep 1
          done

      - uses: surrealdb/surrealkit-action@v1
        with:
          command: test
          args: --fail-fast --json-out results.json
          host: http://localhost:8000
          user: root
          pass: root

      - uses: actions/upload-artifact@v7
        if: always()
        with:
          name: surrealkit-test-results
          path: results.json
```

### Apply migrations to a remote database

```yaml
- uses: surrealdb/surrealkit-action@v1
  with:
    command: rollout
    args: start --yes
    host: ${{ secrets.SURREAL_HOST }}
    ns:   ${{ secrets.SURREAL_NS }}
    db:   ${{ secrets.SURREAL_DB }}
    user: ${{ secrets.SURREAL_USER }}
    pass: ${{ secrets.SURREAL_PASS }}
```

### Install only - run surrealkit manually in later steps

```yaml
- uses: surrealdb/surrealkit-action@v1
  with:
    command: ''          # skip the run step
    version: v0.5.3

- run: surrealkit sync --watch=false
```

## Inputs

| Name | Default | Description |
| --- | --- | --- |
| `version` | `latest` | SurrealKit release to install, e.g. `v0.5.3`. `latest` resolves via the GitHub releases API. |
| `command` | `test` | Subcommand to run after install (`test`, `sync`, `seed`, `rollout`, `lint`, `status`, ...). Set to `''` to install only. |
| `args` | `''` | Additional args appended to the subcommand. |
| `working-directory` | `.` | Directory containing the SurrealKit project (the folder that holds `database/`). |
| `host` | - | SurrealDB host, e.g. `http://localhost:8000`. Exported as `SURREALDB_HOST`. |
| `ns` | - | Namespace. Exported as `SURREALDB_NAMESPACE`. |
| `db` | - | Database name. Exported as `SURREALDB_NAME`. |
| `user` | - | Username. Exported as `SURREALDB_USER`. |
| `pass` | - | Password. Exported as `SURREALDB_PASSWORD`. |
| `github-token` | `${{ github.token }}` | Used only to resolve `latest` against the releases API. |

Any connection input left blank is **not** exported - SurrealKit falls back to its own defaults (`localhost:8000`, `root/root`, namespace `db`, database `test`) or a `.env` file in the working directory.

## Outputs

| Name | Description |
| --- | --- |
| `version` | The resolved version that was installed (e.g. `v0.5.3`). |
| `path` | Absolute path to the directory containing the `surrealkit` binary. |

## Notes

- The `test` subcommand runs SurrealKit's declarative TOML suites from `database/tests/suites/*.toml`. Each suite runs in an isolated ephemeral namespace/database, so you only need one running SurrealDB instance per job.
- macOS Intel (`x86_64-apple-darwin`) is not currently published by SurrealKit. Use `macos-latest` (ARM64) or build from source.
- The binary is cached at `~/.surrealkit/<version>/` for the lifetime of the runner.
