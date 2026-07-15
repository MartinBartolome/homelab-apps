# Vorlage für eine neue App

Dieses Verzeichnis dient als Vorlage für neue Kubernetes-Apps.

## Minimale Dateien

```
apps/APPNAME/
├── namespace.yaml    # Namespace für die App
├── deployment.yaml   # Deployment / StatefulSet
└── service.yaml      # Service (ClusterIP / NodePort)
```

## Neue App anlegen

```bash
# 1. Verzeichnis kopieren
cp -r apps/_template apps/APPNAME

# 2. Manifeste anpassen
vim apps/APPNAME/deployment.yaml

# 3. ArgoCD Application anlegen
cp argocd/_example.yaml argocd/apps/APPNAME.yaml
vim argocd/apps/APPNAME.yaml

# 4. Committen – ArgoCD deployed automatisch
git add apps/APPNAME argocd/apps/APPNAME.yaml
git commit -m "feat: APPNAME hinzufügen"
git push
```
