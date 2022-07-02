---
# YAML header
ignore_macros: true
---

# Tooling Stack

## Important
Before diving deeper in the specific documentation of this repository, you must know that is part of an entire flow
composed by several repositories. In the following diagram you will have a better insight. 

```text
                                                                +----------------------+
                                                         +----->|  3. Tooling Stack    |
                                                         |      +----------------------+
                                                         |
+-----------------------+       +--------------------+   |      +----------------------+
| 1. Terraform EKS/GKE  +------>| 2. Flux clusters   +---+----->|  4. Monitoring Stack |
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
[Monitoring Stack](https://gitlab.infrastructure.s73cloud.com/Infrastructure/monitoring-stack)

## Motivation
We need reliability on the operators we use, and some tools must be configured to work together. When a new cluster is 
created, it needs a whole tooling stack deployed.

One of the problem we detected is about SREs finding some bug on its cluster's tooling. The procedure this case would
be to upgrade the template of the cluster and then apply the changes to all the clusters, spending a lot of time of
different people to do the task, which is basically replicating the changes. Because of this, the SRE chapter decided to
provide a tooling stack, well tested and integrated where all the members can collaborate once and rise the improvements
easier.

## What you should expect
This stack will be made on top of well tested tools, but this does NOT mean zero space for experiments. Some newer tools 
will be tested on separated branches if they help the developers or SRE members to work easier and faster. 

It is expected to test new tools if the paradigm changes, but they will be tested in `develop` clusters first.
Deploying the `develop` stack you can test all the stack with latest changes and collaborate to improve it. When changes
become solid, they will be promoted to `production` deployable stack.

## Some requirements here

> All the following requirements are created inside Kubernetes on cluster creation

#### External Secrets

**A first Secret resource with the Vault credentials.** This Secret is needed by External Secrets to get the credentials 
from Vault, and generate all the Secret resources required by later deployments at change, such as developers' applications.

#### A very special ConfigMap

**A ConfigMap called `cluster-info`**, deployed inside the namespace `kube-system`. This resource is used to customize 
some deployments according to the cloud provider, the environment, etc.

Learning by example is funnier, so the content of the configmap is like the following:

```yaml
apiVersion: v1
data:
 provider: AWS
 account: "111111111111"
 region: eu-west-1
 environment: develop
 name: dremel-develop
kind: ConfigMap
metadata:
 name: cluster-info
 namespace: kube-system
```

#### IAM permissions

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
so you can find some examples on the [examples directory](/examples) for the providers we support.

## How to deploy using Flux
SREs understand how difficult is to deploy everything, so we have done that task for you to make it easier.
On the root of this repository you can find a directory called `flux` with environments inside, i.e `flux/production` or
`flux/develop`. All you need from now is pointing your Flux deployment to the stack you want to deploy, using a GitRepository
and Kustomization as follows:

```yaml
---
# Git repository for Kafka
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
  url: ssh://git@gitlab.s73cloud.com:13579/infrastructure/tooling-stack.git
---
# Deploy monitoring stack
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

Anyway, as described previously, we did it for you in the templates, i.e. the template for
[launching EKS clusters](https://gitlab.infrastructure.s73cloud.com/Infrastructure/flux-clusters/templates/base-cloud/-/tree/master/infrastructure/tooling-stack)

## Troubleshooting

#### _DNS Registries are not being updated on the cloud provider_

Due to we patch the External DNS' ServiceAccount on runtime to add a required annotation, its pods sometimes start
before this process has been done. It can be solved just restarting the pods executing the following command:

```console
kubectl delete pods -l="app.kubernetes.io/name=external-dns" -n external-dns
```

---

#### _PVCs in AWS EKS are always in Pending state when using the CSI StorageClass_

Due to we patch the CSI Driver's ServiceAccount on runtime to add a required annotation, its pods sometimes start
before this process has been done. It can be solved just restarting the pods executing the following command:

```console
kubectl rollout restart -n kube-system deployment.apps/ebs-csi-controller
kubectl rollout restart -n kube-system daemonset.apps/ebs-csi-node
```
---

## Full documentation

You have a simple set of instructions on this [README](./docs/README.md) to launch the docs locally

## How to collaborate

1. Create a branch and change inside everything you need
2. Launch a cluster pointing tooling-stack to this repository but to your branch
3. Open a Pull Request to merge your code. Don't be dirty with the commits (squash them), be clear with the commit message, etc
