# nginx – Reverse Proxy für Custom-Domains

Reverse-Proxy, damit Apps im Homelab statt über IP:Port (`192.168.68.17:30096`)
über eine eigene Domain erreichbar sind, z.B. `http://jellyfin.martinbartolome.ch`.

Aktuell konfiguriert: [`apps/nginx/configmap.yaml`](configmap.yaml) – ein
`server{}`-Block pro Domain/App. Weitere Apps: einfach eine weitere `.conf`
unter `data:` in der ConfigMap ergänzen.

Bereits eingerichtet: `jellyfin.martinbartolome.ch`, `homeassistant.martinbartolome.ch`
und `pihole.martinbartolome.ch`.

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
192.168.68.17 homeassistant.martinbartolome.ch
192.168.68.17 pihole.martinbartolome.ch
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

> Im Router als DNS-Server ebenfalls die **LAN-IP** eintragen (`192.168.68.17`),
> auf Standard-Port 53 – Router-DNS-Einstellungen erlauben normalerweise keinen
> anderen Port. Der `pihole-dns`-Service läuft deshalb als `type: LoadBalancer`
> statt `NodePort` (siehe [apps/pihole/service.yaml](../pihole/service.yaml)),
> damit k3s' ServiceLB Port 53 direkt auf der Host-IP bindet – ein NodePort
> wäre auf den Bereich 30000–32767 beschränkt.


nginx läuft als `LoadBalancer` (k3s ServiceLB) auf Port 80/443 und ist damit
automatisch über jede Host-IP erreichbar (LAN- wie Tailscale-IP) – kein
zusätzliches Setup nötig (gleiches Prinzip wie bei [Jellyfin](../jellyfin/)
und [Pi-hole](../pihole/)). Zusätzlich ist der `nginx-web`-Service explizit
über den Tailscale Kubernetes Operator im Tailnet freigegeben
(`nginx.<dein-tailnet>.ts.net`, siehe
[apps/tailscale-operator/README.md](../tailscale-operator/README.md)) – wie
bei allen anderen Apps in diesem Repo.

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

## TLS-Zertifikat einspielen/erneuern

Alle Vhosts (`jellyfin.conf`, `homeassistant.conf`, `pihole.conf`) nutzen ein echtes,
öffentlich vertrauenswürdiges **Wildcard-Zertifikat** `*.martinbartolome.ch`
(Sectigo, ausgestellt über IONOS) – dadurch gibt es **keine**
Zertifikatswarnung mehr, im Gegensatz zu einem selbstsignierten Zertifikat.
Das ist wichtig für Clients, die (anders als ein normaler Browser) das
Wegklicken einer Warnung nicht erlauben, z.B. die Home-Assistant-Companion-App
beim Login.

Zertifikat und privater Schlüssel liegen bewusst **nicht in Git** (Secrets!),
sondern in einem manuell angelegten Kubernetes-`Secret` `wildcard-martinbartolome-ch-tls`
im Namespace `nginx` (siehe `nginx-certs`-Volume in
[`deployment.yaml`](deployment.yaml)). ArgoCD verwaltet dieses Secret nicht
und überschreibt/löscht es daher auch nicht.

**Secret initial anlegen (oder nach Erneuerung aktualisieren):**

1. Bei IONOS herunterladen: *Zertifikat* (Server-Zertifikat) und
   *Intermediate-Zertifikat* (Chain).
2. Beide zu einer Fullchain-Datei zusammenfügen (Server-Zertifikat **zuerst**,
   danach das Intermediate-Zertifikat):
   ```bash
   cat zertifikat.crt intermediate.crt > fullchain.pem
   ```
3. Secret aus Fullchain + dem privaten Schlüssel (der lokal bei der
   CSR-Erstellung generiert wurde, liegt nicht bei IONOS) erzeugen:
   ```bash
   kubectl create secret tls wildcard-martinbartolome-ch-tls \
     --namespace nginx \
     --cert=fullchain.pem \
     --key=private.key \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
4. nginx neu starten, damit der neue Zertifikatsinhalt gemountet wird
   (Secret-Volumes werden zwar automatisch aktualisiert, aber erst nach
   einiger Zeit – ein Neustart wirkt sofort):
   ```bash
   kubectl rollout restart deploy/nginx -n nginx
   ```

> **Ablaufdatum im Blick behalten**: Anders als bei Let's-Encrypt-Zertifikaten
> (z.B. über `cert-manager`) gibt es hier **keine automatische Erneuerung** –
> das Zertifikat muss vor Ablauf manuell bei IONOS erneuert und die Schritte
> oben wiederholt werden.

## Ergebnis

Im Heimnetz (WLAN/LAN, inkl. TV) kann `https://jellyfin.martinbartolome.ch`
(ebenso `https://homeassistant.martinbartolome.ch` und
`https://pihole.martinbartolome.ch`) geöffnet werden → Pi-hole löst die
Domain zur LAN-IP des Homelab-Hosts auf → nginx (Port 80/443) wertet den
`Host`-Header aus, terminiert TLS mit dem Wildcard-Zertifikat und leitet an
den passenden internen Service weiter (z.B.
`jellyfin-web.jellyfin.svc.cluster.local:8096`). Zusätzlich ist jede App
(inkl. nginx selbst) explizit über den Tailscale Kubernetes Operator im
Tailnet erreichbar – so sind alle Apps sowohl über die eigene Domain als
auch im Tailnet verfügbar.


