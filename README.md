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

## install Tekton 
  - operator hub or CLI (minikube)?  
  - (manual Tekton install: kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.16.3/release.yaml)
  - (manual approval)

---

## Fork repository
[Fork this repository](https://github.com/dp-auto/dpxx-ops/generate) from a `Template`. 
  - Ensure you include all branches by tickinging `Include all branches`. 
  - Fork the respository to **your Git user** e.g. `<mygituser>/dp01-ops`

---

## Clone repository to your local machine

Open new Terminal

Set userid, to your userid, e.g. `odowdaibm`

```bash
export GITUSER=odowdaibm
```

```bash
mkdir -p $HOME/git/datapower
cd $HOME/git/datapower
git clone git@github.com:$GITUSER/dp01-ops.git
```

---

## Create catalog sources

```bash
oc apply -f setup/catalog-sources.yaml
```

---

## Create Datapower `dev` namespace 

```bash
oc apply -f setup/namespaces.yaml
```

---

## Install ArgoCD 

```bash
oc apply -f setup/argocd-operator-sub.yaml
```
HERE!
oc get installplan -n dp01-mgmt -o yaml | grep "name: in" | awk '{print$2}' | xarg oc patch installplan install-xxxxx \
    --namespace openshift-logging \
    --type merge \
    --patch '{"spec":{"approved":true}}'

---


## Install Tekton pipelines

```bash
oc apply -f setup/tekton-operator-sub.yaml
```

---

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
