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

## 
