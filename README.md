# dp-auto-ops
Operational repository to deploy to the cluster

## Fork template repository

## Clone template repository

## Create catalog sources

```bash
oc apply -f setup/catalog-sources.yaml
```

## Create Datapower `dev` namespace 

```bash
oc apply -f setup/dev-namespace.yaml
```

## Install DataPower operator

```bash
oc apply -f setup/dp-operator-sub.yaml
```

* Manually approve in OCP UI

* Fix this with oc patch

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

## 
