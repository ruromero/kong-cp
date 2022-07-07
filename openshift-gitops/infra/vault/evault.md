- The vault app can be borrowed from multi cloud validated pattern project (On-Premise) 
- Set up the HA vault with consul as backend (consul may be on hosted on a cloud provider) (Cloud Provider)
- The following set up is for demonstration only
```
# Waiting for fix - https://github.com/hashicorp/consul-k8s/pull/1307/files
# Not Ideal (Only for demo)
oc adm policy add-scc-to-group privileged system:serviceaccounts:vault
oc adm policy add-scc-to-group anyuid system:serviceaccounts:vault
helm install consul hashicorp/consul --values helm-consul-values.yml -n vault
helm install vault hashicorp/vault --values helm-vault-values.yml -n vault
kubectl -n kong exec vault-0 -- vault status
kubectl -n vault exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
cat cluster-keys.json | jq -r ".unseal_keys_b64[]"
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]") # Do not run an unsealed Vault in production with a single key share and a single key threshold. This approach is only used here to simplify the unsealing process for this demonstration.
kubectl -n vault exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
kubectl -n vault exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY
kubectl -n vault exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY
cat cluster-keys.json | jq -r ".root_token"
kubectl -n vault exec --stdin=true --tty=true vault-0 -- /bin/sh
vault login
kubectl exec vault-0 -n vault -- vault auth enable kubernetes

oc -n vault rsh vault-0

vault write auth/kubernetes/config \
issuer="https://kubernetes.default.svc.cluster.local" \
token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

vault secrets enable -path=avp -version=2 kv

vault kv put avp/test sample=<<license.json>

cat << EOF > /tmp/policy.hcl 
path "avp/data/test" { capabilities = ["read"] } 
EOF

vault policy write argocd-repo-server /tmp/policy.hcl

vault write auth/kubernetes/role/argocd-repo-server \
    bound_service_account_names=argocd-repo-server \
bound_service_account_namespaces=openshift-gitops policies=argocd-repo-server

helm uninstall vault -n vault
helm uninstall consul -n vault
oc adm policy remove-scc-from-group privileged system:serviceaccounts:vault
oc adm policy remove-scc-from-group anyuid system:serviceaccounts:vault
```