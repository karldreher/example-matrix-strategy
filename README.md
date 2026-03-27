# example-matrix-strategy

This repo demonstrates how to build a **dynamic** matrix strategy in GitHub Actions using only tools that are pre-installed on the runner. No third-party actions required beyond `actions/checkout`. It is appropriate for monorepos and other situations where you want to discover work dynamically instead of hardcoding it.

The 2026 edition is a **matrix of matrices** — a static matrix over tools feeds a dynamically-generated matrix over paths. Five different approaches to matrix generation run as cells in a single job, so you can compare simplicity, speed, and overhead directly in the [Actions UI](https://github.com/karldreher/example-matrix-strategy/actions).

## How It Works

The workflow has two jobs, each with its own matrix strategy:

1. **`generate`** — a **static** matrix over tools (`jq`, `bash`, `node`, `bun`, `ripgrep`). Each cell discovers directories and outputs the same JSON array using a different approach.
2. **`consume`** — a **dynamic** matrix over paths, built from the output of `generate`. Fans out into one job per directory.

This is the "matrix of matrices" pattern: Job 1's matrix is hardcoded in the workflow, but Job 2's matrix is constructed at runtime from Job 1's output.

### Job 1: Generate (static tool matrix)

The generator job uses `strategy.matrix.tool` to run five approaches in parallel. Each tool gets its own step, filtered with `if: matrix.tool == '...'`, all sharing the same `id: matrix` so the output reference stays consistent:

```yaml
generate:
  runs-on: ubuntu-24.04
  strategy:
    matrix:
      tool: [jq, bash, node, bun, ripgrep]
  steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup Bun
      if: matrix.tool == 'bun'
      uses: oven-sh/setup-bun@v2

    - name: Install ripgrep
      if: matrix.tool == 'ripgrep'
      run: sudo apt-get update && sudo apt-get install -y ripgrep

    - name: Generate Matrix (jq)
      if: matrix.tool == 'jq'
      id: matrix
      run: echo "matrix=$(ls -d example_*/ | jq -Rnc '[inputs]')" >> $GITHUB_OUTPUT

    - name: Generate Matrix (bash)
      if: matrix.tool == 'bash'
      id: matrix
      run: # ... pure bash JSON construction

    - name: Generate Matrix (node)
      if: matrix.tool == 'node'
      id: matrix
      run: # ... node -e with fs.readdirSync

    # ... one step per tool, same id, same output key
```

Every cell produces the same JSON array — the last cell to finish sets the job output:

```json
["example_1/","example_2/"]
```

Conditional `if:` steps handle tool installation only when needed. Bun requires the `oven-sh/setup-bun` action; ripgrep requires `apt-get install`. The other three tools are pre-installed on `ubuntu-24.04`.

### Job 2: Consume (dynamic paths matrix)

The consumer job unpacks the JSON array with `fromJson` and fans out — one job per directory:

```yaml
consume:
  runs-on: ubuntu-24.04
  needs: [generate]
  strategy:
    matrix:
      paths: ${{ fromJson(needs.generate.outputs.matrix) }}
  env:
    MATRIX_PATH: ${{ matrix.paths }}
  steps:
    - name: Checkout
      uses: actions/checkout@v4
    - name: Echo the output
      run: echo "${MATRIX_PATH}"
    - name: Cat the file
      run: cat "${MATRIX_PATH}file.txt"
```

You can reference `${{ matrix.paths }}` anywhere in the job context — environment variables, step inputs, etc.

### Security: Expression Injection

Notice that `run:` steps use shell variables (`"${MATRIX_PATH}"`, `"${TOOL}"`) rather than directly interpolating `${{ matrix.paths }}` or `${{ matrix.tool }}`. This avoids [script injection](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections) — a common pitfall where expressions are interpolated into the shell script *before* the shell executes, allowing crafted values to break out of the intended command.

Setting values as `env:` variables and referencing them as shell variables in `run:` steps is the recommended pattern.

## The Shootout

Each cell in the `generate` job's tool matrix runs a different approach. You can compare their timing directly in the Actions UI matrix view.

### jq (Recommended)

Pre-installed on `ubuntu-24.04`. One line. Purpose-built for JSON.

```bash
result=$(ls -d example_*/ | jq -Rnc '[inputs]')
```

### Pure Bash

Zero external tools — shell builtins only. Works well for controlled directory names but fragile with special characters.

```bash
dirs=(example_*/)
printf -v items ',"%s"' "${dirs[@]}"
result="[${items:1}]"
```

### Node.js

Pre-installed on `ubuntu-24.04` (Node 20). More verbose but familiar to JavaScript teams.

```bash
result=$(node -e "
  const fs = require('fs');
  const dirs = fs.readdirSync('.')
    .filter(f => fs.statSync(f).isDirectory() && f.startsWith('example_'))
    .map(f => f + '/');
  console.log(JSON.stringify(dirs));
")
```

### Bun

**Not pre-installed** — requires the [`oven-sh/setup-bun`](https://github.com/oven-sh/setup-bun) action. This demonstrates the overhead of adding a non-pre-installed tool: the setup step downloads and installs Bun before the generation command can run.

```bash
result=$(bun -e "
  import { readdirSync, statSync } from 'fs';
  const dirs = readdirSync('.')
    .filter(f => statSync(f).isDirectory() && f.startsWith('example_'))
    .map(f => f + '/');
  console.log(JSON.stringify(dirs));
")
```

### ripgrep

**Not pre-installed** — requires `apt-get install`. ripgrep is a content search tool, not a directory lister, so this is deliberately using the wrong tool for the job. It still needs `jq` for JSON construction, and the install step adds significant overhead.

```bash
result=$(rg --files | grep '^example_' | sed 's|/.*|/|' | sort -u | jq -Rnc '[inputs]')
```

### Results

| Approach | Pre-installed | Lines of Shell | Command Time | Job Duration | Notes |
|----------|---------------|----------------|--------------|--------------|-------|
| jq       | Yes           | 1              | 5ms          | 5s           | Recommended. Purpose-built for JSON |
| bash     | Yes (builtin) | 3              | 2ms          | 3s           | Zero dependencies, fragile with special chars |
| Node.js  | Yes           | ~5             | 69ms         | 4s           | Familiar to JS teams |
| Bun      | No            | ~5 + setup     | 22ms         | 8s           | Requires `oven-sh/setup-bun` action |
| ripgrep  | No            | ~3 + install   | 7ms          | 15s          | Content search tool; still needs jq for JSON |

> **Command Time** = just the matrix generation command. **Job Duration** = total job wall time including checkout, setup/install, and generation. Measured on `ubuntu-24.04` runners, March 2026.

## Alternatives Not Shown

[**dorny/paths-filter**](https://github.com/dorny/paths-filter) is a popular, feature-rich action for filtering changed paths. It's a great tool for complex filtering scenarios, but it adds the overhead of downloading and executing a third-party action. For simple directory enumeration like this example, pre-installed CLI tools are faster and have zero external dependencies.

## References

- [Using a matrix for your jobs](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs) — GitHub's official matrix strategy documentation
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions) — expression injection and other security considerations
- [How to Make Dynamic Matrix in GitHub Actions](https://tomasvotruba.com/blog/2020/11/16/how-to-make-dynamic-matrix-in-github-actions/) — original inspiration (2020, may reference deprecated syntax like `::set-output`)
