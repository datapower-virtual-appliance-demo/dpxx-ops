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
  name: dp01-dev
  labels:
    name: dp01-dev
```

These YAMLs will define the `dp01-dev` namespace which will be used to store Kubernetes resources for this tutorial. We'll explore the contents of this namespace throughout the tutorial.

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

- the `dp01-mgmt` namespace is used to store generic Kubernetes resources relating to DataPower.
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

which details the subscription:

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

See if you can understand each YAML node, referring to [subscriptions](https://olm.operatorframework.io/docs/concepts/crds/subscription/) if you need to learn more. 

## Approve ArgoCD install plan

Let's find our install plan and approve it.

```bash
oc get installplan -n openshift-operators | grep "openshift-gitops-operator" | awk '{print $1}' | \
xargs oc patch installplan \
 --namespace openshift-operators \
 --type merge \
 --patch '{"spec":{"approved":true}}'
```

which will approve the install plan

```bash
installplan.operators.coreos.com/install-xxxxx patched
```

where `install-xxxxx` is the name of the ArgoCD install plan.

## Verify ArgoCD installation

ArgoCD will now install; let's verify the installation has completed successfully by examining the ClusterServiceVersion (CSV) for ArgoCD. A CSV is created for each installation - it holds the exact versions of all dependent software and relevant RBAC permissions.

```bash
oc get clusterserviceversion -n openshift-gitops
```

```bash
NAME                               DISPLAY                    VERSION   REPLACES                                          PHASE
openshift-gitops-operator.v1.5.7   Red Hat OpenShift GitOps   1.5.7     openshift-gitops-operator.v1.5.6-0.1664915551.p   Succeeded
```

Feel free to explore this CSV with, replacing `x.y.z` with the installed version of ArgoCD:

```bash
oc describe csv openshift-gitops-operator.vx.y.z -n openshift-operators
```

## Minor modifications to ArgoCD

```bash
oc patch argocd openshift-gitops  \
 --namespace openshift-gitops \
 --type merge \
 --patch '{"spec":{"applicationInstanceLabelKey":"argocd.argoproj.io/instance"}}'
```

## Add IBM catalog sources

The final operator we need to add to the cluster is the DataPower operator. It is installed from a specific IBM [catalog source](https://olm.operatorframework.io/docs/concepts/crds/catalogsource/), so we first need to add the catalog source to the cluster.

Issue the following command:

```bash
oc apply -f setup/catalog-sources.yaml
```

which will add the sources defined in this YAML to the cluster:

```bash
catalogsource.operators.coreos.com/ibm-operator-catalog created
```

Feel free to examine the catalog source YAML:

```bash
cat setup/catalog-sources.yaml
```

Notice how two catalog sources are added.

---

## Add DataPower operator group

We are going to install the DataPower operator into its own namespace, `dp01-mgmt`. To allow it to access the `dp01-dev` namespace, we create an operator group.

Issue the following command to create the operator group `dp-operator-group`:

```bash
oc apply -f setup/dp-operator-group.yaml
```

which you will see created.

```bash
operatorgroup.operators.coreos.com/dp-operator-group created
```

You can examine this operator group with the following command:

```bash
cat setup/dp-operator-group.yaml
```

which will show you the details of the operator group:

```yaml
```

Notice how this operator can control the `dp01-dev` namespace, where we are going to initially deploy the `dp01` DataPower appliance.

---

## Install DataPower operator

Now we've added these catalog sources, we can install the DataPower operator; we're familar with the process -- it's the same as ArgoCD.

Issue the following command:

```bash
oc apply -f setup/dp-operator-sub.yaml
```

```bash
subscription.operators.coreos.com/datapower-operator created
```

Explore the subscription using the following command: 

```bash
cat setup/dp-operator-sub.yaml
```

which details the subscription:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/datapower-operator.dp01-ns: ''
  name: datapower-operator
  namespace: openshift-operators
spec:
  channel: v1.6
  installPlanApproval: Manual
  name: datapower-operator
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
  startingCSV: datapower-operator.v1.6.3
```

Notice how this operator is installed in the `dp01-mgmt` namespace. Note also the use of `channel` and `startingCSV` to be precise about the exact version of the DataPower operator to be installed.

## Approve and verify DataPower install plan

Let's find our install plan and approve it.

```bash
oc get installplan -n openshift-operators | grep "datapower-operator" | awk '{print $1}' | \
xargs oc patch installplan \
 --namespace openshift-operators \
 --type merge \
 --patch '{"spec":{"approved":true}}'
```

which will approve the install plan:

```bash
installplan.operators.coreos.com/install-xxxxx patched
```

where `install-xxxxx` is the name of the DataPower install plan.

Again, feel free to verify the DataPower installation with the following commands:

```bash
oc get clusterserviceversion -n dp01-mgmt
```

Replace `x.y.z` with the installed version of DataPower in the following command:

```bash
oc describe csv datapower-operator.vx.y.z -n openshift-operators
```

---

## Install Tekton pipelines

Our final task is to install Tekton.  With it, we can create pipelines that populate the operational repository `dp01-ops` using the DataPower configuration and development artefacts stored in `dp01-src`. Once populated by Tekton, ArgoCD will then synchronize these artefacts with the cluster to ensure the cluster is running the most up-tp-date version of `dp01`. 

Issue the following command to create a subscription for Tekton:

```bash
oc apply -f setup/tekton-operator-sub.yaml
```

which will create a subscription:

```bash
subscription.operators.coreos.com/openshift-pipelines-operator created
```

Again, this subscription enables the cluster to keep up-to-date with new version of Tekton. 

Explore the subscription using the following command: 

```bash
cat setup/tekton-operator-sub.yaml
```

which details the subscription:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  channel:  stable
  installPlanApproval: Manual
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

Manual Tekton install: 
```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.16.3/release.yaml)
```
---

## Approve and verify Tekton install plan

Let's find our install plan and approve it.

```bash
oc get installplan -n openshift-operators | grep "openshift-pipelines-operator" | awk '{print $1}' | \
xargs oc patch installplan \
 --namespace openshift-operators \
 --type merge \
 --patch '{"spec":{"approved":true}}'
```

which will approve the install plan

```bash
installplan.operators.coreos.com/install-xxxxx patched
```

where `install-xxxxx` is the name of the Tekton install plan.

Again, feel free to verify the Tekton installation with the following commands:

```bash
oc get clusterserviceversion -n openshift-pipelines
```

(replacing `x.y.z` with the installed version of Tekton)

```bash
oc describe csv openshift-pipelines-operator-rh.vx.y.z -n openshift-operators
```

---

## Generate ssh keys for GitHub access

To allow Tekton to access GitHub, specifically to create YAMLs in the `dp01-ops` repository, we need to set up appropriate SSH keys for access.

Issue the following command to create an SSH key pair:

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ./.ssh/id_rsa -q -N ""
```

Issue the following command to create a `known_hosts` file for SSH access:

```bash
ssh-keyscan -t rsa github.com | tee ./.ssh/github-key-temp | ssh-keygen -lf - && cat ./.ssh/github-key-temp >> ./.ssh/known_hosts
```

Issue the following command to create a secret containing the  SSH private key and `known_hosts` file:

```bash
oc create secret generic dp01-ssh-credentials -n dp01-dev --from-file=id_rsa=./.ssh/id_rsa --from-file=known_hosts=./.ssh/known_hosts --from-file=./.ssh/config --dry-run=client -o yaml > .ssh/dp-git-credentials.yaml
```

Issue the following command to create this secret in the cluster:

```bash
oc apply -f .ssh/dp-git-credentials.yaml
```

Finally, add this secret to the `pipeline` service account to allow it to use `dp-1-ssh-credentials` secret to access GitHub.

```bash
oc patch serviceaccount pipeline \
    --namespace dp01-dev \
    --type merge \
    --patch '{"secrets":[{"name":"dp01-ssh-credentials"}]}'
```

---

## Copy ssh credential into Github

To allow the Tekton pipeline to push the generated DataPower Kubernetes resource YAMLs to the `dp01-ops` repository, we need to add the public key we've just generated to GitHub. 

Use the following link in a browser to access the GitHub user interface to add an SSH public key:

```bash
https://github.com/settings/keys
```

You'll see the following page:

![diagram2](./docs/images/diagram2.png)

Copy your public key to the clipboard:

```bash
pbcopy < ./.ssh/id_rsa.pub
```

Click on `New SSH Key`, and complete the following details:

* Add name `dp01 SSH key`
* Paste key into box

![diagram3](./docs/images/diagram3.png)

* Hit `Add SSH key` button

The Tekton pipeline now has access to your GitHub. 

*(Might be better to use access tokens, to limit scope... consider as change.)*

---

## An ArgoCD application to manage dp01

Finally, we're going to create an ArgoCD application to manage `dp01`.

```bash
cat environments/dev/argocd/dp01.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dp01-argo
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/sync-wave: "100"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    namespace: dp01-dev
    server: https://kubernetes.default.svc
  project: default
  source:
    path: environments/dev/dp01/
    repoURL: https://github.com/dp-auto/dp01-ops.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - Replace=true
```

In your GitHub repository, replace `dp-auto` with your Git user id and deploy it to the cluster:
(**Add push commands**)

Notice how the this Argo application will be monitoring GitHub for resources to deploy to the cluster:

```yaml
  source:
    path: environments/dev/dp01/
    repoURL: https://github.com/odowdaibm/dp01-ops.git
    targetRevision: main
```

See how:
  - `repoURL: https://github.com/odowdaibm/dp01-ops.git` identifies the repository where the YAMLs are located
  - `targetRevision: main` identifies the branch within the repository
  - `path: environments/dev/dp01/` identifies the folder within the repository

## Deploy dp01-argo to the cluster

Let's deploy this ArgoCD application to the cluster:

```bash
oc apply -f environments/dev/argocd/dp01.yaml
```

which will complete with:

```bash
application.argoproj.io/dp01-argo created
```

We now have an ArgoCD application monitoring our repository.

## View dp01-argo in ArgoCD UI

We can use the ArgoCD UI to look at the `dp01-argo` application and the resources it is managing:

Issue the following command to identify the URL for the ArgoCD login page:

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{"https://"}{.spec.host}{"\n"}' 
```

which will return a URL similar to this:

```bash
https://openshift-gitops-server-openshift-gitops.vpc-mq-cluster1-d02cf90349a0fe46c9804e3ab1fe2643-0000.eu-gb.containers.appdomain.cloud
```

Issue the following command to determin the ArgoCD `password` for the `admin` user: 

```bash
oc extract secret/openshift-gitops-cluster -n openshift-gitops --keys="admin.password" --to=-
```

Login to ArgoCD with `admin` and `password`.

You will see the following screen:

![diagram4](./docs/images/diagram4.png)

The ArgoCD application `dp01-argo` is monitoring the `https://github.com/ODOWDAIBM/dp01-ops` repository.

We will now run the Tekton pipeline to populate this repository.

---

## Congratulations

You've set up your cluster for DataPower.  Let's run a pipeline to populate the repository.  

Continue [here](https://github.com/dp-auto/dpxx-src#readme): 

---

