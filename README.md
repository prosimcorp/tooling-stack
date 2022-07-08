# Tooling Stack

## Important

> <div style="color: orange">ATTENTION</div>
> Before diving deeper in the specific documentation of this repository, you must know that is part of an entire flow
> composed by several repositories. In the following diagram you will have a better insight.

```text
                                                                +----------------------+
                                                         +----->|  3. Tooling Stack    |
                                                         |      +----------------------+
                                                         |
+-----------------------+       +--------------------+   |      +----------------------+
| 1. Automated EKS      +------>| 2. FluxCD Template +---+----->|  4. Monitoring Stack |
+-----------------------+       +--------------------+   |      +----------------------+
                                                         |
                                                         |      +----------------------+
                                                         +----->|  5. Applications     |
                                                                +----------------------+
```

## Description

Stack to deploy those tools you need inside the cluster (to deploy applications), which are not included by default
in Kubernetes.

This is the stack you need to be deployed before the
[Monitoring Stack](https://github.com/prosimcorp/monitoring-stack)

## Motivation

As SREs, we need reliability on the operators we use, and some tools must be configured to work together.
When a new cluster is created, it needs a whole tooling stack deployed.

One of the problem we detected in te past is about SREs finding some bug on their clusters' tooling. The procedure this
case would be to fix it and upgrade the cluster, then apply the changes to the rest of the clusters, spending a lot of time of
different people to do the task, which is basically replicating the changes. Because of this, we decided to
provide a tooling stack, well tested and integrated where everyone can collaborate once and rise the improvements
easier.

## What you should expect

This stack is made on top of well tested tools, but this does NOT mean zero space for experiments.
Some newer tools will be tested on separated branches if they help the developers or SRE members to work easier and faster.

While polishing any tool inside the stack, some parameters could change between what we consider `develop` environment, and
`production`: all the changes are always tested on `develop` environment first and stabilized before promoting them to `production`
when changes become solid.

Deploying the stack with `develop` parameters you can test all the stack with latest changes and collaborate to improve it.

## Some requirements here

> All the following requirements are created inside Kubernetes on cluster creation if you create the cluster using
> [Automated EKS](https://github.com/prosimcorp/automated-eks)

### External Secrets

**A first Secret resource with the Vault credentials.** This Secret is needed by External Secrets to get the credentials
from Vault, and generate all the Secret resources required by later deployments at change, such as developers' applications.

### A very special ConfigMap

**A ConfigMap called `cluster-info`**, deployed inside the namespace `kube-system`. This resource is used to customize
some deployments according to the cloud provider, the environment, etc.

Learning by example is funnier, so the content of the configmap is like the following:

```yaml
apiVersion: v1
data:
  provider: AWS
  account: "111111111111" # Project ID or Account depending on the provider
  environment: develop # The alias for the environment to name the project
  region: eu-west-1
  name: your-cluster-name
kind: ConfigMap
metadata:
  name: cluster-info
  namespace: kube-system
```

### IAM permissions

**Some deployments need IAM permissions to be able to manage resources on the cloud providers**.

Examples for this are:

- `external-dns`: which manages DNS registries
- `CSI driver`: which manage the volumes or the snapshots

These permissions are populated to Kubernetes using
[AWS IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) or
[GCP Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity).
This means that some Kubernetes ServiceAccounts need to have an annotation to link IAM permissions
(usually created using Terraform) to Kubernetes.

You don't have to care about these annotations because they are automatically crafted and configured using the values
from `cluster-info` ConfigMap.

For doing this kind on "magic" we rely on naming convention, so the following example illustrates better
the pattern: `{CLUSTER_NAME}-{PACKAGE_NAME}`

For example, the expected names for AWS IAM Roles or GCP IAM ServiceAccounts, are the following:

- **External DNS:** `{CLUSTER_NAME}-external-dns`
- **CSI Driver:** `{CLUSTER_NAME}-csi-driver`

> To be clear, the suffix is exactly the name of the directory for the package inside `deploy/`

As you know in this point, we have automated everything, but IAM permissions must be already created,
so you can find some examples on the [examples directory](./examples/README.md) for the providers we support.

## How to deploy using Flux

SREs understand how difficult is to deploy everything, so we have done that task for you to make it easier.
On the root of this repository you can find a directory called `flux` with environments inside, i.e `flux/production` or
`flux/develop`. All you need from now is pointing your Flux deployment to the stack you want to deploy, using a GitRepository
and Kustomization as follows:

```yaml
---
# Git repository for Tooling Stack
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: tooling-stack
  namespace: flux-system
spec:
  interval: 20s
  ref:
    branch: master
    tag: v0.3.0
  secretRef:
    name: flux-system
  url: ssh://git@github.com/prosimcorp/tooling-stack.git
---
# Deploy Tooling Stack
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: tooling-stack
  namespace: flux-system
spec:
  interval: 10m
  retryInterval: 1m
  path: ./flux/develop
  prune: true
  sourceRef:
    kind: GitRepository
    name: tooling-stack
    namespace: flux-system
```

Pay special attention to the `spec.path` because there is where the desired stack will be defined, setting
the parameter to `./flux/develop` or to `./flux/production`

Anyway, as described previously, we did a template for you, with everything already configured on
[FluxCD Template](https://github.com/prosimcorp/fluxcd-template), to make it easier to start.

## Troubleshooting

### _DNS Registries are not being updated on the cloud provider_

Due to we patch the External DNS' ServiceAccount on runtime to add a required annotation, its pods sometimes start
before this process has been done. It can be solved just restarting the pods executing the following command:

```console
kubectl delete pods -l="app.kubernetes.io/name=external-dns" -n external-dns
```

---

### _PVCs in AWS EKS are always in Pending state when using the CSI StorageClass_

Due to we patch the CSI Driver's ServiceAccount on runtime to add a required annotation, its pods sometimes start
before this process has been done. It can be solved just restarting the pods executing the following command:

```console
kubectl rollout restart -n kube-system deployment.apps/ebs-csi-controller
kubectl rollout restart -n kube-system daemonset.apps/ebs-csi-node
```

---

## How to collaborate

1. Open an issue and discuss about the problem to find together the best way to solve it
2. Fork the repository, create a branch and change inside everything you need
3. Launch a cluster pointing tooling-stack to your repository with the changes
4. Open a Pull Request to merge your code. Don't be dirty with the commits (squash them), be clear with the commit message, etc
5. Cheers 🎉
