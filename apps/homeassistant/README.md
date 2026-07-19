# Home Assistant

Smart-Home-Zentrale. Läuft als einzelne Instanz mit persistenter Config
(`/config` per PVC) und ist – wie [Jellyfin](../jellyfin/) und
[Pi-hole](../pihole/) – sowohl per NodePort, per Tailscale-Operator als auch
über die eigene Domain `homeassistant.martinbartolome.ch` erreichbar.

## Was ist schon vorkonfiguriert?

- **`default_config:`** – Standard-Integrationen (Verlauf, Logbuch,
  Automatisierungen, Frontend, Backup, ...) sind aktiv.
- **Reverse-Proxy/Tailscale-kompatibel** – `http.trusted_proxies` /
  `use_x_forwarded_for` sowie `external_url`/`internal_url` sind gesetzt
  (siehe [`configmap.yaml`](configmap.yaml)), sonst schlägt der Login über
  `homeassistant.martinbartolome.ch` mit *"400: Bad Request"* fehl.
- **Dreame-Staubsauger & Dyson-Luftreiniger** – die dafür nötigen
  Custom-Integrationen ([`Tasshack/dreame-vacuum`](https://github.com/Tasshack/dreame-vacuum)
  und [`libdyson-wg/ha-dyson`](https://github.com/libdyson-wg/ha-dyson)) werden
  per `initContainer` automatisch nach `/config/custom_components` installiert
  (siehe [`deployment.yaml`](deployment.yaml)) – keine manuelle HACS-Installation
  nötig, sie erscheinen direkt in der Integrationssuche.

## Was muss manuell eingerichtet werden – und warum

Die folgenden Schritte lassen sich bewusst **nicht** per Git/YAML abbilden,
da sie entweder eine physische Aktion oder persönliche Zugangsdaten
erfordern (die aus Sicherheitsgründen nicht im Repo landen sollten):

1. **Ersteinrichtung** (nach dem ersten Start): Sprache, Standort/Zeitzone,
   Admin-Benutzer – über den Einrichtungsassistenten unter
   `http://<homelab-ip>:30123` bzw. `http://homeassistant.martinbartolome.ch`.
2. **Philips Hue**: *Einstellungen → Geräte & Dienste → Integration
   hinzufügen → Philips Hue*. Die Bridge wird im lokalen Netz automatisch
   gefunden, danach die physische Taste auf der Bridge drücken (Pairing
   kann aus Sicherheitsgründen der Bridge nicht automatisiert werden).
3. **Tado**: *Integration hinzufügen → Tado* – Tado-Account-Zugangsdaten
   interaktiv eingeben. HA speichert sie verschlüsselt lokal; sie gehören
   nicht in Git.
4. **Dreame-Staubsauger**: *Integration hinzufügen → Dreame Vacuum* – je nach
   Modell per Cloud-Account-Login oder lokalem Token (siehe
   [Doku](https://github.com/Tasshack/dreame-vacuum#configuration)).
5. **Dyson-Luftreiniger**: *Integration hinzufügen → Dyson* – Setup per
   MyDyson-Account-Login oder WLAN-Sticker-Daten des Geräts (siehe
   [Doku](https://github.com/libdyson-wg/ha-dyson#setup)).

> Diese Schritte sind identisch mit der Ersteinrichtung von
> [Jellyfin](../jellyfin/) (Sprache, Admin-User, Bibliotheken), die laut
> Haupt-[README](../../README.md) ebenfalls manuell über die Weboberfläche
> erfolgt.

## Zugriff

- **NodePort**: `http://<homelab-ip>:30123`
- **Tailscale-Operator**: `http://homeassistant.<dein-tailnet>.ts.net`
  (siehe [apps/tailscale-operator/README.md](../tailscale-operator/README.md))
- **Eigene Domain** (Heimnetz/Tailnet): `http://homeassistant.martinbartolome.ch`
  – DNS-Eintrag in [apps/pihole/custom-dns-configmap.yaml](../pihole/custom-dns-configmap.yaml),
  Reverse-Proxy-Vhost in [apps/nginx/configmap.yaml](../nginx/configmap.yaml)
  (Details zur Funktionsweise: [apps/nginx/README.md](../nginx/README.md))

## Nützliche kubectl-Befehle

```bash
kubectl get all -n homeassistant
kubectl logs -n homeassistant -l app=homeassistant -f

# Custom-Integrationen im Pod prüfen
kubectl exec -n homeassistant deploy/homeassistant -- ls /config/custom_components
```
