# 🚀 Argo CD Personal Cheat Sheet & Workflow (GitLab Edition)

Dokumentasi praktis pengelolaan Argo CD di WSL2 dengan integrasi GitLab Repository.

---

## 1. Akses Dashboard (WSL2 -> Windows Browser)

### Forward Port (Terminal 1)
```bash
kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8080:443
```
- **URL Browser (Windows):** `https://localhost:8080`
- **Username:** `admin`

### Ambil Initial Password (Terminal 2)
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

---

## 2. CLI Authentication

### Login ke Argo CD CLI
```bash
argocd login localhost:8080 --username admin --insecure
```

### Ganti Password Admin (Opsional)
```bash
argocd account update-password
```

---

## 3. GitLab Repository Management

### Tambah Public Repository
```bash
argocd repo add https://gitlab.com/<username-or-group>/<repo-name>.git
```

### Tambah Private Repository (PAT / SSH)

#### A. Menggunakan Personal Access Token (PAT)
```bash
argocd repo add https://gitlab.com/<username-or-group>/<repo-name>.git \
  --username <gitlab-username> \
  --password <gitlab-personal-access-token>
```

#### B. Menggunakan SSH Key
```bash
argocd repo add git@gitlab.com:<username-or-group>/<repo-name>.git \
  --ssh-private-key-path ~/.ssh/id_rsa
```

---

## 4. Application Deployment

### Cara A: Create App via CLI
```bash
argocd app create my-app \
  --repo https://gitlab.com/<username-or-group>/<repo-name>.git \
  --path manifests/ \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

### Cara B: Create App via Manifest Declarative
```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://gitlab.com/<username-or-group>/<repo-name>.git'
    targetRevision: HEAD
    path: 'manifests/'
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

---

## 5. Daily Operations & CLI Cheat Sheet

```bash
# Cek semua daftar aplikasi
argocd app list

# Cek detail status aplikasi
argocd app get my-app

# Trigger manual sync
argocd app sync my-app

# Cek perbedaan (Git vs Cluster)
argocd app diff my-app

# Rollback ke commit sebelumnya
argocd app rollback my-app

# Hapus aplikasi dari Argo CD
argocd app delete my-app
```

---

## 6. Troubleshooting & Maintenance

### Reset Password Admin Interaktif (Pod Direct)
```bash
kubectl exec -n argocd -it deploy/argocd-server -- argocd admin reset-password
```

### Restart Semua Komponen Argo CD
```bash
kubectl rollout restart deployment -n argocd -l app.kubernetes.io/part-of=argocd
```

### Cek Log Pod Argo CD Server
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=100 -f
```

### Apply ApplicationSet CRD (Server-Side Apply)
```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/applicationset-crd.yaml
```

### Clean Wipe Argo CD (Total Flush)
```bash
kubectl delete namespace argocd --force --grace-period=0
kubectl delete crd applications.argoproj.io applicationsets.argoproj.io appprojects.argoproj.io --ignore-not-found
kubectl delete clusterrole -l app.kubernetes.io/part-of=argocd --ignore-not-found
kubectl delete clusterrolebinding -l app.kubernetes.io/part-of=argocd --ignore-not-found
```
