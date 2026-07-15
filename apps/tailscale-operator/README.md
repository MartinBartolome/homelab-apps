# Tailscale Kubernetes Operator

Installiert den [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator) per Helm-Chart (siehe `argocd/apps/tailscale-operator.yaml`, keine Manifeste in diesem Ordner nötig).

Damit lässt sich pro `Service` **explizit deklarieren**, dass eine App im Tailnet erreichbar sein soll – statt sich implizit darauf zu verlassen, dass der Host per NodePort auch über sein Tailscale-Interface erreichbar ist.

## Einmalige manuelle Einrichtung

Diese Schritte lassen sich nicht per Git/ArgoCD abbilden (Tailscale-Account-Konfiguration + Secret):

1. **ACLs** im [Tailscale Admin Console](https://login.tailscale.com/admin/acls) ergänzen:

   ```json
   {
     "tagOwners": {
       "tag:k8s-operator": ["autogroup:admin"],
       "tag:k8s":          ["tag:k8s-operator"]
     }
   }
   ```

2. **OAuth-Client** anlegen: [Trust credentials](https://login.tailscale.com/admin/settings/trust-credentials) → *+ Credential*
   - Scopes: `General > Services` (Read/Write), `Devices > Core` (Read/Write), `Keys > Auth Keys` (Read/Write)
   - Tag: `tag:k8s-operator`

   > ⚠️ Ohne `Keys > Auth Keys` scheitert der Operator-Start mit
   > `creating operator authkey: ... does not have enough permissions (403)`.
   > Scopes lassen sich bei einem bestehenden OAuth-Client nicht nachträglich
   > ändern – in dem Fall einen neuen Client mit allen drei Scopes anlegen und
   > das Secret (Schritt 3) sowie den Operator-Pod neu erstellen.

3. **Secret** im Cluster anlegen (Namespace muss existieren, wird beim ersten ArgoCD-Sync per `CreateNamespace=true` erzeugt):

   ```bash
   kubectl create namespace tailscale --dry-run=client -o yaml | kubectl apply -f -
   kubectl create secret generic operator-oauth \
     -n tailscale \
     --from-literal=client_id="<CLIENT_ID>" \
     --from-literal=client_secret="<CLIENT_SECRET>"
   ```

4. Committen/pushen (die `tailscale-operator.yaml` Application), ArgoCD deployt den Operator automatisch.

## Eine App via Tailscale erreichbar machen

Auf dem `Service` der App folgende Annotations setzen (Layer-3-Ingress, funktioniert mit dem bestehenden `Service`-Typ, z.B. `NodePort`):

```yaml
metadata:
  annotations:
    tailscale.com/expose: "true"
    tailscale.com/hostname: "jellyfin"   # ergibt jellyfin.<dein-tailnet>.ts.net
```

Der Operator legt dafür einen zusätzlichen Proxy-Pod an, der als eigenes Gerät im Tailnet erscheint und den Traffic zur `ClusterIP` des Service weiterleitet. Den MagicDNS-Namen findet man im Tailscale Admin Console unter *Machines*, oder:

```bash
kubectl get svc -n <namespace> <service-name>
```

Beispiele bereits konfiguriert:
- Jellyfin: [apps/jellyfin/service.yaml](../jellyfin/service.yaml)
- Pi-hole (Web-UI + DNS): [apps/pihole/service.yaml](../pihole/service.yaml)
