# Vertical Pod Autoscaler (VPA) — end-to-end for aksvpatest

## Goal
Right-size CPU/memory for `swagger-api`, `postgres`, and `nginx-demo` using AKS VPA, while keeping **GitOps** (Argo CD) as source of truth.

## Recommended mode for this lab
| Mode | Behaviour | Use here? |
|------|-----------|-----------|
| **Off** | Recommendations only; no pod eviction | **Yes (default in Git)** |
| Initial | Sets requests only on new pods | Optional later |
| Recreate / Auto | Evicts pods to apply new requests | **No** with Argo `selfHeal` until process is clear |

Flow: **VPA Off → read recommendations → update Git YAML → Argo sync**.

---

## Step 1 — Enable AKS VPA add-on (once)

```powershell
az aks get-credentials -g rg-aks-gitops -n aks-cluster-01 --overwrite-existing

az aks update `
  --resource-group rg-aks-gitops `
  --name aks-cluster-01 `
  --enable-vpa
```

Verify:

```powershell
az aks show -g rg-aks-gitops -n aks-cluster-01 --query verticalPodAutoscalerProfile -o json

kubectl get pods -n kube-system | findstr vpa
# expect: vpa-recommender, vpa-updater, vpa-admission-controller

kubectl get crd | findstr verticalpodautoscaler
```

Disable later (if needed):

```powershell
az aks update -g rg-aks-gitops -n aks-cluster-01 --disable-vpa
```

---

## Step 2 — GitOps VPA objects (already added in repo)

| File | Target |
|------|--------|
| `k8s/swagger-app/vpa-swagger-api.yaml` | Deployment/swagger-api |
| `k8s/swagger-app/vpa-postgres.yaml` | Deployment/postgres |
| `k8s/app-deployment/vpa-nginx-demo.yaml` | Deployment/nginx-demo |

Synced by existing Argo apps `swagger-app` and `demo-app-deployment` after push.

Commit & push:

```powershell
cd C:\Users\venka\aksvpatest
git add k8s/swagger-app/vpa-*.yaml k8s/app-deployment/vpa-nginx-demo.yaml k8s/vpa
git commit -m "Add VPA recommendation policies for swagger, postgres, nginx"
git push origin add-aks-workflow
```

Force Argo refresh:

```powershell
kubectl annotate application swagger-app -n argocd argocd.argoproj.io/refresh=hard --overwrite
kubectl annotate application demo-app-deployment -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

---

## Step 3 — Confirm VPA objects

```powershell
kubectl get vpa -A
kubectl describe vpa swagger-api-vpa -n swagger-app
kubectl describe vpa postgres-vpa -n swagger-app
kubectl describe vpa nginx-demo-vpa -n demo-app-namespace
```

Look for **Status → Recommendation → Container Recommendations** (Target / Lower / Upper / Uncapped).

If empty for a few minutes: normal. Generate traffic (load test), wait 5–15+ minutes.

---

## Step 4 — Generate usage for better recommendations

```powershell
# API forward
kubectl port-forward -n swagger-app svc/swagger-api 9000:9000

# Load (other terminal)
cd C:\Users\venka\aksvpatest
python scripts\swagger_load_test.py --mode both --duration 180 --workers 20
```

Then re-check:

```powershell
kubectl get vpa -n swagger-app -o yaml
```

---

## Step 5 — Apply recommendations via Git (not kubectl edit)

Example: if VPA target says cpu `25m`, memory `96Mi` for `api`:

1. Edit `k8s/swagger-app/api.yaml` requests/limits  
2. Push to `add-aks-workflow`  
3. Argo syncs — cluster matches Git  

Do **not** `kubectl set resources` if you want Argo selfHeal to stay clean.

---

## Step 6 — Optional: move from Off → Initial (advanced)

Only after you trust recommendations:

```yaml
updatePolicy:
  updateMode: "Initial"
```

New pods get VPA requests at creation; existing pods unchanged until restart. Still prefer writing final values back to Git.

Avoid `Recreate` on free-tier single-node + Postgres single replica unless you accept downtime.

---

## VPA vs KRR vs kubectl top

| Tool | Role |
|------|------|
| kubectl top | Instant util snapshot |
| KRR | Prometheus history → CLI recommendations |
| VPA (Off) | In-cluster continuous recommendations on VPA CR |

Use all three; **Git remains source of truth**.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| CRD not found | Step 1 `--enable-vpa` not finished |
| Argo OutOfSync / Unknown | VPA applied before add-on ready; re-sync after CRDs exist |
| Empty recommendations | Need more metrics time + traffic |
| Pods thrashing | You set Recreate; switch back to Off |
| OOM after shrink | Raise `minAllowed` / keep headroom in Git limits |

---

## Cleanup

```powershell
# Remove VPA objects from Git (delete files + push) OR:
kubectl delete vpa swagger-api-vpa postgres-vpa -n swagger-app
kubectl delete vpa nginx-demo-vpa -n demo-app-namespace

az aks update -g rg-aks-gitops -n aks-cluster-01 --disable-vpa
```
