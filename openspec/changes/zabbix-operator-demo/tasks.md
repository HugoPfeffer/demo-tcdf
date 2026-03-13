## 1. Namespace

- [ ] 1.1 Create `argocd/namespace.yaml` — `v1/Namespace` for `zabbix-demo` with sync-wave `0`
  - Verify: `oc get namespace zabbix-demo`

## 2. RBAC and SCC Grants

- [ ] 2.1 Create `argocd/rbac.yaml` — `Role` granting `use` on `anyuid` SCC + `RoleBinding` to Zabbix service accounts in `zabbix-demo`, sync-wave `1`
  - Verify: `oc get role,rolebinding -n zabbix-demo`

## 3. TLS Certificate Issuer

- [ ] 3.1 Create `argocd/cluster-issuer.yaml` — `cert-manager.io/v1 ClusterIssuer` (self-signed), sync-wave `1`
  - Verify: `oc get clusterissuer -o wide` shows `Ready: True`

## 4. Zabbix Server Stack

- [ ] 4.1 Create `argocd/zabbix-full.yaml` — `zabbix.com/v1alpha1 ZabbixFull` CR in `zabbix-demo` with `storageClassName: ocs-external-storagecluster-ceph-rbd`, sync-wave `2`
  - Verify: `oc get pods -n zabbix-demo` shows server, web, java-gateway, postgres pods Running
  - Verify: `oc get svc -n zabbix-demo` shows ClusterIP services on ports 10051 and 8080

## 5. Zabbix Web Route

- [ ] 5.1 Create `argocd/zabbix-web-route.yaml` — `route.openshift.io/v1 Route` with edge TLS termination and cert-manager annotations (`cert-manager.io/issuer-kind: ClusterIssuer`, `cert-manager.io/issuer-name`), sync-wave `3`
  - Verify: `oc get route -n zabbix-demo` shows the route with TLS
  - Verify: `curl -kI https://zabbix-demo.apps.cluster-d45v2.dynamic.redhatworkshops.io` returns Zabbix login page

## 6. Sample Application with Zabbix Agent

- [ ] 6.1 Create `argocd/sample-app.yaml` — `apps/v1 Deployment` with app container + `zabbix-agent2` sidecar (env: `ZBX_SERVER_HOST`, `ZBX_ACTIVESERVERS`, `ZBX_HOSTNAME=sample-app-agent`) + `v1 Service` exposing port 10050 for passive checks, sync-wave `3`
  - Verify: `oc get pods -n zabbix-demo -l app=sample-app` shows pod Running with 2/2 containers ready
  - Verify: Agent logs show successful connection to server: `oc logs -n zabbix-demo -l app=sample-app -c zabbix-agent2`

## 7. ArgoCD Application

- [ ] 7.1 Create `argocd/argocd-application.yaml` — `argoproj.io/v1alpha1 Application` in `openshift-gitops` namespace, source path `argocd/`, automated sync with `selfHeal: true` and `prune: true`, sync-wave `4`
  - Verify: `oc get application -n openshift-gitops` shows Synced and Healthy

## 8. End-to-End Validation

- [ ] 8.1 Trigger ArgoCD sync and verify all resources deploy in wave order
  - Verify: `oc get pods -n zabbix-demo` — all pods Running
  - Verify: Route accessible via HTTPS with valid TLS certificate
- [ ] 8.2 Confirm Zabbix Agent appears as connected host in Zabbix Web Dashboard with live metrics (CPU, memory, disk, network)
