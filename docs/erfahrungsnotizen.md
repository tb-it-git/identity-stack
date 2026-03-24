# Erfahrungsnotizen: Identity Stack Aufbau
 
Dokumentierte Fallstricke aus dem Aufbau des Identity Stacks. Sortiert nach Komponente.
 
---
 
## 1. LLDAP
 
### Image-Tag: `latest-debian-rootless` verwenden
Der Tag `stable-rootless` existiert nicht auf docker.io. Das `latest`-Image (ohne rootless) versucht `/app` per `user:`-Direktive zu chown – das schlägt als non-root fehl. Lösung: `docker.io/lldap/lldap:latest-debian-rootless`.
 
### Datenverzeichnis-Rechte
`lldap-data` braucht Owner `1000:1000` – vor dem ersten Start setzen, sonst startet der Container nicht.
 
### Registry-Prompt bei Podman
Podman fragt bei unqualifizierten Image-Namen interaktiv nach der Registry. In einer Automatisierung ein möglicher Stolperstein. Entweder `unqualified-search-registries = ["docker.io"]` in `/etc/containers/registries.conf` setzen oder Image-Namen fully-qualified angeben (`docker.io/lldap/lldap:...`).
 
### Sonderzeichen in Passwörtern
Zeichen wie `!`, `#`, `$`, `%` in LLDAP-Passwörtern verursachen Probleme bei Diensten die das Passwort in Config-Dateien verwenden (SSSD, Pocket ID Bind-Passwort). Empfehlung: Nur alphanumerische Passwörter verwenden.
 
### Fehlende POSIX-Attribute (Wichtig, sollte man LLDAP auch für SSH Zugänge nutzen)
Ohne `uidNumber` und `gidNumber` als Custom Attributes werden User von Linux-Clients (SSSD) nicht gefunden. Steht in keiner Quick-Start-Anleitung. UIDs ab 10000 vergeben.
 
---
 
## 2. Pocket ID
 
### Setup-URL hat sich geändert
Version 2.x nutzt `/setup` statt `/login/setup`. Wenn die Seite "Not found" zeigt: richtige URL prüfen.
 
### Passkey-Dialog startet nicht
Der Browser startet den WebAuthn-Dialog nur, wenn er dem TLS-Zertifikat vertraut. Bei selbstsignierten Zertifikaten muss die CA auf dem Client-Gerät importiert sein – sonst passiert beim Klick auf "Passkey erstellen" schlicht nichts.
 
### Reihenfolge: Erst Passkey, dann LDAP
Beim ersten Start muss der Admin-Account lokal mit Passkey angelegt werden. Erst danach LDAP aktivieren. Wenn man LDAP vorher aktiviert und keine User synchronisiert sind, sperrt man sich aus.
 
### Setup als abgeschlossen markiert
Das Setup wird als abgeschlossen markiert sobald ein JWT-Key in der internen `kv`-Tabelle existiert. Zum Reset: `pocket-id-data/data.db` löschen.
 
### Service Account braucht richtige Gruppe
Der LDAP-Bind-User (z.B. `svc-pocketid`) muss in der Gruppe `lldap_strict_readonly` sein. Ohne Gruppenmitgliedschaft schlägt der Sync fehl.
 
### OIDC-Client Zugang einrichten
Beim Anlegen eines OIDC-Clients entweder eine LLDAP-Gruppe zuweisen oder uneingeschränkten Zugang aktivieren. Sonst kann sich niemand über den Client anmelden.
 
### UI_CONFIG_DISABLED
Muss auf `false` stehen, damit LDAP über die Web-UI konfiguriert werden kann. Die Env-Variable überschreibt die Web-UI-Einstellungen nicht – sie schaltet nur die Konfigurationsseite frei.
 
### Datenverzeichnis-Rechte
`pocket-id-data` braucht Owner `1000:1000` – wie bei LLDAP vor dem ersten Start setzen.
 
---
 
## 3. Nextcloud
 
### SSL-Trust: Drei Trust-Stores für ein Zertifikat
Bei selbstsignierten Zertifikaten muss die CA an drei Stellen vertrauenswürdig sein:
1. Im Browser des Users (CA auf Client importieren)
2. Im Nextcloud-Container per `update-ca-certificates`
3. In PHPs eigenem CA-Bundle (CA an `/etc/ssl/certs/ca-certificates.crt` anhängen)
 
Ohne Schritt 3 scheitert die OIDC-Kommunikation mit Pocket ID mit einem TLS-Fehler.
 
### "violates local access rules"
Wenn Pocket ID und Nextcloud auf dem gleichen Server laufen, blockiert Nextcloud die Kommunikation zum OIDC-Provider. Lösung:
```bash
podman exec -u www-data nextcloud php occ config:system:set allow_local_remote_servers --type=boolean --value=true
```
 
### Trusted Domains
Die Domain von Pocket ID (z.B. `id.example.com`) muss als trusted domain eingetragen werden, sonst lehnt Nextcloud die OIDC-Redirects ab:
```bash
podman exec -u www-data nextcloud php occ config:system:set trusted_domains 1 --value="id.example.com"
```
 
### overwriteprotocol geht bei Reset verloren
`overwriteprotocol: https` und `overwrite.cli.url` werden in der Nextcloud-Datenbank gespeichert, nicht in der Config-Datei. Bei `rm -rf nextcloud-data/*` sind sie weg und müssen neu gesetzt werden:
```bash
podman exec -u www-data nextcloud php occ config:system:set overwriteprotocol --value="https"
podman exec -u www-data nextcloud php occ config:system:set overwrite.cli.url --value="https://cloud.example.com"
```
 
### OIDC-App: user_oidc, nicht oidc_login
Es gibt zwei OIDC-Apps für Nextcloud. Die richtige ist `user_oidc`. Die ältere `oidc_login` hat Probleme mit dem Token-Exchange und liefert "client id or secret not provided".
 
### Datenverzeichnis-Rechte
`nextcloud-data` braucht Owner `33:33` (www-data im Container).
 
### Admin-Fallback
Wenn der OIDC-Login nicht funktioniert, kommt man über `https://cloud.example.com/login?direct=1` immer zum lokalen Admin-Login.
 
---
 
## 4. SSH Login via SSSD (optional)
 
> Dieser Abschnitt betrifft den SSH-Zugang zu Linux-Clients über LLDAP – nicht relevant für den Web-Login per Passkey.
 
### SSSD verarbeitet Sonderzeichen im Bind-Passwort nicht
In der `sssd.conf` werden `!`, `#`, `$`, `%` im `ldap_default_bind_password` nicht korrekt interpretiert. Alphanumerische Passwörter verwenden.
 
### min_id beachten
User mit `uidNumber` kleiner als `min_id` (Standard: 1000, empfohlen: 10000) werden ignoriert. UIDs in LLDAP passend vergeben.
 
### NSSwitch konfigurieren
`sss` muss bei `passwd`, `group` und `shadow` in `/etc/nsswitch.conf` eingetragen sein. Ohne das findet das System keine LDAP-User, auch wenn SSSD korrekt läuft.
 
### TLS deaktivieren für localhost
Wenn LLDAP nur auf localhost lauscht (kein TLS), muss in der SSSD-Config `ldap_auth_disable_tls_never_use_in_production = true` gesetzt werden. Der Name der Option ist Warnung genug.
 
### SSSD-Cache leeren nach Änderungen
Nach jeder Änderung an der SSSD-Config oder an LLDAP-Attributen:
```bash
systemctl stop sssd
rm -rf /var/lib/sss/db/* /var/lib/sss/mc/*
systemctl start sssd
```
Ohne das liefert SSSD veraltete Ergebnisse aus dem Cache.
 
### Fedora/RHEL: authselect statt pam-auth-update
Auf Fedora und RHEL nicht `pam-auth-update` verwenden (existiert nicht), sondern:
```bash
authselect select sssd with-mkhomedir --force
systemctl enable --now oddjobd
```
 
### FIDO2 SSH-Key-Algorithmen erlauben
Für FIDO2-basierte SSH-Keys muss in `sshd_config` stehen:
```
PubkeyAcceptedAlgorithms +sk-ssh-ed25519@openssh.com,sk-ecdsa-sha2-nistp256@openssh.com
```
 
---
 
## Was gut lief
 
- LLDAP ist in Minuten aufgesetzt und die Web-UI ist intuitiv und simpel
- Pocket ID Passkey-Login fühlt sich deutlich schneller an als Passwort
- Caddy als Reverse Proxy ist extrem simpel zu konfigurieren
- Der gesamte Stack läuft in Containern auf einem einzelnen Server
- LDAP-Sync von Pocket ID funktioniert zuverlässig (stündlich automatisch)
- One-Time-Token Onboarding ist elegant
 
## Was schwierig war
 
- Selbstsignierte Zertifikate + Browser-Trust + Container-Trust
- SSSD-Debugging ist mühsam (Logs nicht immer aussagekräftig)
- Pocket ID Setup-Prozess mit LDAP: Erst lokal, dann LDAP
- Nextcloud OIDC-Anbindung hat mehrere versteckte Voraussetzungen
