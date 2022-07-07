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

### Create the bookinfo app
```bash
oc create ns bookinfo # add this as pre-req in data plane
oc apply -f openshift-gitops/bookinfo/app.yaml # Do Initialize the roles first
oc -n bookinfo port-forward svc/productpage 9080:9080
```

- TODO
    - Iteration 1
        - [] PostInstall for CP
        - [] Data Plane
        - [] kustomize in openshift-gitops/
        - [] Monitoring in data plane
        - [X] Bookinfo app
        - [] app of apps (helm chart)
        - [] Enterprise Vault - Retrieving secrets from the vault
    - Iteration 2
        - [] Automation for Setup, initialize and unsealing of vault.




