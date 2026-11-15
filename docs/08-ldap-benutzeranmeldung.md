# 08 – LDAP-Benutzeranmeldung über PAM und SSH

## 1. Ziel

Ein LDAP-Benutzer soll sich auf einem Debian-12-Client anmelden können.

Die Authentifizierung erfolgt über SSSD und PAM gegen einen OpenLDAP-Server. Bei der ersten Anmeldung soll automatisch ein Home-Verzeichnis angelegt werden.

## 2. Testumgebung

| System | Hostname | IP-Adresse |
|---|---|---|
| LDAP-Server | server1.srv.lcd | 192.168.1.191 |
| LDAP-Client | client1.srv.lcd | 192.168.1.223 |
| Arbeitsplatzrechner | digedag | 192.168.1.151 |

LDAP-Benutzer: `testuser1`

LDAP-DN: `uid=testuser1,ou=people,dc=srv,dc=lcd`

## 3. PAM-Konfiguration

Auf `client1.srv.lcd` wurde überprüft, dass PAM bereits SSSD verwendet.

Die Dateien `common-auth`, `common-account`, `common-session` und `common-password` enthalten entsprechende Einträge für `pam_sss.so`.

Die SSSD-Domäne `srv.lcd` war online und verwendete `server1.srv.lcd` als LDAP-Server.

## 4. Automatische Home-Verzeichnisse

Über `pam-auth-update` wurde die Option `Create home directory on login` aktiviert.

In `/etc/pam.d/common-session` ist danach folgender Eintrag vorhanden:

```text
session optional pam_mkhomedir.so
```

## 5. LDAP-Passwort setzen

Auf `server1.srv.lcd`:

```bash
ldappasswd -x \
  -H ldaps://server1.srv.lcd:636 \
  -D "cn=admin,dc=srv,dc=lcd" -W \
  -S "uid=testuser1,ou=people,dc=srv,dc=lcd"
```

Das Passwort wurde interaktiv vergeben und nicht in der Dokumentation gespeichert.

Anschließend wurde die LDAP-Authentifizierung überprüft:

```bash
ldapwhoami -x \
  -H ldaps://server1.srv.lcd:636 \
  -D "uid=testuser1,ou=people,dc=srv,dc=lcd" -W
```

Ergebnis:

```text
dn:uid=testuser1,ou=people,dc=srv,dc=lcd
```

## 6. Lokale Anmeldung über PAM

Auf `client1.srv.lcd`:

```bash
su - testuser1
```

Die Anmeldung war erfolgreich.

```text
uid=10001(testuser1) gid=10000(ldapusers) Gruppen=10000(ldapusers)
```

Das Home-Verzeichnis `/home/testuser1` wurde angelegt und gehört `testuser1:ldapusers`.

## 7. SSH-Konfiguration

Die effektive SSH-Konfiguration wurde geprüft:

```text
usepam yes
passwordauthentication yes
kbdinteractiveauthentication no
```

Es waren keine Änderungen an der SSH-Konfiguration erforderlich.

## 8. SSH-Anmeldung

Vom Arbeitsplatzrechner `digedag`:

```bash
ssh testuser1@192.168.1.223
```

Die Anmeldung mit dem LDAP-Passwort war erfolgreich.

### Funktionstests

```bash
whoami
id
hostname -f
pwd
```

Ergebnisse:

```text
testuser1
uid=10001(testuser1) gid=10000(ldapusers) Gruppen=10000(ldapusers)
client1.srv.lcd
/home/testuser1
```

## 9. Ergebnis

Die LDAP-Benutzeranmeldung auf Debian 12 funktioniert erfolgreich.

- OpenLDAP stellt die Benutzerinformationen und die Passwortauthentifizierung bereit.
- SSSD verbindet den Debian-Client über LDAPS mit OpenLDAP.
- NSS löst Benutzer- und Gruppeninformationen auf.
- PAM ermöglicht die Anmeldung und erstellt das Home-Verzeichnis.
- OpenSSH erlaubt die Anmeldung des LDAP-Benutzers.

Die lokale Anmeldung als `hugo` blieb während der Einrichtung erhalten.

**Sicherheitshinweis:** Für diese Laborumgebung wurde die SSH-Passwortauthentifizierung verwendet. Für produktive Systeme sollten zusätzlich geeignete SSH-Zugriffsbeschränkungen und weitere Schutzmaßnahmen geprüft werden.


