# 09 – SSH-Zugriffssteuerung über LDAP-Gruppen

## 1. Ziel

Der SSH-Zugriff auf den Debian-12-Client `client1.srv.lcd` soll auf berechtigte Benutzer beschränkt werden.

LDAP-Benutzer benötigen die Mitgliedschaft in der Gruppe `sshusers`. Der lokale Administrationsbenutzer `hugo` soll weiterhin Zugriff behalten.

## 2. Testumgebung

| System | Hostname | IP-Adresse |
|---|---|---|
| LDAP-Server | server1.srv.lcd | 192.168.1.191 |
| LDAP-Client | client1.srv.lcd | 192.168.1.223 |
| Arbeitsplatz | digedag | 192.168.1.151 |

LDAP-Basis: `dc=srv,dc=lcd`

## 3. LDAP-Gruppe sshusers

Die Gruppe wurde im LDAP-Verzeichnis angelegt:

```ldif
dn: cn=sshusers,ou=groups,dc=srv,dc=lcd
objectClass: top
objectClass: posixGroup
cn: sshusers
gidNumber: 10001
memberUid: testuser1
```

Auf `client1` wurde die Gruppe über SSSD erfolgreich aufgelöst:

```text
sshusers:*:10001:testuser1
```

Die Gruppenmitgliedschaft wurde mit `id testuser1` bestätigt:

```text
uid=10001(testuser1) gid=10000(ldapusers) Gruppen=10000(ldapusers),10001(sshusers)
```

## 4. SSH-Zugriffsregel

Vor der Änderung wurde die SSH-Konfiguration gesichert:

```bash
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.vor-ldap-gruppen
```

In `/etc/ssh/sshd_config` wurde im globalen Konfigurationsbereich folgende Regel eingetragen:

```text
AllowGroups hugo sshusers
```

Damit dürfen sich Benutzer anmelden, die mindestens einer der beiden Gruppen angehören.

Die Konfiguration wurde überprüft:

```bash
sudo sshd -t
sudo sshd -T | grep '^allowgroups '
```

Ergebnis:

```text
allowgroups hugo
allowgroups sshusers
```

Anschließend wurde der SSH-Dienst neu geladen:

```bash
sudo systemctl reload ssh
```

## 5. Positive SSH-Tests

Die SSH-Anmeldung wurde für beide berechtigten Benutzer von `digedag` aus getestet:

```bash
ssh hugo@192.168.1.223
ssh testuser1@192.168.1.223
```

Beide Anmeldungen waren erfolgreich.

## 6. Zweiter LDAP-Testbenutzer

Für die Gegenprobe wurde `testuser2` angelegt.

| Attribut | Wert |
|---|---|
| UID | 10002 |
| GID | 10000 |
| Gruppe | ldapusers |
| Home-Verzeichnis | /home/testuser2 |
| Login-Shell | /bin/bash |

Der Benutzer gehört ausdrücklich nicht zur Gruppe `sshusers`.

Die LDAP-Authentifizierung wurde mit `ldapwhoami` erfolgreich getestet:

```text
dn:uid=testuser2,ou=people,dc=srv,dc=lcd
```

Auf `client1` wurde die Gruppenmitgliedschaft geprüft:

```text
uid=10002(testuser2) gid=10000(ldapusers) Gruppen=10000(ldapusers)
```

## 7. Negativer SSH-Test

Von `digedag` wurde eine SSH-Anmeldung als `testuser2` versucht.

Die Anmeldung wurde verweigert.

Im SSH-Protokoll auf `client1` wurde die Ursache nachgewiesen:

```text
User testuser2 from 192.168.1.151 not allowed because none of user's groups are listed in AllowGroups
```

Damit ist bestätigt, dass OpenSSH den Zugriff aufgrund der fehlenden Gruppenmitgliedschaft verhindert.

## 8. Ergebnis

Die SSH-Zugriffssteuerung über LDAP-Gruppen funktioniert.

| Benutzer | SSH-Zugriff |
|---|---|
| hugo | Erlaubt |
| testuser1 | Erlaubt |
| testuser2 | Verweigert |

Die Benutzer- und Gruppeninformationen werden zentral durch OpenLDAP verwaltet und auf dem Client über SSSD bereitgestellt.

OpenSSH verwendet die Gruppeninformationen, um den Zugriff anhand von `AllowGroups` zu beschränken.

**Sicherheitshinweis:** Änderungen an SSH-Zugriffsregeln sollten immer mit einer bestehenden Administrationssitzung und einer vorherigen Konfigurationssicherung durchgeführt werden.


