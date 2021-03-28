# Cloud execution

- Enhancement Proposal PR: (leave this empty)
- Contributors: dmpetrov

# Summary

Users need to run ML experiments in cloud instances or remote machines.
DVC should support this out of the box.

# Motivation

Today users need to spend time on instrumneting remote execution when there is
  a lack of local resources for ML trainig (like GPU or RAM) or when it is a
  company polocy to train in a dedicated hardware.
The existing way of executing training through [CML](cml.dev) (CI/CD) is not
  the best approach in an experementation phase since it has an overhead on
  the execution time and it creates some footprint/records in CI/CD that is
  a bit overwelming when number of experiments is huge (10s, 100s or 1000s).
This feature can save time for users and improve ML experimentation exerience.

There are multiple scenarios of remote execution:

1. Allocate a cloud istance (with a specified configuration) and execute an
  ML experiment `dvc exp run` (or `dvc repro`) in on it.
2. Allocate a cloud istance and execute a perticular stage of an experiement
  on it.
3. Execute in an existing running instance (hot instance) without instance
   allocation.
4. Execute in a remote machine (cloud or on-premise one).
5. Execute in one of the exisintg running instances (instance pool).
6. Execute a set of experiments (from exp queue for example) in an instance
   pool or a single instance.
7. Execute an experiment from SaaS\Web using one of the methods from
   above and DVC machinary.

# Detailed design

## Iterative Terraform provider

The problem of executing cloud instances was solved in the CML project.

CML separates out the user API part (commands) and the cloud instace
  management code part.
The cloud management is extracted to a separate **Iterative Terraform (TF)
  provider**:
  https://github.com/iterative/terraform-provider-iterative

The Iterative provider is specific to data scienitists, not DevOps (TF's
target audience). It has a bit simplified API and some ML specific
functionality (some of these are comming) like: recover spot-instance if
it was rewoked or attach a reusable drive.

The provider can be used from DVC for cloud instace management. Some Python
library might required for access to GoLang specific terraform-provider.


## Instances configuration

Instance configuration is a definition of an instance and it's behaviour:
- cloud providor (AWS, GCP< Azure, on-premise, ...)
- instance type
- many other options

Instance configuration is well developed in CML project and can be reused.

## Executors and naming

Many instance types and even clouds could be used in a single DVC project.
We need a way of configuring **executors** and reusing them the same way
  as user can configure a data remote.
It can look like this:
```bash
$ dvc exp run --executor my-gpu-tesla
...
$ dvc exp run --executor my-gpu-tesla8
...
$ dvc exp run --executor tesla8-ram
```

# How We Teach This

We need to introduce a concept **Executor** to DVC.
This convept should not break the compatibility and should not require
significant changes in the docs (except command options).
However, it will require a new section in the docs to explain the cloud
execution for the users who'd like to use this feature.

# Drawbacks

This feature opens new user's scenarios which increases complexity of DVC,
it will require new sections in the docs, etc.
It might be beneficial to extract as much of this functionality as possible to
external products and tools (terraform providers for example). 

More people might confuse DVC with data engineering pipeline execution tools
such as AirFlow, Dagster and others.
We should provide a clear guideline when DVC should be used for workflow
execution and when it should not.

# Alternatives

The cloud management logic can be implemented in DVC, not in an external tool.

# Unresolved questions

?
