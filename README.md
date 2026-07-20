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
│   ├── _example.yaml      # Vorlage für neue Apps (liegt bewusst NICHT in apps/, da app-of-apps sonst versucht, sie anzuwenden)
│   └── apps/              # Eine ArgoCD Application pro App (vom Ansible-Bootstrap angelegt)
│       └── pihole.yaml
└── apps/
    ├── pihole/            # Pi-hole DNS-Blocker
    │   ├── namespace.yaml
    │   ├── configmap.yaml
    │   ├── pvc.yaml
    │   ├── deployment.yaml
    │   └── service.yaml
    ├── jellyfin/          # Jellyfin Mediaserver
    │   ├── namespace.yaml
    │   ├── configmap.yaml
    │   ├── pvc.yaml
    │   ├── deployment.yaml
    │   └── service.yaml
    ├── nginx/             # Reverse Proxy für Custom-Domains (z.B. jellyfin.martinbartolome.ch)
    │   ├── namespace.yaml
    │   ├── configmap.yaml
    │   ├── deployment.yaml
    │   └── service.yaml
    ├── homeassistant/     # Home Assistant Smart-Home-Zentrale
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
| [Jellyfin](apps/jellyfin/) | `jellyfin` | Mediaserver – bindet NAS-Freigaben (`Filme`, `Serien`, `Videos`) per hostPath ein |
| [Tailscale Operator](apps/tailscale-operator/) | `tailscale` | Erlaubt es, Services per Annotation explizit im Tailnet freizugeben |
| [nginx](apps/nginx/) | `nginx` | Reverse Proxy, macht Apps über eigene Domains (z.B. `jellyfin.martinbartolome.ch`) im Tailnet erreichbar |
| [Home Assistant](apps/homeassistant/) | `homeassistant` | Smart-Home-Zentrale (Hue, Tado, Dreame-Staubsauger, Dyson-Luftreiniger) |

---

## Neue App hinzufügen

1. Verzeichnis anlegen: `apps/APPNAME/`
2. Kubernetes-Manifeste rein (Deployment, Service, …)
3. ArgoCD-Application anlegen: `argocd/apps/APPNAME.yaml` (Vorlage: `argocd/_example.yaml`)

   > ⚠️ Die Vorlage liegt bewusst außerhalb von `argocd/apps/` – dieser Ordner wird von `app-of-apps` überwacht und jede Datei darin 1:1 als `Application` angewendet. Läge `_example.yaml` dort, würde ArgoCD versuchen, sie mit dem ungültigen Namen `APPNAME` anzuwenden und der Sync würde fehlschlagen (`a lowercase RFC 1123 subdomain must consist of...`).
4. Committen und pushen – ArgoCD deployt automatisch

### Minimalbeispiel

```bash
mkdir apps/meine-app
cp apps/_template/* apps/meine-app/
cp argocd/_example.yaml argocd/apps/meine-app.yaml
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

## Zugriff auf Jellyfin

Jellyfin läuft als `NodePort`-Service und ist direkt über die Homelab-PC-IP erreichbar (kein Port-Forward nötig):

```
http://192.168.68.17:30096
```

Die NAS-Freigaben (`\\192.168.68.10\Filme`, `\Serien`, `\Videos`) werden per CIFS/SMB auf dem k3s-Node unter `/mnt/nas/Filme`, `/mnt/nas/Serien`, `/mnt/nas/Videos` gemountet (Benutzername/Passwort via `homelab-setup`, Rolle `nas-mounts`) und per `hostPath` direkt in den Jellyfin-Pod eingebunden (`/media/Filme`, `/media/Serien`, `/media/Videos`).

Die Ersteinrichtung (Sprache, Admin-User, Bibliotheken) erfolgt manuell über die Jellyfin-Weboberfläche.

---

## Zugriff auf Home Assistant

Home Assistant läuft ebenfalls als `NodePort`-Service:

```
http://192.168.68.17:30123
```

`default_config`, Reverse-Proxy/Tailscale-kompatible URLs sowie die
Custom-Integrationen für Dreame-Staubsauger und Dyson-Luftreiniger sind
bereits vorkonfiguriert bzw. vorinstalliert. Ersteinrichtung (Admin-User) und
das Verbinden von Hue, Tado, Dreame und Dyson (Account-Login bzw.
Bridge-Taste) erfolgen manuell über die Weboberfläche – Details und Gründe
dafür: [apps/homeassistant/README.md](apps/homeassistant/README.md).

---

## Explizite Tailscale-Erreichbarkeit

Bisher war eine App nur "zufällig" via Tailscale erreichbar, weil der Host im Tailnet ist und `NodePort`-Services auf allen Interfaces binden. Mit dem [Tailscale Kubernetes Operator](apps/tailscale-operator/) lässt sich das jetzt **explizit im Manifest deklarieren**:

```yaml
metadata:
  annotations:
    tailscale.com/expose: "true"
    tailscale.com/hostname: "jellyfin"
```

Details zur (einmaligen) Einrichtung des Operators: [apps/tailscale-operator/README.md](apps/tailscale-operator/README.md).

Bereits so konfiguriert: [Jellyfin](apps/jellyfin/service.yaml) (`jellyfin.<tailnet>.ts.net`), [Pi-hole](apps/pihole/service.yaml) (Web-UI unter `pihole.<tailnet>.ts.net`, Port 80; DNS unter `pihole-dns.<tailnet>.ts.net`, Port 53), [Home Assistant](apps/homeassistant/service.yaml) (`homeassistant.<tailnet>.ts.net`) und [nginx](apps/nginx/service.yaml) selbst (`nginx.<tailnet>.ts.net`) – damit ist jede App im Repo explizit im Tailnet erreichbar.

---

## Zugriff auf Jellyfin über eigene Domain (jellyfin.martinbartolome.ch)

Zusätzlich zu NodePort und Tailscale-Operator-Hostname ist Jellyfin über einen
[nginx](apps/nginx/) Reverse Proxy unter `http://jellyfin.martinbartolome.ch`
erreichbar – im gesamten Heimnetz (WLAN/LAN, z.B. für einen Smart-TV) sowie
optional auch von unterwegs übers Tailnet.

Nach dem gleichen Prinzip ist auch [Home Assistant](apps/homeassistant/) unter
`http://homeassistant.martinbartolome.ch` erreichbar (siehe
[apps/homeassistant/README.md](apps/homeassistant/README.md)) sowie [Pi-hole](apps/pihole/)
unter `http://pihole.martinbartolome.ch`. Damit sind alle Apps mit Web-UI
sowohl über die eigene Domain als auch explizit im Tailnet erreichbar.

Der DNS-Eintrag liegt als Code in [apps/pihole/custom-dns-configmap.yaml](apps/pihole/custom-dns-configmap.yaml)
(kein manueller Schritt in der Pi-hole-UI nötig). Details, Voraussetzungen
(Pi-hole muss als DNS-Server der Geräte genutzt werden) und die optionale
Tailscale-Nameserver-Einrichtung für reinen Remote-Zugriff:
[apps/nginx/README.md](apps/nginx/README.md).

---

## Nützliche kubectl-Befehle

```bash
# Alle Apps anzeigen
kubectl get applications -n argocd

# Pi-hole Status
kubectl get all -n pihole

# Logs Pi-hole
kubectl logs -n pihole -l app=pihole -f

# Jellyfin Status
kubectl get all -n jellyfin

# Logs Jellyfin
kubectl logs -n jellyfin -l app=jellyfin -f

# Home Assistant Status
kubectl get all -n homeassistant

# Logs Home Assistant
kubectl logs -n homeassistant -l app=homeassistant -f

# Manuell sync erzwingen (normalerweise nicht nötig)
kubectl annotate application pihole -n argocd argocd.argoproj.io/refresh=hard
```
