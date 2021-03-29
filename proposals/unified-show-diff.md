# Title: Unified show/diff command for all output types

- Enhancement Proposal PR: (leave this empty)
- Contributors: dberenbaum

# Summary

Establish a unified command to combine and `show` or `diff` all outputs of
specified types, including:

* Metrics
* Plots
* Images (depends on https://github.com/iterative/dvc/discussions/5681)

These would all be put under a single `dvc outputs show/diff` command.

# Motivation

* Enable users to `show` and `diff` multiple different output types within a
  single command.
* Reduce number of top-level commands by grouping these together.
* Promote consistent UI, making it easier for users and for consumption by
  downstream apps.
* Generate flexible abstraction for future needs (additional output types or
  additional functionality across all output types).

# Detailed design

### Definitions

**Outputs**: include any `outs` specified in dvc stages, but only `metrics`,
`plots`, and `images` will be supported for now. It's possible that other output
types may be added in the future as needed.

### Command Line Interface

Combine all `show`/`diff` commands into one top-level command:

```console
usage: dvc outputs [--type] {show,diff}

Commands to visualize and compare outputs (metrics, plots, and images).


positional arguments:
  COMMAND
    show    Generate report from output files.
    diff    Compare multiple versions of outputs in a single report.

optional arguments:
    --type  Restrict report to single output type (metrics, plots, images).
```

### Requirements of each command:

`show` implementations will:
* Support file/directory `targets` as positional arguments.
* By default, show all outputs in the workspace.

`diff` implementations will:
* Support arbitrary number of Git revisions (more than two) as positional
  arguments to compare.
* Support `--targets` as specific files for which to compare revisions.
* Support comparing the last `n` commits (see `dvc exp show -n` and
  https://github.com/iterative/dvc/issues/5710).
* Support comparing an arbitrary number of filenames (more than two) in the
  current workspace to each other (see
  https://github.com/iterative/dvc/issues/5693).
* By default, compare all outputs in the workspace to the `HEAD` commit.

The existing arguments available in the current `show` and `diff`
implementations will need to be reviewed to determine which can be adopted,
which need to be changed, and which can be dropped.

### Abstractions from Git

In Git, the [diff
driver](https://git-scm.com/docs/gitattributes#_defining_an_external_diff_driver)
determines how the diff is calculated between the files. A custom diff driver
can be configured in `.gitconfig` and used for specified files in
`.gitattributes`. Each driver takes the same parameters passed by Git. The
[difftool](https://git-scm.com/docs/git-difftool) is the frontend that shows the
outputs of the diffs.  

Metrics, plots, images, and other output types each can be thought of as having
their own different `show`/`diff` driver (`show` and `diff` can probably share a
driver). There may be a common set of parameters for all drivers, although there
might be parameters specific to a particular driver (for example, the `template`
for `dvc plots`). Each driver should return a machine-readable diff.

There could also be multiple difftool frontends to visualize the diff of all
output types together (for example, all diffs get aggregated into one HTML page.
Other downstream apps may implement their own difftools. There also might be
difftools for specific drivers/output types (for example, an image file viewer).

# How We Teach This

One potential confusion is that "outputs" is vague and used elsewhere in docs,
but it provides enough flexibility that any future output types can be added
easily.

Another potential confusion is the number of different ways in which users can
invoke `diff`, which makes it powerful but possibly overloaded. Care should be
taken when designing and documenting the `diff` command line interface to make
it explainable.

If support for custom drivers or difftools is prioritized, these concepts will
need to be explained along with instructions and examples for how to implement
them.

The primary sections within the docs that will be impacted are (this is not an
exhaustive list):
* content/docs/command-reference/metrics
* content/docs/command-reference/plots
* content/docs/start/metrics-parameters-plots.md

# Drawbacks

A common abstraction for all output types minimizes the flexibility of each
output type's `show` and `diff` commands.

This would also be a breaking change for users of the existing commands.

# Alternatives

* Keep separate commands for each output type but work to achieve more
  consistency and abstract where possible.
* Combine commands but report each output type separately.

# Unresolved questions

* Should `params` be included even though they are not outputs? Should `dvc
  params diff` be modified to be consistent with this proposal?
* How should `dvc exp diff` be modified per this proposal?
