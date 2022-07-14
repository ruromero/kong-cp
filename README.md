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

### Add dataplane cluster
```bash
export ARGOCD_SERVER_URL=$(oc get routes -n openshift-gitops | grep redhat-kong-gitops-server | awk '{print $2}')
argocd login $ARGOCD_SERVER_URL
argocd cluster add dp
argocd cluster add cp
```

### Create the project for control plane and data plane
```bash
oc apply -f openshift-gitops/infra/project.yaml
```

### Install the hasicorp vault
Refer [Vault setup](/openshift-gitops/infra/vault/evault.md) for basic dev setup of vault

### Deploy control plane
```bash
oc apply -f openshift-gitops/overlays/cp/
```

### Deploy data plane
```bash
oc apply -f openshift-gitops/overlays/dp/
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

## Validate Data Plane deployment

Check the clustering status
```bash
http `oc get route -n kong kong-kong-admin --template='{{ .spec.host }}'`/clustering/status
```

output
```bash
HTTP/1.1 200 OK
access-control-allow-origin: *
cache-control: private
content-length: 2
content-type: application/json; charset=utf-8
date: Wed, 13 Jul 2022 21:15:38 GMT
deprecation: true
server: kong/2.8.1.1-enterprise-edition
set-cookie: 9da87f6e8821b5f9e46a0f05aee42078=0cc26972bced6fbd36b0fb312aa44651; path=/; HttpOnly
x-kong-admin-latency: 9
x-kong-admin-request-id: DQrbDNP3jBIr6aDcxE5GnJrfewL8X5fb

{}
```

Check Prom Targets (or check in the browser)
```bash
curl $(oc get routes -n kong --context dp prometheus-service --template='{{ .spec.host }}' )/api/v1/targets | jq '.data.activeTargets[] | {health,scrapePool}'
```

output
```json
{
  "health": "up",
  "scrapePool": "serviceMonitor/kong/kong-status/0"
}
```

Check Grafana

Get password
```
oc get secret -n kong --context dp grafana-admin-credentials  --template='{{ .data.GF_SECURITY_ADMIN_PASSWORD }}' | base64 -d
```

Go to grafana route (takes a while to come up sometimes)
```bash
oc get route -n kong --context dp grafana-service --template='{{ .spec.host }}'
```

Login with user `admin` and the password from two steps ago.

Check patch results:

Check ingress host
```bash
oc get ing demo-app-ingress -n bookinfo --context dp -ojsonpath='{ .spec.rules[0].host }'
```

output
```bash
demo-app-kong.api.mso-test-cwylie.fsi-env2.rhecoeng.com
```

Check the KeycloakClient redirectUri
```bash
oc get keycloakclient kuma-demo-client -n keycloak --context dp -ojsonpath='{ .spec.client.redirectUris[0] }'
```

output
```bash
http://demo-app-kong.api.mso-test-cwylie.fsi-env2.rhecoeng.com/*
```

Check the KeycloakClient rootUrl
```bash
oc get keycloakclient kuma-demo-client -n keycloak --context dp -ojsonpath='{ .spec.client.rootUrl }'
```

output
```bash
http://demo-app-kong.api.mso-test-cwylie.fsi-env2.rhecoeng.com
```

Check the KongPlugin:
```bash
oc get kongplugin keycloak-auth-plugin -n bookinfo --context dp --template='{{ .config.issuer }}'
```

output
```
https://keycloak-keycloak.com/auth/realms/kong
```