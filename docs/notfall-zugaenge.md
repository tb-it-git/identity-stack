# Notfall-Zugänge (Break-Glass)

## Grundregel

Jeder Dienst bekommt einen lokalen Notfall-Admin, der nicht über OIDC/LLDAP läuft.
Passwort ausdrucken → versiegelter Umschlag → Tresor.
Die Docker Befehle können 1:1 auch mit Podman genutzt werden.

---
### Nextcloud

Methode	Zugang
* Direkter Login	`https://cloud.intern/login?direct=1` (umgeht OIDC)
* CLI Passwort-Reset	`docker exec -u www-data nextcloud php occ user:resetpassword admin`
* CLI neuer Admin	`docker exec -u www-data nextcloud php occ user:setting <user> settings is_admin 1`

---

### Pocket ID
Methode	Zugang
* One-Time-Token	`docker compose exec pocket-id /app/pocket-id one-time-access-token <user>`
* Erzeugt Einmal-Link → neuen Passkey einrichten. Braucht nur Shell-Zugang, keinen Passkey.

---
Checkliste
* Lokalen Notfall-Admin pro Dienst angelegt (nicht über LDAP/OIDC)
* Passwörter/Tokens offline dokumentiert (Tresor)
* Nextcloud `?direct=1` Login getestet
* Pocket ID One-Time-Token-Befehl getestet
* Shell-Zugang zum Server unabhängig von den Diensten sichergestellt
