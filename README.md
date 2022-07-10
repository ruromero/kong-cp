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
Refer [Vault setup](/openshift-gitops/infra/vault/evault.md) for basic dev setup of vault

### Create the apps
```bash
oc apply -f openshift-gitops/overlays/cp/
```

## Validate Control Plane deployment

### Check the admin endpoint is available
```bash
http `oc get route -n kong kong-kong-admin --template='{{.spec.host}}'` | jq .version
```

```
"2.8.1.1-enterprise-edition"
```

### Check the cluster-urls
```bash
oc get cm cluster-urls -n kong -oyaml
```

```
apiVersion: v1
data:
  CLUSTER_TELEMETRY_URL: aada71fce488d417b9db340ce87c9b4b-763449203.us-west-1.elb.amazonaws.com
  CLUSTER_URL: aff89d1140e6b41299f85bd663592c20-1822993472.us-west-1.elb.amazonaws.com
kind: ConfigMap
metadata:
  creationTimestamp: "2022-07-02T12:00:54Z"
  name: cluster-urls
  namespace: kong
  resourceVersion: "4008563"
  uid: 309e0540-e8e9-470a-b795-b00816206273
```

### Check the logs of patch deploy job
```bash
oc logs -n kong -l job-name=patch-deploy
```
```
redhat-kong-gitops-server-openshift-gitops.apps.cwylie-us-west-1b.kni.syseng.devcluster.openshift.com
Be4OqiXClFN7paoWDIAPZj1tUnfsK06J
'admin:login' logged in successfully
Context 'redhat-kong-gitops-server-openshift-gitops.apps.cwylie-us-west-1b.kni.syseng.devcluster.openshift.com' updated
time="2022-07-02T12:47:20Z" level=info msg="Resource 'kong-kong' patched"
```


## Create the bookinfo app
```bash
oc create ns bookinfo # add this as pre-req in data plane
oc apply -f openshift-gitops/bookinfo/app.yaml # Do Initialize the roles first
oc -n bookinfo port-forward svc/productpage 9080:9080
```

- TODO
    - Iteration 1
        - [X] PostInstall for CP
        - [] Data Plane
        - [] kustomize in openshift-gitops/
        - [] Monitoring in data plane
        - [X] Bookinfo app
        - [] app of apps (helm chart)
        - [X] Enterprise Vault - Retrieving secrets from the vault
    - Iteration 2
        - [] Automation for Setup, initialize and unsealing of vault.




