# Title: Unified show/diff command for all output types

- Enhancement Proposal PR: (leave this empty)
- Contributors: dberenbaum

# Summary

Establish a unified command to `show` or `diff` all outputs of specified types,
including:

* Metrics
* Plots
* Images

These would all be put under a single `dvc outputs show/diff` command.

# Motivation

* Users can `show` and `diff` multiple different output types within a single
  command.
* Reduces number of top-level commands by grouping these together.
* Promotes consistent UI, making it easier for users and for consumption by
  downstream apps.
* Generates flexible abstraction for future needs (additional output types or
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

### Common Abstractions

All output types should share a common interface for `show` and `diff` so that
outputs can be easily combined into one report. Multiple report formats may be
supported: HTML may be the most sensible for user consumption, and JSON or some
other machine-readable format may be needed for downstream app consumption.

`show` implementations will:
* Support file `targets` as positional arguments.
* Support directories as `targets`.
* By default, show all outputs in the workspace.

`diff` implementations will:
* Support arbitrary number of Git `revisions` as positional arguments to
  compare.
* Support `--targets` as specific files for which to compare revisions.
* Support comparing the last `n` commits (see `dvc exp show -n`).
* Support comparing an arbitrary number of filenames in the current workspace to
  each other (see https://github.com/iterative/dvc/issues/5693).
* By default, compare all outputs in the workspace to the `HEAD` commit.

The existing arguments available in the current `show` and `diff`
implementations will need to be reviewed to determine which can be adopted,
which need to be changed, and which can be dropped.

# How We Teach This

One potential confusion is that "outputs" is vague and used elsewhere in docs,
but it provides enough flexibility that any future output types can be added
easily.

Another potential confusion is the number of different ways in which users can
invoke `diff`, which makes it powerful but possibly overloaded. Care should be
taken when designing and documenting the `diff` command line interface to make
it explainable.

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
