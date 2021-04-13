# Cloud execution

- Enhancement Proposal PR: (leave this empty)
- Contributors: dmpetrov

# Summary

Users need to run ML experiments in cloud instances or remote machines when
user does not have enough resources in a local machine or a bgger scale is
required for ML training (5 GPU instances running differnet experiments).

The cloud execution should be supported in DVC experiments level when all
the training results are presented in a single experiments table.

# Motivation

Today users need to spend time on instrumenting remote execution when there is
  a lack of local resources for ML trainig (like GPU or RAM) or when it is a
  company policy to train on dedicated hardware or on Kubernetes cluster.
The existing way of executing training through [CML](cml.dev) (CI/CD) works
  great for handing off ML models between team members but is not the best
  approach in an experimentation phase since CI/CD creates an overhead on the
  execution time and it creates some footprint/records in CI/CD that is
  a bit overwhelming when the number of experiments is huge (10s, 100s or
  1000s).
This feature can save time for users and improve ML experimentation experience.

## Scenarios

1. **Run exp.** Allocate a cloud istance (with a specified configuration) and
  execute an ML experiment `dvc exp run` (or `dvc repro`) on it.
2. **Run stage.** Allocate a cloud istance and execute a particular stage of
   an experiment on it.
3. **Hot instance.** Execute on an existing running instance without instance
   allocation.
4. **Remote machine.** Execute in a remote machine (cloud or on-premise one).
5. **Instance pool.** Execute on one of the existing running instances.
6. **Exp queue** Execute a set of experiments (from exp queue for example) in
   an instance pool or a single instance.
7. **Web run.** Execute an experiment from SaaS\Web using one of the methods
   from above and DVC machinery.

## Optimizations

Multiple cloud instance optimization technicques might be used to optimize
  the usage of resources or execution time:
1. **Spot instances.**
2. **Transparent spot instance.** Recover the execution if a spot instance
   was terminated. DVC checkpoints and pipeline stages should be used for
   preserving the state.
3. **Volumes.** Volumes can be attached and reused in instances to minimize
   data cache synchronization time.
4. **Shared volumes.** Separate cloud services (such as Multi-Attach EBS or
   EFS) might be needed for sharing data cache between multiple instances.

## Monitoring 

Additional features might be needed to improve user experienc:
1. **State monitoring.** A user should see if an instance was already
   allocated, what stage is running.
2. **Metrics monitoring.**  Latest metrics monitoring (with some reasonable
   delay - 10-30-60 seconds)


## Resource orchestrators

1. AWS
2. Azure
3. GCP (optional)
4. Remote over SSH
5. Kubernetes (K8S)
6. HashiCorp Nomad (optional)
7. Container services (ECS and Azure/GCP analogs)


# Detailed design

## Iterative Terraform provider

The cloud management part seems like a heavy part of the project (from
  the implementation point of view). However, the problem of executing and
  managing cloud instances was solved in the CML project.

CML separates out the user API part (commands) and the cloud instace
  management code part.
The cloud management is extracted to a separate **Iterative Terraform (TF)
  provider**:
  https://github.com/iterative/terraform-provider-iterative

The Iterative provider is specific to data scientists, not DevOps (TF's
  target audience). It has a bit simplified API and some ML specific
  functionality (some of these are coming) like: recover spot-instance if
  it was deleted or attach a reusable drive.

The Iteratve TF provider can be used from DVC for cloud instace management.
Some Python libraries might be required  for access to GoLang specific
terraform-provider.

Today the provider supports:
- AWS, Azure, K8S orchestrators
- Spot instances and Volumes optimizations.
A few of the initial scenarios can be already implemented with these
functionalities. More functionality is comming.

## Instances configuration

Instance configuration is a definition of an instance and its behavior:
- orchestrator: AWS, GCP, Azure, remote, K8S, ...
- instance type
- volume
- many other options

Instance configuration is well developed in CML project and can be reused.

## Executors and naming

Ideally, a user should be able to define or overwrite executors in config
(`dvc.yml` or `.dvc/config`). `dvc exp run` should execute,
provision/terminate if needed.

The executor definition (`p3.large` AWS instance in `us-west` with `abcdef`
volume attached) should be decoupled from pipeline definition the same way
as remotes are: stage `train` runs on `my-gpu-tesla`, not executor
definition with `p3.large`.

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

## Queue of executors

Queue of executors (like 4 `p3.large` "hot" instances with shared volume)
seems like a very common case and should be natively supported. It should
look like a singele unit (`my-gpu-tesla-hotpool`) from pipeline point of
view.

## Queue of experiments

Executing experiments is a slow and expensive operation and might need more
granular management on the experiment level: create, remove, list, etc.
See here: https://github.com/iterative/dvc/issues/5615

# How We Teach This

We need to introduce a concept **Executor** to DVC.
This convept should not break the compatibility and should not require
significant changes in the docs (except command options).v
However, it will require a new section in the docs to explain the cloud
execution for the users who'd like to use this feature.

# Drawbacks

First, this feature opens new user scenarios which increase the complexity
of DVC, it will require new sections in the docs, etc. However, it seems
like the only way of implementing remote execution if the DVC experements
table is used as a common leddger of ML runs.

It might be beneficial to extract as much of this functionality as possible
to external products and tools (terraform providers for example) to
simplify DVC.

Second, more people might confuse DVC with data engineerings pipeline
execution tools such as AirFlow, Dagster and others.
We should provide a clear guideline when DVC should be used for workflow
execution and when it should not.

# Alternatives

The cloud management logic can be implemented in DVC without using Iterative
FT providers.

# Unresolved questions

TBD
