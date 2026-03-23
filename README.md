# Self-Hosted Identity Stack

Ansible-basiertes Setup für einen self-hosted Identity Stack mit zentraler Benutzerverwaltung, Passkey-Authentifizierung und FIDO2 SSH-MFA.

## Komponenten

| Komponente | Zweck | Port |
|------------|-------|------|
| **LLDAP** | Zentrale User-/Gruppenverwaltung (LDAP) | 3890 (LDAP), 17170 (Web-UI) |
| **Pocket ID** | OIDC-Provider mit Passkey-Login | 1411 |
| **Nextcloud** | Dateien, Kalender, Kontakte | 8080 |
| **Caddy** | Reverse Proxy mit TLS | 443 |
| **MailHog** | Mail-Catcher für Entwicklung/Test | 8025 |

## Architektur

```
┌──────────────────────────────────────────────┐
│                   LLDAP                       │
│         (User, Gruppen, SSH-Keys)             │
│           nur localhost:3890                  │
└─────────┬──────────────────┬─────────────────┘
          │                  │
          ▼                  ▼
┌─────────────────┐  ┌─────────────────────────┐
│   Pocket ID     │  │   Linux-Clients (SSSD)   │
│  (OIDC/Passkeys)│  │  SSH-Key-Lookup + FIDO2  │
└────────┬────────┘  └─────────────────────────┘
         │
         ▼
┌─────────────────┐
│   Nextcloud     │
│   (OIDC-Login)  │
└─────────────────┘
```

LLDAP ist nur auf `localhost` erreichbar. Zugriff auf die Web-UI erfolgt über SSH-Tunnel:

```bash
ssh -L 17170:localhost:17170 user@server
# Dann im Browser: http://localhost:17170
```

## Voraussetzungen

- Ansible 2.12+
- Zielsystem: Debian/Mint/Proxmox oder Fedora/RHEL
- Docker + Docker Compose auf dem Server
- Ein FIDO2-Stick (z.B. YubiKey 5 NFC) für SSH-MFA und Passkeys (empfohlen)

## Schnellstart

### 1. Repo klonen

```bash
git clone https://github.com/<user>/identity-stack.git
cd identity-stack
```

### 2. Inventory anpassen

```bash
cp inventory.example inventory
# Server-IP/Hostname eintragen
```

### 3. Variablen konfigurieren

```bash
cp group_vars/all.yml.example group_vars/all.yml
# LLDAP Base-DN, Bind-Passwort, Domain etc. anpassen
```

### 4. Docker-Stack deployen (LLDAP + Pocket ID + Nextcloud)

```bash
ansible-playbook -i inventory site.yml --tags docker-stack
```

### 5. LLDAP einrichten

```bash
# SSH-Tunnel zum Server
ssh -L 17170:localhost:17170 user@server

# Im Browser: http://localhost:17170
# Login: admin / <LLDAP_LDAP_USER_PASS>
# Custom Attributes anlegen: sshPublicKey, uidNumber, gidNumber
# User anlegen mit SSH-Key, uidNumber (ab 10000), gidNumber
```

### 6. Clients an LLDAP anbinden (SSSD + FIDO2)

```bash
ansible-playbook -i inventory site.yml --tags lldap-client
```

### 7. Pocket ID einrichten

```bash
# Beim ersten Start: https://id.<domain>/setup
# Lokalen Admin-Account mit Passkey anlegen
# Danach LDAP-Sync aktivieren (Container neu starten mit LDAP-Variablen)
# Für LDAP-User Passkey einrichten:
docker compose exec pocket-id /app/pocket-id one-time-access-token <username>
```

## Repo-Struktur

```
identity-stack/
├── site.yml                    # Haupt-Playbook
├── inventory.example           # Beispiel-Inventory
├── group_vars/
│   └── all.yml.example         # Beispiel-Variablen
├── roles/
│   ├── docker-stack/           # Docker Compose Stack
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   ├── docker-compose.yml.j2
│   │   │   ├── pocket-id.env.j2
│   │   │   └── Caddyfile.j2
│   │   └── defaults/
│   │       └── main.yml
│   └── lldap-client/           # SSSD + FIDO2 Client-Setup
│       ├── tasks/
│       │   └── main.yml
│       ├── templates/
│       │   └── sssd.conf.j2
│       ├── defaults/
│       │   └── main.yml
│       └── handlers/
│           └── main.yml
├── docs/
│   ├── aufbauplan.md
│   ├── notfall-zugaenge.md
│   └── konzept-userverwaltung.md
└── README.md
```

## FIDO2 SSH-MFA einrichten (Client-Seite)

Nach dem Ausrollen des Playbooks auf dem Server können User ihre FIDO2-Keys registrieren:

```bash
# YubiKey einstecken, SSH-Key generieren
ssh-keygen -t ed25519-sk -O resident -O verify-required

# Public Key in LLDAP hinterlegen (über Web-UI)
# Oder direkt auf den Server kopieren:
ssh-copy-id -i ~/.ssh/id_ed25519_sk.pub user@server
```

Beim SSH-Login wird dann automatisch der YubiKey abgefragt (Touch + PIN).

## Bekannte Fallstricke

- **SSSD Bind-Passwort:** Keine Sonderzeichen (`!`, `#`, `$`, `%`) verwenden
- **LLDAP POSIX-Attribute:** Jeder User braucht `uidNumber` und `gidNumber` (ab 10000)
- **SSSD ohne TLS:** `ldap_auth_disable_tls_never_use_in_production = true` ist gesetzt (nur localhost)
- **NSSwitch:** `sss` muss bei `passwd`, `group`, `shadow` eingetragen sein
- **Pocket ID Setup:** Erst lokalen Admin anlegen, dann LDAP aktivieren
- **FIDO2:** OpenSSH 8.2+ und `libfido2` erforderlich
- **Fedora/RHEL:** `authselect` + `oddjobd` statt `pam-auth-update`

## Notfall-Zugänge

Siehe [docs/notfall-zugaenge.md](docs/notfall-zugaenge.md) für Break-Glass-Verfahren aller Dienste.

## Lizenz

MIT
