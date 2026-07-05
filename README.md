# homelab-apps

Kubernetes-Manifeste für mein Homelab. Apps werden via **ArgoCD** (GitOps) automatisch deployed – jeder Push auf `main` wird innerhalb von Minuten auf dem Cluster angewendet.

---

## Funktionsweise

```
Git Push → ArgoCD erkennt Änderung → kubectl apply → App läuft
```

ArgoCD überwacht dieses Repo mit dem **App-of-Apps**-Pattern:
- `argocd/apps/` enthält eine ArgoCD-`Application` pro App
- Jede `Application` zeigt auf das Verzeichnis der zugehörigen App (z.B. `apps/pihole/`)
- ArgoCD synchronisiert automatisch

---

## Struktur

```
homelab-apps/
├── argocd/
│   └── apps/              # Eine ArgoCD Application pro App (vom Ansible-Bootstrap angelegt)
│       ├── pihole.yaml
│       └── _example.yaml  # Vorlage für neue Apps
└── apps/
    ├── pihole/            # Pi-hole DNS-Blocker
    │   ├── namespace.yaml
    │   ├── configmap.yaml
    │   ├── pvc.yaml
    │   ├── deployment.yaml
    │   └── service.yaml
    └── _template/         # Vorlage für neue Apps
        └── README.md
```

---

## Apps

| App | Namespace | Beschreibung |
|---|---|---|
| [Pi-hole](apps/pihole/) | `pihole` | DNS-Blocker / lokaler DNS-Resolver |

---

## Neue App hinzufügen

1. Verzeichnis anlegen: `apps/APPNAME/`
2. Kubernetes-Manifeste rein (Deployment, Service, …)
3. ArgoCD-Application anlegen: `argocd/apps/APPNAME.yaml` (Vorlage: `argocd/apps/_example.yaml`)
4. Committen und pushen – ArgoCD deployt automatisch

### Minimalbeispiel

```bash
mkdir apps/meine-app
cp apps/_template/* apps/meine-app/
cp argocd/apps/_example.yaml argocd/apps/meine-app.yaml
# Manifeste und ArgoCD-App anpassen
git add . && git commit -m "feat: meine-app hinzufügen"
git push
```

---

## Zugriff auf ArgoCD UI

```bash
# Port-Forward (vom Homelab-PC oder via Tailscale)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Browser: https://localhost:8080
# User: admin
# Passwort: wurde beim Ansible-Run ausgegeben
```

---

## Nützliche kubectl-Befehle

```bash
# Alle Apps anzeigen
kubectl get applications -n argocd

# Pi-hole Status
kubectl get all -n pihole

# Logs Pi-hole
kubectl logs -n pihole -l app=pihole -f

# Manuell sync erzwingen (normalerweise nicht nötig)
kubectl annotate application pihole -n argocd argocd.argoproj.io/refresh=hard
```
