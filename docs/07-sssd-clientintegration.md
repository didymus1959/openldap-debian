# 07 – LDAP-Clientintegration mit SSSD

## 1. Ziel

Ein Debian-12-Client soll Benutzer- und Gruppeninformationen aus dem OpenLDAP-Verzeichnis über SSSD beziehen.

Die LDAP-Verbindung erfolgt verschlüsselt über LDAPS.

## 2. Testumgebung

| System | Hostname | IP-Adresse |
|---|---|---|
| LDAP-Server | server1.srv.lcd | 192.168.1.191 |
| LDAP-Client | client1.srv.lcd | 192.168.1.223 |

LDAP-Basis: `dc=srv,dc=lcd`

## 3. Ausgangszustand

Vor der SSSD-Installation lieferten folgende Befehle keine Ergebnisse:

```bash
getent passwd testuser1
getent group ldapusers
```

Der LDAP-Testbenutzer war bereits über `ldapsearch` erreichbar.

## 4. Installation

Auf `client1.srv.lcd` wurden folgende Pakete installiert:

```bash
sudo apt update
sudo apt install sssd sssd-ldap sssd-tools libnss-sss libpam-sss
```

## 5. NSS-Konfiguration

Die Datei `/etc/nsswitch.conf` enthält:

```text
passwd:         files systemd sss
group:          files systemd sss
shadow:         files systemd sss
```

Damit kann Linux über NSS auch SSSD zur Benutzer- und Gruppenauflösung verwenden.

## 6. SSSD-Konfiguration

Die Datei `/etc/sssd/sssd.conf` enthält:

```ini
[sssd]
config_file_version = 2
services = nss, pam
domains = srv.lcd

[nss]

[pam]

[domain/srv.lcd]
id_provider = ldap
auth_provider = ldap

ldap_uri = ldaps://server1.srv.lcd:636
ldap_search_base = dc=srv,dc=lcd
ldap_user_search_base = ou=people,dc=srv,dc=lcd
ldap_group_search_base = ou=groups,dc=srv,dc=lcd

ldap_schema = rfc2307
ldap_id_use_start_tls = false
ldap_tls_cacert = /usr/local/share/ca-certificates/openldap-lab-ca.crt
ldap_tls_reqcert = demand

cache_credentials = true
enumerate = false
```

Die Dateiberechtigungen wurden auf `root:root` und `600` gesetzt:

```bash
sudo chown root:root /etc/sssd/sssd.conf
sudo chmod 600 /etc/sssd/sssd.conf
```

## 7. Konfigurationsprüfung

```bash
sudo sssctl config-check
```

Ergebnis:

```text
Issues identified by validators: 0
Messages generated during configuration merging: 0
Used configuration snippet files: 0
```

## 8. SSSD-Dienst

```bash
sudo systemctl enable --now sssd
systemctl is-active sssd
```

Ergebnis:

```text
active
```

## 9. Funktionstests

### LDAP-Benutzer auflösen

```bash
getent passwd testuser1
```

Ergebnis:

```text
testuser1:*:10001:10000:Test Benutzer:/home/testuser1:/bin/bash
```

### LDAP-Gruppe auflösen

```bash
getent group ldapusers
```

Ergebnis:

```text
ldapusers:*:10000:testuser1
```

### Benutzeridentität prüfen

```bash
id testuser1
```

Ergebnis:

```text
uid=10001(testuser1) gid=10000(ldapusers) Gruppen=10000(ldapusers)
```

### Numerische Kennungen auflösen

```bash
getent passwd 10001
getent group 10000
```

Ergebnis:

```text
testuser1:*:10001:10000:Test Benutzer:/home/testuser1:/bin/bash
ldapusers:*:10000:testuser1
```

## 10. Ergebnis

Die LDAP-Benutzer- und Gruppenauflösung über SSSD funktioniert erfolgreich.

Der Debian-12-Client erkennt den LDAP-Testbenutzer `testuser1` mit UID `10001` und die LDAP-Gruppe `ldapusers` mit GID `10000`.

Die Kommunikation zum LDAP-Server ist über LDAPS und Zertifikatsprüfung abgesichert.

Die Anmeldung als LDAP-Benutzer über PAM oder SSH wurde noch nicht getestet und wird in einer folgenden Übung behandelt.

