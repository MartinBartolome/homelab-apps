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
- **HACS (Home Assistant Community Store)** – wird per `initContainer`
  automatisch als `custom_component` nach `/config/custom_components/hacs`
  installiert (siehe [`deployment.yaml`](deployment.yaml)). Die einmalige
  Aktivierung (GitHub-Account verknüpfen) ist dennoch ein interaktiver
  Schritt, siehe [unten](#was-muss-manuell-eingerichtet-werden--und-warum).
- **Dreame-Staubsauger & Dyson-Luftreiniger** – die dafür nötigen
  Custom-Integrationen ([`Tasshack/dreame-vacuum`](https://github.com/Tasshack/dreame-vacuum)
  und [`libdyson-wg/ha-dyson`](https://github.com/libdyson-wg/ha-dyson)) werden
  ebenfalls per `initContainer` automatisch nach `/config/custom_components`
  installiert (siehe [`deployment.yaml`](deployment.yaml)) – unabhängig von
  HACS, sie erscheinen direkt in der Integrationssuche.

## Was muss manuell eingerichtet werden – und warum

Die folgenden Schritte lassen sich bewusst **nicht** per Git/YAML abbilden,
da sie entweder eine physische Aktion oder persönliche Zugangsdaten
erfordern (die aus Sicherheitsgründen nicht im Repo landen sollten):

1. **Ersteinrichtung** (nach dem ersten Start): Sprache, Standort/Zeitzone,
   Admin-Benutzer – über den Einrichtungsassistenten unter
   `http://<homelab-ip>:30123` bzw. `http://homeassistant.martinbartolome.ch`.
2. **HACS aktivieren**: HACS ist zwar bereits als Integration installiert
   (siehe oben), die Erstaktivierung erfordert aber einen interaktiven
   GitHub-Login (Device-Code-Verfahren), der aus Sicherheitsgründen nicht
   automatisiert werden kann. Nach einem Neustart von HA: *Einstellungen →
   Geräte & Dienste → Integration hinzufügen → HACS* und den Anweisungen
   folgen (GitHub-Konto verknüpfen, ggf. HA neu starten).
   > Hinweis: War `/config/custom_components/hacs` beim allerersten
   > Deployment (vor dem ersten HA-Start) noch nicht vorhanden, installiert
   > der `initContainer` HACS erst beim nächsten Pod-Neustart, da HACS
   > selbst eine bereits gestartete HA-Instanz voraussetzt.
3. **Philips Hue**: *Einstellungen → Geräte & Dienste → Integration
   hinzufügen → Philips Hue*. Die Bridge wird im lokalen Netz automatisch
   gefunden, danach die physische Taste auf der Bridge drücken (Pairing
   kann aus Sicherheitsgründen der Bridge nicht automatisiert werden).
4. **Tado**: *Integration hinzufügen → Tado* – Tado-Account-Zugangsdaten
   interaktiv eingeben. HA speichert sie verschlüsselt lokal; sie gehören
   nicht in Git.
5. **Dreame-Staubsauger**: *Integration hinzufügen → Dreame Vacuum* – je nach
   Modell per Cloud-Account-Login oder lokalem Token (siehe
   [Doku](https://github.com/Tasshack/dreame-vacuum#configuration)).
6. **Dyson-Luftreiniger**: *Integration hinzufügen → Dyson* – Setup per
   MyDyson-Account-Login oder WLAN-Sticker-Daten des Geräts (siehe
   [Doku](https://github.com/libdyson-wg/ha-dyson#setup)).
7. **Matter / TadoX-Bridge**: *Integration hinzufügen → Matter (BETA)* –
   als Server-URL `ws://localhost:5580/ws` eingeben (der Matter Server läuft
   als Sidecar-Container im selben Pod, siehe [`deployment.yaml`](deployment.yaml)),
   anschließend den Pairing-/Setup-Code der TadoX-Bridge eingeben (App oder
   Aufdruck auf der Bridge). Kommissionierung ist ein interaktiver Vorgang
   und lässt sich nicht per YAML vorkonfigurieren.
   > Hinweis: Damit die mDNS/IPv6-Multicast-basierte Kommissionierung im
   > lokalen Netz funktioniert, läuft der komplette Pod mit `hostNetwork: true`
   > (siehe [`deployment.yaml`](deployment.yaml)) – das Pod-Overlay-Netzwerk
   > von k3s leitet Multicast sonst nicht zuverlässig weiter.

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

# Matter-Server-Logs separat prüfen (eigener Container im selben Pod)
kubectl logs -n homeassistant -l app=homeassistant -c matter-server -f
```
