### Install the operator

```bash
oc apply -f openshift-gitops/infra/gitops-operators.yaml
```
### Monitor the operator install
```bash
oc get csv -n openshift-gitops -w
```
### Delete the default app installed by the operator
```bash
oc delete argocd openshift-gitops -n openshift-gitops
```
### Deploy a argocd app with vault plugin
```bash
oc apply -f openshift-gitops/infra/argocd.yaml
```

### Get the route and password of argocd gui
```bash
oc get routes -n openshift-gitops redhat-kong-gitops-server --template='{{ .spec.host }}'
```
```bash
oc get secret -n openshift-gitops redhat-kong-gitops-cluster -ojsonpath='{.data.admin\.password}' | base64 -d
```

### Install the hasicorp vault
Refer [Vault setup](/openshift-gitops/vault.md) for basic dev setup of vault

### Create the apps
```bash
oc apply -f openshift-gitops/overlays/cp/
```

- TODO
    - [] PostInstall for CP
    - [] Data Plane
    - [] kustomize in openshift-gitops/
    - [] Monitoring in data plane
    - [] Bookinfo app
    - [] app of apps
    - [] helm chart of app of apps
    - [] Enterprise Vault

