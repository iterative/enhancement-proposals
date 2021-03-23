- Enhancement Proposal PR: (leave this empty)
- Contributors: dberenbaum, (add your Github handle)

# Summary

The `dvc exp show` command shows a comparison of the results of different
experiments along with the parameters and metrics values for each experiment.
The command's output currently includes all values that are defined in `params.yaml` or
any other referenced parameters files in any DVC stage. It also includes all
metrics in any referenced metrics files in any DVC stage. This proposal allows
users to see only parameters and metrics relevant to specified pipeline stages.

See https://github.com/iterative/dvc/issues/5451 for more background.

# Motivation

`dvc exp show` is the main view by which users may compare experiments, and it's
becoming useful for other purposes, like comparing multiple commits (see
https://discord.com/channels/485586884165107732/563406153334128681/822464119906369606).

It's also likely the busiest DVC output:

```console
┏━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━
┃ Experiment     ┃ Created      ┃     auc ┃ prepare.split ┃ prepare.seed ┃ featurize.max_features ┃ featurize.ngrams ┃ train.seed ┃ train.n_estimators ┃ topic_modeling.seed ┃ top
┡━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━
│ workspace      │ -            │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ 20170428            │ 10
│ topic_modeling │ 04:41 PM     │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ 20170428            │ 10
│ master         │ Feb 17, 2021 │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                   │ -
│ └── exp-56d11  │ 04:40 PM     │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ 20170428            │ 10
│ 263732a        │ Feb 11, 2021 │  0.5674 │ 0.2           │ 20170428     │ 1500                   │ 2                │ -          │ 100                │ -                   │ -
│ b297969        │ Feb 11, 2021 │  0.5674 │ 0.2           │ 20170428     │ 1500                   │ 2                │ -          │ 100                │ -                   │ -
│ 4edd930        │ Feb 11, 2021 │  0.5674 │ 0.2           │ 20170428     │ 1500                   │ 2                │ -          │ 100                │ -                   │ -
│ 72ed9cd        │ Nov 15, 2020 │  0.5674 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 50                 │ -                   │ -
│ ├── exp-47dfa  │ Feb 17, 2021 │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                   │ -
│ └── exp-44136  │ Feb 17, 2021 │  0.5674 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 50                 │ -                   │ -
│ f8e9d93        │ Nov 15, 2020 │ 0.54175 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 50                 │ -                   │ -
```

This table view is from a simplified example and is still missing several
columns, so this view can become cluttered quickly. Eliminating irrelevant
information is a priority.

There are several ways in which the included parameters might be irrelevant:
* Parameters files may include values that aren't tracked by DVC at all (for
  example, debug level).
* The parameters may be used in a different pipeline branch (for example, the
  `topic_modeling` stage is in a separate branch of the pipeline from the
  `evaluate` stage, which produces the relevant experiment metrics).
* The parameters may be used in a downstream pipeline branch (for example, the
  post-processing `combine` stage parameters aren't relevant to experiment
  evaluation).

```console
                +-------------------+
                | data/data.xml.dvc |
                +-------------------+
                          *
                          *
                          *
                     +---------+
                     | prepare |
                     +---------+
                          *
                          *
                          *
                    +-----------+
                    | featurize |
                  **+-----------+**
              ****       *         ****
          ****           *             ***
        **              *                 ***
+-------+               *                    **
| train |             **                      *
+-------+            *                        *
         **        **                         *
           **    **                           *
             *  *                             *
        +----------+                  +----------------+
        | evaluate |                  | topic_modeling |
        +----------+                  +----------------+
                   ***             ****
                      *         ***
                       **     **
                     +---------+
                     | combine |
                     +---------+
```

There are no apparent reasons that users want to see untracked parameters in
`dvc exp show`. Unlike `dvc params diff` and `dvc metrics diff`, `dvc exp show`
is not applicable to arbitrary files. Likewise, there are no apparent reasons
that users want to see parameters or metrics from irrelevant stages of their
pipelines. Users could keep untracked parameters in other files if they don't
want them shown in `dvc exp show`, and they could exclude parameters or metrics
from irrelevant stages, but DVC already has enough information to understand
that these should not be shown.

# Detailed design

`dvc exp show` could take stages as (positional?) arguments, similar to `dvc exp
run`. The help message could be similar:

```console
positional arguments:
stages               Stages for which to print experiment info. 'dvc.yaml'
by default.
```

If no stages are passed, parameters and metrics defined in any stage would be
shown. This differs from current behavior only because untracked parameters
would be excluded.

If stages are passed, parameters and metrics defined anywhere in the pipeline
for those stages would be shown.

Using the above examples, `dvc exp show evaluate` would be equivalent to using
the `--exclude-params` and `exclude-metrics` options to exclude all parameters
and metrics for the `topic_modeling` and `combine` stages:

```console
┏━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━┳━━━┓
┃ Experiment     ┃ Created      ┃     auc ┃ prepare.split ┃ prepare.seed ┃ featurize.max_features ┃ featurize.ngrams ┃ train.seed ┃ train.n_estimators ┃ train.random_state ┃ … ┃
┡━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━╇━━━┩
│ workspace      │ -            │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ topic_modeling │ Mar 18, 2021 │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ ├── exp-e2784  │ 12:15 PM     │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ └── exp-44136  │ 10:29 AM     │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ master         │ Feb 17, 2021 │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ └── exp-56d11  │ Mar 18, 2021 │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ 263732a        │ Feb 11, 2021 │  0.5674 │ 0.2           │ 20170428     │ 1500                   │ 2                │ -          │ 100                │ 20170428           │ - │
│ b297969        │ Feb 11, 2021 │  0.5674 │ 0.2           │ 20170428     │ 1500                   │ 2                │ -          │ 100                │ 20170428           │ - │
│ 4edd930        │ Feb 11, 2021 │  0.5674 │ 0.2           │ 20170428     │ 1500                   │ 2                │ -          │ 100                │ 20170428           │ - │
│ 72ed9cd        │ Nov 15, 2020 │  0.5674 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 50                 │ -                  │ - │
│ ├── exp-47dfa  │ Feb 17, 2021 │ 0.51625 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ └── exp-44136  │ Feb 17, 2021 │  0.5674 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 50                 │ -                  │ - │
│ f8e9d93        │ Nov 15, 2020 │ 0.54175 │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 50                 │ -                  │ - │
│ 377c988        │ Nov 15, 2020 │ 0.54175 │ 0.2           │ 20170428     │ 500                    │ 1                │ 20170428   │ 50                 │ -                  │ - │
│ 27d4e7c        │ Nov 15, 2020 │       - │ 0.2           │ 20170428     │ 500                    │ 1                │ 20170428   │ 50                 │ -                  │ - │
│ 8bf2091        │ Nov 15, 2020 │       - │ 0.2           │ 20170428     │ 500                    │ 1                │ 20170428   │ 50                 │ -                  │ - │
│ ff9e2fa        │ Nov 15, 2020 │       - │ 0.2           │ 20170428     │ 500                    │ 1                │ 20170428   │ 50                 │ -                  │ - │
│ 11e1705        │ Nov 15, 2020 │       - │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ a82585e        │ Nov 15, 2020 │       - │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ c778a78        │ Nov 15, 2020 │       - │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ 551082e        │ Nov 15, 2020 │       - │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
│ 15bef96        │ Nov 15, 2020 │       - │ 0.2           │ 20170428     │ 1500                   │ 2                │ 20170428   │ 10                 │ -                  │ - │
└────────────────┴──────────────┴─────────┴───────────────┴──────────────┴────────────────────────┴──────────────────┴────────────┴────────────────────┴────────────────────┴───┘
```

This table includes parameters and metrics from the `evaluate` stage and all
stages upstream of it.  It's possible to also have a `--single-item` argument to
exclude upstream changes, but this is out of scope for now.

# How We Teach This

See [Detailed design](#detailed-design) for a proposed help message.

This is consistent with other commands, and it's only one additional argument to
an existing command, so it likely doesn't require much teaching. It might be
necessary to explicitly add that parameters and metrics from upstream stages
will be shown.

# Drawbacks

This proposal may require changes to how DVC collects and tracks parameters and
metrics. Implementation could break existing functionality, and it is
inconsistent with the current behaviors of `dvc params`, `dvc metrics`, and `dvc
exp diff` commands.

Further, this proposal adds to an already crowded set of `dvc exp show` options,
and there is no single way to show parameters and metrics that's going to work
for every use case. For example, this proposal does nothing to filter out
parameters for upstream stages, which may frequently be irrelevant if only the
stages near the end of the pipeline change during experimentation.

# Alternatives

There are many alternative methods to refine the output of `dvc exp show`:

* Current functionality: `exp show` already has options to include/exclude any
  column, so the flexibility to get the same output as suggested here already
  exists.
* Expand the include/exclude options to accept wildcard/glob/regex arguments for
  more flexibility to exclude an entire file or section of a file like
  `exclude-params=params.json:local_config*`.
* Save configuration options for `dvc exp show` so they can be reused and
  modified without typing out all options at the command line each time.
* Add a tsv/csv output format option in `dvc exp show` to enable users to
  manipulate the table with other tools.

This proposal does not preclude any of these options. The current output
will continue to be available except for untracked parameters. The other
alternatives may still be implemented.

Each of the alternatives provide flexibility to modify the `dvc exp show`
outputs, which is important but not the goal of this proposal. Instead, the
proposed feature uses the information DVC has to make smart decisions about what
to show and save the users from needing to make modifications that would always
be desirable.

# Unresolved questions

* What about consistency with `parameters`, `metrics`, and `exp diff` commands?
  Should these also be modified or left as is? As mentioned above, `parameters`
  and `metrics` in particular may be used on arbitrary files outside of DVC
  projects.
* Should experiments themselves be excluded based on what stages were run? Using
  the example above, if a user ran both `dvc exp run evaluate` and `dvc exp run
  topic_modeling` experiments, should all of these experiments show when running
  `dvc exp show evaluate`?
