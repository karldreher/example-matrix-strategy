# example-matrix-strategy

This repo contains a simple example of how to compose a **dynamic** matrix strategy using Github Actions.  It is appropriate for use in monorepos, and other situations where "less-is-more" to avoid refactoring later.

Please feel free to review the [workflow](https://github.com/karldreher/example-matrix-strategy/blob/main/.github/workflows/main.yaml) and [Actions history](https://github.com/karldreher/example-matrix-strategy/actions) while following along with this example, to gain a better understanding of matrix strategy in general, as well as how to dynamically create one.

# Example

This is a simple example, which will help you understand a more complex concept.

In this repo there are two folders, "example_1" and "example_2".
We're going to use `ls` to gather these folders to construct our matrix.

## Example - Matrix Generation (Job 1)

Here, we will review the generation of the matrix.

```bash
        paths=$(ls -d */)
        values=$(jq -n --arg array "${paths}" '$array | split("\n")')
        echo matrix=${values} >> $GITHUB_OUTPUT
```
In the first line, we are using an `ls` primitive to gather the paths.
In the second line, things get a little more complicated; we're using `jq` to put these into an array which will look like this when it's done:

```json
[
  "example_1/",
  "example_2/"
]
```

**This structure is important**, because the downstream "Job 2" will use the `fromJSON` expression to unpack these.  Especially because our `ls` output is newline-separated, JSON is probably the easiest way for us to get the data from one place to another.  There are other approaches, but Actions is generally speaking expecting an array e.g. `["item-1", "item-2"]`.  (What makes it a matrix is usually 2 or more arrays together, this concept is not explained here)

## Example - Matrix Strategy (Job 2)

The matrix strategy itself is relatively easy to implement, once the output is in a predictable format.

Matrix strategy is implemented at a **Job level**, so it is advantageous to separate the generation and consumption of the matrix.  Actually based on our actions, matrix strategy will fork into the number of jobs specified.

```yaml
  use-matrix:
    runs-on: ubuntu-22.04
    needs: [ createMatrix ]
    strategy:
      matrix:
        paths: ${{ fromJson(needs.createMatrix.outputs.matrix) }}

    env:
      MATRIX_PATH: ${{ matrix.paths }}
```

Here, after constructing the matrix based on the output of the last job, we just need to reference `${{ matrix.paths }}` anywhere that we want to reuse the items in the array.  As demonstrated in this workflow, you could set this to an environment variable or compose other actions, but pretty much anything in the context of the Job could take advantage of this as a variable.

### References

- https://tomasvotruba.com/blog/2020/11/16/how-to-make-dynamic-matrix-in-github-actions/

- https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs
