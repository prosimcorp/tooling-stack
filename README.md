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
| 1. Automated K8s      +------>| 2. FluxCD Template +---+----->|  4. Monitoring Stack |
+-----------------------+       +--------------------+   |      +----------------------+
                                                         |
                                                         |      +----------------------+
                                                         +----->|  5. Applications     |
                                                                +----------------------+
```

**[1] Automated K8S:** This stage consists of a Kubernetes cluster which is created and automated using GitOps approach.  
We cover this stage with several repositories ready for different cloud providers:

  * [Automated EKS](https://github.com/prosimcorp/automated-eks)
  * [Automated GKE](https://github.com/prosimcorp/automated-gke)
  * [Automated DO](https://github.com/prosimcorp/automated-do)

> This stage can be covered for different cloud providers or with different technologies. Don't hesitate to code them 
> if needed

**[2] [FluxCD Template](https://github.com/prosimcorp/fluxcd-template)**

**[3] [Tooling Stack](https://github.com/prosimcorp/tooling-stack)**

**[4] [Monitoring Stack](https://github.com/prosimcorp/monitoring-stack)**

## Description

Stack to deploy those tools you need inside the cluster (to deploy applications), which are not included by default
in Kubernetes.

## Motivation

As SREs, we need reliability on the operators we use, and some tools must be configured to work together.
When a new cluster is created, it needs a whole tooling stack deployed.

One of the problem we detected in te past is about SREs finding some bug on their clusters' tooling. The procedure this
case would be to fix it and upgrade the cluster, then apply the changes to the rest of the clusters, spending a lot of 
time of different people to do the task, which is basically replicating the changes. Because of this, we decided to
provide a tooling stack, well tested and integrated where everyone can collaborate once and rise the improvements
easier.

## What you should expect

This stack is made on top of well tested tools, but this does NOT mean zero space for experiments.
Some newer tools will be tested on separated branches if we detect they help the developers or SRE members to work easier 
and faster.

While polishing any tool inside the stack, some parameters could differ between what we consider `develop` environment, 
and `production`. All the changes are always tested on `develop` first, and stabilized before promoting them to `production` 
when changes become solid.

We can not cover the specific use case for all the cloud providers. For that reason we have decided to keep the scope
limited to the major ones, and depending on the time we can spend on this project, increase the number in the future. 
The problem here is not to maintain the agnostic operators or controllers with the most sane config parameters, but 
those that are attached directly to a provider, like CSI controllers when not included by default on some provider, 
that force us to do some magic tricks _**cough! cough! Amazon**_

**Supported cloud providers:**

- Amazon Web Services `aws`
- Google Cloud Platform `gcp`
- DigitalOcean `do`

## Some requirements here

> All the following requirements are created inside Kubernetes on cluster creation if you create the cluster using
> any flavor of [Automated K8S](./README.md#important)

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

### A very special ConfigMap

**A ConfigMap called `cluster-info`**, deployed inside the namespace `kube-system`. This resource is used to customize
some deployments according to the cloud provider, the environment, etc.

Learning by example is funnier, so the content of the configmap is like the following:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-info
  namespace: kube-system
data:
  # The provider. Available values are: AWS, GCP, DO
  provider: AWS 
  # Project ID on GCP, account number on AWS, or project name on DO
  account: "111111111111"
  region: eu-west-1
  name: your-kubernetes-cluster-name
```

## How to deploy manually

> ??????The purpose for this project is not to be deployed by hand but deployed by using GitOps, 
> with tools like FluxCD or ArgoCD.

This repository is composed by deployments for several projects. Some ones can be deployed using `Kustomize` and some 
others that use `Helm`. They are all allocated inside `deploy` directory, and you can deploy them all, step by step, 
by entering one directory, deploying its content and then moving to the next.

## How to deploy using Flux

SREs understand how difficult is to deploy everything, so we have done that task for you to make it easier.
On the root of this repository you can find a directory called `flux` with environments inside, i.e `flux/production` or
`flux/develop`. All you need from now is pointing your Flux deployment to the stack you want to deploy, using a GitRepository
and Kustomization as follows:

```yaml
---
# Point git repository for Tooling Stack
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: tooling-stack
  namespace: flux-system
spec:
  interval: 20s
  ref:
    branch: master
    tag: v0.1.0
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
  path: ./fluxcd/production/do
  prune: false
  sourceRef:
    kind: GitRepository
    name: tooling-stack
    namespace: flux-system
```

Pay special attention to the `spec.path` parameter. There is where the desired stack is defined, setting
the parameter according to the following pattern `./fluxcd/{environment}/{provider}`

**Supported values:**

- Environments: `develop`, `production`
- Cloud providers: `aws`, `gcp`, `do`

Examples:
- Production values on DigitalOcean: 

  ```yaml
  ...
  path: ./fluxcd/production/do
  ```
  
- Develop values on GCP: 
  ```yaml
  ...
  path: ./fluxcd/develop/gcp
  ```

Anyway, as described previously, we did a template for you with larger documentation to make it easier to start:
[FluxCD Template](https://github.com/prosimcorp/fluxcd-template).

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

### _Why isn't External Secrets included?_

We did it at the very beginning, and we still include it in our internal clusters to get the secrets from our vault.
So we recommend it a lot. The key point is that the scope of this repository is to help to automate Kubernetes creation 
stage with the basics, and the basics are different for every company. [External Secrets](https://external-secrets.io/) 
can be configured with a lot of secrets providers, for example, 
[we use Hashicorp Vault for it](https://external-secrets.io/v0.5.7/provider-hashicorp-vault/). 

And now the question: which one do we choose for you?

For these reasons, we finally discarded it when made this repository public. But you can fork this repository and include
External Secrets' deployment with the same design we used, just for your needs. That way we update the rest of the tools, 
and your responsibility is only updating the manifests for External Secrets ????

---

## How to collaborate

> By deploying the stack with `develop` parameters you can test all the stack with the latest changes, and collaborate 
> to improve it.

1. Open an issue and discuss the problem to find together the best way to solve it
2. Fork the repository, create a branch and change inside everything you need
3. Launch a cluster pointing tooling-stack to your repository with the changes
4. Open a Pull Request to merge your code. Don't be dirty with the commits (squash them), be clear with the commit message, etc
5. Cheers ????
