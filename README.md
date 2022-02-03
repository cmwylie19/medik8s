# Medik8s
- [install](#install)
- [Demo](#demo)
- [uninstall](#uninstall)

## Install
Install medik8s through the operator
```
k create ns poison-pill
k create -f install/medik8s-operator.yaml
```

## Demo
Install CockroachDB in a Deployment with a single replica backed by a PVC and PV with accessMode set to `RWO`
```
k create ns cockroachdb
k create -f install/standalone-crdb.yaml
```

## Uninstall
Clean up medik8s and CockroachDB
```
k delete -f install/
k delete -f ds -n poison-pill --all --force --grace-period=0
k delete ns poison-pill --wait
```