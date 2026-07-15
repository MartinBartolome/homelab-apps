# nginx – Reverse Proxy für Custom-Domains im Tailnet

Reverse-Proxy, damit Apps im Homelab statt über IP:Port (`192.168.68.17:30096`)
über eine eigene Domain erreichbar sind, z.B. `https://jellyfin.martinbartolome.ch`.

Aktuell konfiguriert: [`apps/nginx/configmap.yaml`](configmap.yaml) – ein
`server{}`-Block pro Domain/App. Weitere Apps: einfach eine weitere `.conf`
unter `data:` in der ConfigMap ergänzen.

## Warum nginx statt Tailscale-Operator-Hostname?

Der Tailscale Operator (`apps/tailscale-operator/`) macht Jellyfin bereits unter
`jellyfin.<dein-tailnet>.ts.net` erreichbar. Für eine **eigene** Domain
(`jellyfin.martinbartolome.ch`) braucht es zusätzlich:

1. Einen Reverse-Proxy, der den `Host`-Header auswertet und passend weiterleitet (nginx, dieser App)
2. Eine DNS-Auflösung von `jellyfin.martinbartolome.ch`, die **nur im Tailnet** funktioniert

## Einmalige manuelle Einrichtung (nicht per Git/ArgoCD abbildbar)

### 1. DNS-Auflösung über Pi-hole

Da Pi-hole bereits als lokaler DNS-Resolver läuft, wird die Domain dort als
**Local DNS Record** eingetragen (Pi-hole Web-UI → *Local DNS* → *DNS Records*):

```
Domain:  jellyfin.martinbartolome.ch
IP:      <Tailscale-IP des Homelab-Hosts, z.B. 100.x.x.x>
```

> Die IP findest du mit `tailscale ip -4` auf dem Homelab-PC oder in der
> [Tailscale Admin Console](https://login.tailscale.com/admin/machines).

### 2. Pi-hole als Nameserver im Tailnet hinterlegen

Damit Geräte im Tailnet die Pi-hole-Auflösung auch nutzen, in der
[Tailscale Admin Console → DNS](https://login.tailscale.com/admin/dns):

- **Nameserver hinzufügen**: Tailscale-IP von Pi-hole (`pihole-web`-Host, NodePort erreichbar)
- Am einfachsten als **globaler Nameserver** (dann übernimmt Pi-hole DNS + Werbeblocker fürs ganze Tailnet), alternativ als **Split DNS** nur für `martinbartolome.ch`, falls die restliche DNS-Auflösung unverändert bleiben soll

### 3. nginx im Tailnet erreichbar machen

nginx läuft als `NodePort` (Port `30800`) und ist damit automatisch über jede
Host-IP erreichbar, auch über die Tailscale-IP – kein zusätzliches Setup nötig
(gleiches Prinzip wie bei [Jellyfin](../jellyfin/) und [Pi-hole](../pihole/)).

## Ergebnis

Jedes Gerät im Tailnet kann anschließend `http://jellyfin.martinbartolome.ch`
öffnen → Pi-hole löst die Domain zur Tailscale-IP des Homelab-Hosts auf → nginx
(Port 30800) wertet den `Host`-Header aus und leitet an
`jellyfin-web.jellyfin.svc.cluster.local:8096` weiter.

> HTTPS/TLS ist hier bewusst nicht konfiguriert (reiner Tailnet-interner
> Zugriff über HTTP). Falls gewünscht, könnte z.B. `cert-manager` mit einer
> DNS-01-Challenge oder ein selbstsigniertes Zertifikat ergänzt werden.
