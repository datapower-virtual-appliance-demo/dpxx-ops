## Overview

This tutorial demonstrates how to do CICD with DataPower using GitOps on Kubernetes. 

In this tutorial, you will:

1. Create a Kubernetes cluster and image registry, if required.
2. Install ArgoCD applications to manage cluster deployment of DataPower-related Kubernetes resources.
3. Create an operational repository to store DataPower resources that are deployed to the Kubernetes cluster.
4. Create a source Git repository to store the configuration and development artefacts for a virtual DataPower appliance.
5. Run a Tekton pipleline to build, test, version and deliver the DataPower-related resources ready for deployment.
6. Gain experience with the IBM-supplied DataPower operator and container.

At the end of this tutorial, you will have a solid foundation of GitOps and CICD for DataPower in a Kubernetes environment.

---

## Introduction 

The following diagram shows a GitOps CICD pipeline for DataPower:

![diagram1](./docs/images/diagram1.drawio.png)

Notice: 

- The git repository `dp01-src` holds the source configuration for the DataPower appliance `dp01`.
- The `dp01-src` repository also holds the source for a multi-protocol gateway on `dp01`.
- A Tekton pipeline uses the `dp01-src` repository to build, package, test, version and deliver resources that define the `dp01` DataPower appliance.
- If the pipeline is successful, then the YAMLs that define `dp01` are stored in the operational repository `dp01-ops` and the container image for `dp01` is stored in an image registry.
- Shortly after the changes are committed to the git repository, an ArgoCD application detects the updated YAMLs. It applies them to the cluster to update the running `dp01` DataPower appliance.

This tutorial will walk you through the process of setting up this configuration:
- Step 1: Follow the instructions in this README to set up your cluster, ArgoCD and the `dp01-ops` repository.
- Step 2: Follow [these instructions](https://github.com/dp-auto/dpxx-src#readme) to create the `dp01-src` respository, run a tekton pipeline to populate it, and interact with the new or updated DataPower appliance `dp01`.

---

## Install Kubernetes

Cover Minikube OCP options -->links

---

## Fork repository
[Fork this repository](https://github.com/dp-auto/dpxx-ops/generate) from a `Template`. 
  - In the `Repsoitory name` field, specify `dp01-ops`

This repsoitory will be cloned to the specified GitHub account.

---

## Clone repository to your local machine

We're going to use the contents of this repository to configure our cluster for CICD and GitOps. We're going to use a copy of the `dp01-ops` respository on our local machine to do this.

Open new Terminal window. 

In it, store your Git userId in the `GITUSER` environment variable, e.g. `odowdaibm`

```bash
export GITUSER=odowdaibm
```

Now clone the respository to your local machine. It's best practice to store all git repositories of a common root folder called `git`. We will keep both the dp01 source and operation repositories under a subfolder `datapower`.

Issue the following commands to optionally create this folder structure, and clone the `dp01-ops` repository.

```bash
mkdir -p $HOME/git/datapower
cd $HOME/git/datapower
git clone git@github.com:$GITUSER/dp01-ops.git
```

---

## Explore the `dp01-ops` repository

The contents of the `dp01-ops` repository will be synchronized with the Kubernetes cluster such that every object in the repository wil be deployed to the cluster. Let's briefly explore the contents of this repository.

Issue the following command:

```bash
cd dp01-ops
cat setup/namespaces.yaml
```

which shows the following YAMLs.

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: dp01-mgmt
  labels:
    name: dp01-mgmt
---
kind: Namespace
apiVersion: v1
metadata:
  name: dp01-dev
  labels:
    name: dp01-dev
```

These YAMLs will define two namespaces `dp01-mgmt` and `dp01-dev` which will be used to store Kubernetes resources for this tutorial. We'll explore the respository in more detail as we proceed through the tutorial.

---

## Create Datapower `dev` namespace 

Let's use this YAML to define two namespaces in our cluster:

```bash
oc apply -f setup/namespaces.yaml
```

which will create the `dp01-mgmt` and `dp01-dev` namespaces in the cluster.

```bash
namespace/dp01-mgmt created
namespace/dp01-dev created
```

We'll see how: 

- the `dp01-mgmt` namespace is used to store generic Kubernetes resources catalog sources and operators.
- the `dp01-dev` namespace is used to store specific Kubernetes resources relating to `dp01`.

As the turorial proceeds, we'll see how the contents of the `dp01-ops` repository **fully** defines the contents of all resources relating to our DataPower deployment. Moreover, we're going to set up the cluster such it is **automatically** updated whenever this `dp01-ops` repository changes. This concept is called **continuous deployemnt** and we'll use ArgoCD to acheive it.

---

## Create ArgoCD subscription 

Let's install ArgoCD to enable continuous deployment: 

Use the following command to create a subscription for ArgoCD:

```bash
oc apply -f setup/argocd-operator-sub.yaml
```

which will create a subscription for ArgoCD: 

```bash
subscription.operators.coreos.com/openshift-gitops-operator created
```

This subscription enables the cluster to keep up-to-date with new version of ArgoCD. Each release has an [install plan](https://olm.operatorframework.io/docs/concepts/olm-architecture/) that is used to maintain it. In what might seem like a contradiction, our subscription creates an install plan that requires manual approval; we'll understand why a little later. 

Explore the subscription using the following command: 

```bash
cat setup/argocd-operator-sub.yaml
```

which details the subscription.

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Manual
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

[Learn more about subscriptions](https://olm.operatorframework.io/docs/concepts/crds/subscription/). 

## Approve ArgoCD install plan


Let's find our install plan and approve it.

oc get installplan -n dp01-mgmt -o yaml | grep "name: in" | awk '{print$2}' | \
xarg oc patch installplan 
 --namespace openshift-logging \
 --type merge \
 --patch '{"spec":{"approved":true}}'

---

## Install Tekton pipelines

```bash
oc apply -f setup/tekton-operator-sub.yaml
```

Manual Tekton install: 
```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.16.3/release.yaml)
```

---

## Create catalog sources

```bash
oc apply -f setup/catalog-sources.yaml
```

## Install DataPower operator

```bash
oc apply -f setup/dp-operator-sub.yaml
```

---

## Manually approve in OCP UI

Fix this with oc patch

```bash
oc get installplan -n dp01-mgmt -o yaml | grep "name: in" | awk '{print$2}' | xarg oc patch installplan install-xxxxx \
    --namespace openshift-logging \
    --type merge \
    --patch '{"spec":{"approved":true}}'
```

```
oc patch installplan install-xxxxx \
    --namespace openshift-logging \
    --type merge \
    --patch '{"spec":{"approved":true}}'
```

---

## Generate ssh keys for GitHub access

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ./.ssh/id_rsa -q -N ""
```

```bash
ssh-keyscan -t rsa github.com | tee ./.ssh/github-key-temp | ssh-keygen -lf - && cat ./.ssh/github-key-temp >> ./.ssh/known_hosts
```

```bash
oc create secret generic dp01-ssh-credentials -n dp01-dev --from-file=id_rsa=./.ssh/id_rsa --from-file=known_hosts=./.ssh/known_hosts --from-file=./.ssh/config --dry-run=client -o=yaml
```

```bash
oc create secret generic dp01-ssh-credentials -n dp01-dev --from-file=id_rsa=./.ssh/id_rsa --from-file=known_hosts=./.ssh/known_hosts --from-file=./.ssh/config --dry-run=client -o yaml > dp-git-credentials.yaml
```

```bash
oc apply -f dp-git-credentials.yaml
```

---

## Copy ssh credential into Github

Need to add ssh public key to GitHub to authorise access:

```bash
https://github.com/settings/keys
```

![diagram2](./docs/images/diagram2.png)


Copy public key to clipboard

```bash
pbcopy < ./.ssh/id_rsa.pub
```

Click on `New SSH Key`

* Add name `dp01 SSH key`
* Paste key into box

![diagram3](./docs/images/diagram3.png)

* Hit `Add SSH key` button

---

## Patch `pipeline` serviceaccount

```bash
oc patch serviceaccount pipeline \
    --namespace dp01-dev \
    --type merge \
    --patch '{"secrets":[{"name":"dp01-ssh-credentials"}]}'
```

---

