# Aufbauplan: Self-Hosted Identity Stack

LLDAP + Pocket ID + Nextcloud mit Passkey-Login. Alles auf einem Server, alles in Containern, automatisiert per Ansible.

## Übersicht

```
┌──────────────────────────────────────────────┐
│                   LLDAP                       │
│         (User, Gruppen, SSH-Keys)             │
│           nur localhost:3890                  │
└─────────┬──────────────────┬─────────────────┘
          │ LDAP-Sync        │
          ▼                  │
┌─────────────────┐          │
│   Pocket ID     │          │
│  (OIDC/Passkeys)│          │
└────────┬────────┘          │
         │ OIDC              │
         ▼                   │
┌─────────────────┐          │
│   Nextcloud     │          │
│   (OIDC-Login)  │          │
└─────────────────┘          │

┌─────────────────┐  ┌──────┴──────────────────┐
│   MailHog       │  │   Caddy                 │
│  (Mail-Catcher) │  │  (Reverse Proxy + TLS)  │
└─────────────────┘  └─────────────────────────┘
```

## Voraussetzungen

- Ansible 2.12+ auf dem Control-Node
- Podman + podman-compose **oder** Docker + Docker Compose auf dem Zielserver
- Domain oder hosts-Einträge (z.B. `id.example.com`, `cloud.example.com`)
- Passkey-fähiges Gerät (Smartphone, YubiKey, oder Passwortmanager)

---

## Phase 1: Repo klonen und konfigurieren

```bash
git clone https://github.com/<user>/identity-stack.git
cd identity-stack

# Inventory anlegen
cp inventory.example inventory
# → Server-IP/Hostname eintragen

# Variablen konfigurieren
cp group_vars/all.yml.example group_vars/all.yml
# → Domain, Passwörter, Base-DN etc. anpassen
```

Wichtige Variablen in `group_vars/all.yml`:

```yaml
# Container Runtime
container_runtime: "podman"       # oder "docker"
compose_cmd: "podman-compose"     # oder "docker compose"

# Domain
domain: "example.com"

# TLS
tls_mode: "selfsigned"            # oder "certbot" oder "existing"

# LLDAP
lldap_base_dn: "dc=forschung,dc=intern"
lldap_admin_password: "sicheres-passwort"

# Pocket ID
pocket_id_app_url: "https://id.example.com"
pocket_id_encryption_key: "mindestens-16-zeichen"
```

> **Wichtig:** Alle Passwörter **ohne Sonderzeichen** (`!`, `#`, `$`, `%`).
> Empfehlung: Passwörter per `ansible-vault encrypt_string` verschlüsseln.

---

## Phase 2: TLS-Zertifikate erstellen (optional bei Self-Signed)

```bash
ansible-playbook -i inventory site.yml --tags tls
```

Erstellt unter `/opt/certs/self/` eine eigene CA und ein Server-Zertifikat mit den konfigurierten Domains als SAN.

> Bei `tls_mode: "certbot"` wird stattdessen ein Let's Encrypt Zertifikat angefordert.
> Bei `tls_mode: "existing"` wird nur geprüft ob die Zertifikate unter den angegebenen Pfaden existieren.

### Manueller Schritt: CA auf Client-Geräten importieren

**Linux:**
```bash
cp /opt/certs/self/ca.crt /usr/local/share/ca-certificates/identity-stack-ca.crt
update-ca-certificates
```

**Windows:**

`ca.crt` doppelklicken → Zertifikat installieren → **Lokaler Computer** → **Vertrauenswürdige Stammzertifizierungsstellen**

> Nur das CA-Zertifikat (`ca.crt`) importieren, nicht das Server-Zertifikat.
> Zwingend unter "Vertrauenswürdige Stammzertifizierungsstellen", sonst warnt der Browser weiterhin.
> Browser nach Import komplett schließen und neu öffnen.

---

## Phase 3: Docker/Podman Stack deployen

```bash
ansible-playbook -i inventory site.yml --tags docker-stack
```

Dieses Playbook:
- Erstellt alle Verzeichnisse mit korrekten Permissions (`1000:1000` für LLDAP/Pocket ID, `33:33` für Nextcloud)
- Deployt `docker-compose.yml`, `pocket-id.env` und `Caddyfile` aus Templates
- Bei Podman: Installiert `podman-compose` und konfiguriert die Default-Registry
- Startet den kompletten Stack (LLDAP, Pocket ID, Nextcloud, Caddy, MailHog)

### Prüfen ob alles läuft

```bash
podman-compose ps
podman logs lldap
podman logs pocket-id
podman logs caddy
```

---

## Phase 4: LLDAP einrichten

> Manueller Schritt – LLDAP wird über die Web-UI konfiguriert.

```bash
# SSH-Tunnel zur Web-UI (LLDAP ist nur auf localhost erreichbar)
ssh -L 17170:localhost:17170 user@server
```

Im Browser `http://localhost:17170` öffnen, Login: `admin` / `<lldap_admin_password>`

### 4.1 Service Account anlegen

| User | Gruppe | Zweck |
|------|--------|-------|
| `svc-pocketid` | `lldap_strict_readonly` | Pocket ID LDAP-Sync |

> Passwort ohne Sonderzeichen!

### 4.2 Admin-User anlegen

- Username nach Wahl
- Gruppe: `lldap_admin`
- E-Mail setzen (wird für Pocket ID benötigt)

### 4.3 Weitere User anlegen

Für jeden User E-Mail setzen. Weitere Attribute (SSH-Keys, POSIX UIDs) sind optional und werden für den Passkey-Login nicht benötigt.

---

## Phase 5: Pocket ID einrichten

> Manueller Schritt – Pocket ID wird über Browser und Web-UI konfiguriert.

### 5.1 Admin-Passkey anlegen

Im Browser: **`https://id.example.com/setup`**

Dort den ersten Admin-Account mit Passkey erstellen.

> **Nicht** `/login/setup` – die URL hat sich in v2.x geändert!
> Falls der Passkey-Dialog nicht erscheint: CA-Zertifikat korrekt importiert?
> Falls "Not found": Browser-Cache leeren oder Inkognito-Fenster.

### 5.2 LDAP-Anbindung konfigurieren

In Pocket ID einloggen → **Settings** → **Application Configuration** → LDAP:

| Feld | Wert |
|------|------|
| LDAP Enabled | ✓ |
| LDAP URL | `ldap://lldap:3890` |
| Bind DN | `uid=svc-pocketid,ou=people,dc=forschung,dc=intern` |
| Bind Password | `<passwort-ohne-sonderzeichen>` |
| Base DN | `ou=people,dc=forschung,dc=intern` |
| Admin Group | `lldap_admin` |

Sync anstoßen → prüfen ob LLDAP-User in Pocket ID erscheinen.

> Der LDAP-Sync läuft danach automatisch beim Start und jede Stunde.

### 5.3 OIDC-Client für Nextcloud anlegen

In Pocket ID → **OIDC Clients** → **Add OIDC Client**:

| Feld | Wert |
|------|------|
| Name | Nextcloud |
| Callback URL | `https://cloud.example.com/apps/user_oidc/code` |

Entweder eine Gruppe zuweisen oder **uneingeschränkten Zugang** aktivieren.

**Client ID und Client Secret notieren!**

### 5.4 LDAP-User Passkey einrichten

```bash
podman exec pocket-id /app/pocket-id one-time-access-token <username-oder-email>
```

Den ausgegebenen Link dem User geben → im Browser öffnen → Passkey einrichten.

---

## Phase 6: Nextcloud OIDC anbinden

> Teils automatisierbar, teils manuelle Konfiguration über die Web-UI.

### 6.1 OIDC-App installieren

```bash
podman exec -u www-data nextcloud php occ app:install user_oidc
```

### 6.2 SSL-Vertrauen herstellen (bei Self-Signed)

```bash
podman cp /opt/certs/self/ca.crt nextcloud:/usr/local/share/ca-certificates/identity-stack-ca.crt
podman exec nextcloud update-ca-certificates
podman exec nextcloud bash -c "cat /usr/local/share/ca-certificates/identity-stack-ca.crt >> /etc/ssl/certs/ca-certificates.crt"
podman-compose restart nextcloud
```

### 6.3 Grundkonfiguration

```bash
# HTTPS erzwingen (WICHTIG – geht bei Daten-Reset verloren!)
podman exec -u www-data nextcloud php occ config:system:set overwriteprotocol --value="https"
podman exec -u www-data nextcloud php occ config:system:set overwrite.cli.url --value="https://cloud.example.com"

# Pocket ID Domain als trusted domain
podman exec -u www-data nextcloud php occ config:system:set trusted_domains 1 --value="id.example.com"

# Lokale Server-Zugriffe erlauben (nötig wenn alles auf einem Server)
podman exec -u www-data nextcloud php occ config:system:set allow_local_remote_servers --type=boolean --value=true
```

> **Wichtig:** `overwriteprotocol` und `overwrite.cli.url` werden in der Nextcloud-Datenbank gespeichert.
> Bei `rm -rf nextcloud-data/*` gehen sie verloren und müssen neu gesetzt werden!

### 6.4 OIDC-Provider konfigurieren

> Manueller Schritt über die Nextcloud Web-UI.

Admin-Login: `https://cloud.example.com/login?direct=1`

Einstellungen → **OpenID Connect** → Provider hinzufügen:

| Feld | Wert |
|------|------|
| Identifier | PocketID |
| Client ID | `<aus Phase 5.3>` |
| Client Secret | `<aus Phase 5.3>` |
| Discovery Endpoint | `https://id.example.com/.well-known/openid-configuration` |

### 6.5 Testen

Abmelden → Login-Seite → **PocketID** → Passkey-Auth → eingeloggt in Nextcloud.

Admin-Fallback immer über: `https://cloud.example.com/login?direct=1`

### Nützliche Nextcloud Podman-Befehle

```bash
# Logs anzeigen
podman logs -f nextcloud

# Fehler-Logs
podman exec -u www-data nextcloud php occ log:tail --level=error

# Aktuelle Konfiguration prüfen
podman exec -u www-data nextcloud php occ config:system:get overwriteprotocol
podman exec -u www-data nextcloud php occ config:system:get trusted_domains

# Brute-Force-Schutz zurücksetzen
podman exec -u www-data nextcloud php occ security:bruteforce:reset <ip-adresse>

# OIDC-App Status
podman exec -u www-data nextcloud php occ app:list | grep oidc

# Container Shell
podman exec -it -u www-data nextcloud bash
```

---

## Zusammenfassung

| # | Phase                                    | Methode  | Zeitaufwand    |
|---|------------------------------------------|----------|----------------|
| 1 | Repo klonen + konfigurieren              | Ansible  | ~15 Minuten    |
| 2 | TLS-Zertifikate + CA auf Clients         | Ansible + manuell | ~30 Minuten |
| 3 | Stack deployen                           | Ansible  | ~15 Minuten    |
| 4 | LLDAP einrichten                         | Manuell (Web-UI) | ~30 Minuten |
| 5 | Pocket ID (Passkey, LDAP, OIDC-Client)   | Manuell (Web-UI + CLI) | ~45 Minuten |
| 6 | Nextcloud OIDC                           | CLI + manuell (Web-UI) | ~1 Stunde |
|   | **Gesamt**                               |          | **~3-4 Stunden**|

### Ergebnis

- **LLDAP:** Zentrale Benutzerverwaltung mit Web-UI
- **Pocket ID:** Passkey-basiertes SSO (kein Passwort nötig)
- **Nextcloud:** Login per Passkey über Pocket ID
- **MailHog:** Fängt alle E-Mails ab (Tokens, Benachrichtigungen)
- **Caddy:** TLS-Terminierung mit selbstsignierten oder Let's Encrypt Zertifikaten
- **Onboarding:** LLDAP-User anlegen → One-Time-Token → Passkey einrichten → fertig

---

## Ansible-Befehle Kurzreferenz

```bash
# Alles auf einmal
ansible-playbook -i inventory site.yml

# Einzelne Phasen
ansible-playbook -i inventory site.yml --tags tls           # Nur Zertifikate
ansible-playbook -i inventory site.yml --tags docker-stack  # Nur Stack deployen
ansible-playbook -i inventory site.yml --tags trust-ca      # CA auf Clients verteilen
ansible-playbook -i inventory site.yml --tags lldap-client  # SSSD-Clients (optional, separat)
```

---

## Fallstricke & Lessons Learned

### Podman
- Image-Namen **fully-qualified** angeben (`docker.io/library/...`) oder `unqualified-search-registries = ["docker.io"]` in `/etc/containers/registries.conf`
- LLDAP: Tag **`latest-debian-rootless`** verwenden (`stable-rootless` existiert nicht)
- Datenverzeichnisse: Owner **`1000:1000`** (LLDAP, Pocket ID), **`33:33`** (Nextcloud) – wird vom Playbook automatisch gesetzt

### Pocket ID
- Setup-URL: **`/setup`** (nicht `/login/setup` – geändert in v2.x)
- **Erst** Admin-Passkey anlegen, **dann** LDAP aktivieren
- LDAP am besten über die **Web-UI** konfigurieren (`UI_CONFIG_DISABLED=false`)
- LDAP-Sync: Beim Start + stündlich automatisch, manuell über Web-UI
- Service Account `svc-pocketid` braucht Gruppe **`lldap_strict_readonly`**
- OIDC-Clients: Gruppe zuweisen **oder** uneingeschränkten Zugang aktivieren
- Passwörter: **Keine Sonderzeichen** (`!`, `#`, `$`, `%`)

### Nextcloud
- **`overwriteprotocol: https`** muss nach jedem Daten-Reset neu gesetzt werden!
- **`allow_local_remote_servers = true`** wenn alles auf einem Server (sonst `violates local access rules`)
- Pocket ID Domain als **`trusted_domains`** eintragen
- Self-Signed CA: In Container importieren **und** in PHP CA-Bundle einhängen
- Admin-Fallback immer über: `https://cloud.example.com/login?direct=1`

### Zertifikate
- Nur **CA-Zertifikat** (`ca.crt`) auf Clients importieren, nicht das Server-Zertifikat
- Windows: Zwingend unter **Vertrauenswürdige Stammzertifizierungsstellen**
- Ablegen unter **`/opt/certs/self/`** (separater Ordner vermeidet Verwechslung)
- Browser nach CA-Import komplett schließen und neu öffnen
