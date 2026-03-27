# example-matrix-strategy

This repo demonstrates how to build a **dynamic** matrix strategy in GitHub Actions using only tools that are pre-installed on the runner. No third-party actions required beyond `actions/checkout`. It is appropriate for monorepos and other situations where you want to discover work dynamically instead of hardcoding it.

The 2026 edition turns this into a **shootout** — five different approaches to matrix generation, each running as its own job, so you can compare simplicity, speed, and overhead directly in the [Actions UI](https://github.com/karldreher/example-matrix-strategy/actions).

## How It Works

The pattern has two jobs:

1. **Generate** — discover directories and output a JSON array
2. **Consume** — use `strategy.matrix` with `fromJson` to fan out into parallel jobs

### Job 1: Matrix Generation

The recommended approach uses `ls` and `jq`, both pre-installed on `ubuntu-24.04`:

```bash
echo "matrix=$(ls -d example_*/ | jq -Rnc '[inputs]')" >> $GITHUB_OUTPUT
```

Breaking this down:
- `ls -d example_*/` lists directories matching the glob
- `jq -Rnc '[inputs]'` reads each line as raw text (`-R`), collects them into an array via null input + `inputs` (`-n`), and outputs compact JSON (`-c`)

The result is a JSON array sent to `$GITHUB_OUTPUT` for downstream jobs:

```json
["example_1/","example_2/"]
```

This is then declared as a job output:

```yaml
outputs:
  matrix: ${{ steps.matrix.outputs.matrix }}
```

### Job 2: Matrix Consumption

Matrix strategy is implemented at the **job level**. The consumer job unpacks the JSON array with `fromJson` and fans out — one job per array item:

```yaml
use_matrix_jq:
  runs-on: ubuntu-24.04
  needs: [createMatrix_jq]
  strategy:
    matrix:
      paths: ${{ fromJson(needs.createMatrix_jq.outputs.matrix) }}
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

Notice that `run:` steps use the shell variable `"${MATRIX_PATH}"` rather than directly interpolating `${{ matrix.paths }}`. This avoids [script injection](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections) — a common pitfall where expressions are interpolated into the shell script *before* the shell executes, allowing crafted values to break out of the intended command.

Setting the value as an `env:` variable at the job level and referencing it as `"${MATRIX_PATH}"` in `run:` steps is the recommended pattern.

## The Shootout

The [workflow](.github/workflows/main.yaml) implements five different approaches to matrix generation. Each runs as its own job with its own consumer, so you can compare them end-to-end in the Actions UI.

### jq (Recommended)

Pre-installed on `ubuntu-24.04`. One line. Purpose-built for JSON.

```bash
echo "matrix=$(ls -d example_*/ | jq -Rnc '[inputs]')" >> $GITHUB_OUTPUT
```

### Pure Bash

Zero external tools — shell builtins only. Works well for controlled directory names but fragile with special characters.

```bash
dirs=(example_*/)
printf -v items ',"%s"' "${dirs[@]}"
echo "matrix=[${items:1}]" >> $GITHUB_OUTPUT
```

### Node.js

Pre-installed on `ubuntu-24.04` (Node 20). More verbose but familiar to JavaScript teams.

```bash
matrix=$(node -e "
  const fs = require('fs');
  const dirs = fs.readdirSync('.')
    .filter(f => fs.statSync(f).isDirectory() && f.startsWith('example_'))
    .map(f => f + '/');
  console.log(JSON.stringify(dirs));
")
echo "matrix=${matrix}" >> $GITHUB_OUTPUT
```

### Bun

**Not pre-installed** — requires the [`oven-sh/setup-bun`](https://github.com/oven-sh/setup-bun) action. This demonstrates the overhead of adding a non-pre-installed tool: the setup step downloads and installs Bun before the generation command can run.

```bash
matrix=$(bun -e "
  import { readdirSync, statSync } from 'fs';
  const dirs = readdirSync('.')
    .filter(f => statSync(f).isDirectory() && f.startsWith('example_'))
    .map(f => f + '/');
  console.log(JSON.stringify(dirs));
")
echo "matrix=${matrix}" >> $GITHUB_OUTPUT
```

### ripgrep

**Not pre-installed** — requires `apt-get install`. ripgrep is a content search tool, not a directory lister, so this is deliberately using the wrong tool for the job. It still needs `jq` for JSON construction, and the install step adds significant overhead.

```bash
echo "matrix=$(rg --files | grep '^example_' | sed 's|/.*|/|' | sort -u | jq -Rnc '[inputs]')" >> $GITHUB_OUTPUT
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
