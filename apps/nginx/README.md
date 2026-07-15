# nginx – Reverse Proxy für Custom-Domains

Reverse-Proxy, damit Apps im Homelab statt über IP:Port (`192.168.68.17:30096`)
über eine eigene Domain erreichbar sind, z.B. `http://jellyfin.martinbartolome.ch`.

Aktuell konfiguriert: [`apps/nginx/configmap.yaml`](configmap.yaml) – ein
`server{}`-Block pro Domain/App. Weitere Apps: einfach eine weitere `.conf`
unter `data:` in der ConfigMap ergänzen.

## Warum nginx statt Tailscale-Operator-Hostname?

Der Tailscale Operator (`apps/tailscale-operator/`) macht Jellyfin bereits unter
`jellyfin.<dein-tailnet>.ts.net` erreichbar. Für eine **eigene** Domain
(`jellyfin.martinbartolome.ch`) braucht es zusätzlich:

1. Einen Reverse-Proxy, der den `Host`-Header auswertet und passend weiterleitet (nginx, dieser App)
2. Eine DNS-Auflösung von `jellyfin.martinbartolome.ch` im Heimnetz/Tailnet

## DNS-Auflösung über Pi-hole (per Code verwaltet)

Der Local-DNS-Eintrag liegt als ConfigMap im Repo:
[`apps/pihole/custom-dns-configmap.yaml`](../pihole/custom-dns-configmap.yaml)
(gemountet nach `/etc/pihole/custom.list`) – kein manueller Schritt in der
Pi-hole-Web-UI mehr nötig.

```
192.168.68.17 jellyfin.martinbartolome.ch
```

Es wird bewusst die **LAN-IP** des Homelab-Hosts eingetragen (nicht die
Tailscale-IP), damit auch Geräte funktionieren, die nicht im Tailnet sind
(z.B. ein Smart-TV im WLAN, das kein Tailscale installieren kann). Damit das
greift, muss Pi-hole als DNS-Server für das jeweilige Gerät genutzt werden:

- **Ganzes Heimnetz**: Router so konfigurieren, dass er Pi-hole per DHCP als
  DNS-Server verteilt (falls noch nicht geschehen) – dann funktioniert es für
  alle Geräte im WLAN/LAN automatisch, auch für den TV.
- **Einzelnes Gerät** (z.B. nur der TV): DNS-Server manuell in dessen
  Netzwerkeinstellungen auf die LAN-IP von Pi-hole setzen.

nginx läuft als `NodePort` (Port `30800`) und ist damit automatisch über jede
Host-IP erreichbar (LAN- wie Tailscale-IP) – kein zusätzliches Setup nötig
(gleiches Prinzip wie bei [Jellyfin](../jellyfin/) und [Pi-hole](../pihole/)).

> Nach einer Änderung an `custom-dns-configmap.yaml` greift der neue Inhalt
> erst nach einem Reload/Neustart des Pi-hole-Pods (siehe Kommentar in der
> Datei) – ArgoCD aktualisiert nur die Datei im Pod, lädt sie aber nicht
> automatisch neu.

## Optional: Zugriff auch von unterwegs (außerhalb des Heimnetzes)

Damit die Domain auch für Geräte auflösbar ist, die **nur** über Tailscale
verbunden sind (z.B. unterwegs, nicht im Heim-WLAN), zusätzlich – einmalig,
nicht per Git abbildbar – in der
[Tailscale Admin Console → DNS](https://login.tailscale.com/admin/dns):

- **Nameserver hinzufügen**: Tailscale-IP von Pi-hole (`tailscale ip -4` auf
  dem Homelab-Host, oder [Tailscale Admin Console](https://login.tailscale.com/admin/machines))
- Als **globaler Nameserver** (Pi-hole übernimmt DNS + Werbeblocker fürs
  ganze Tailnet) oder als **Split DNS** nur für `martinbartolome.ch`

> Wichtig: Die LAN-IP (`192.168.68.17`) ist von unterwegs über Tailscale
> **nicht** erreichbar, da kein Tailscale-Subnet-Router für das Heimnetz
> eingerichtet ist. Für reinen Remote-Zugriff bräuchte es entweder einen
> Subnet-Router (`--advertise-routes=192.168.68.0/24` in
> [homelab-setup/roles/tailscale](../../../homelab-setup/roles/tailscale/))
> oder weiterhin den bestehenden `*.ts.net`-Hostnamen des Tailscale-Operators.

## Ergebnis

Im Heimnetz (WLAN/LAN, inkl. TV) kann `http://jellyfin.martinbartolome.ch`
öffnen → Pi-hole löst die Domain zur LAN-IP des Homelab-Hosts auf → nginx
(Port 30800) wertet den `Host`-Header aus und leitet an
`jellyfin-web.jellyfin.svc.cluster.local:8096` weiter.

> HTTPS/TLS ist hier bewusst nicht konfiguriert (reiner Zugriff über HTTP im
> Heimnetz/Tailnet). Falls gewünscht, könnte z.B. `cert-manager` mit einer
> DNS-01-Challenge oder ein selbstsigniertes Zertifikat ergänzt werden.

