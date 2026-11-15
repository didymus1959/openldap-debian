# 06 – LDAPS, sichere LDAP-Clients und TLS-Zwang

## 1. Ziel

Absicherung der LDAP-Kommunikation zwischen zwei Debian-12-VMs unter Proxmox.

## 2. Testumgebung

| System | Hostname | IP-Adresse |
|---|---|---|
| Proxmox-Host | – | 192.168.1.20 |
| LDAP-Server | server1.srv.lcd | 192.168.1.191 |
| LDAP-Client | client1.srv.lcd | 192.168.1.223 |

LDAP-Basis: `dc=srv,dc=lcd`

## 3. LDAPS

Der LDAP-Server stellt LDAPS über Port 636 bereit.

Die Verbindung wurde vom Client erfolgreich getestet:

```bash
ldapwhoami -x -H ldaps://server1.srv.lcd:636
```

Ergebnis:

```text
anonymous
```

## 4. LDAP-Client

Der Debian-12-Client verwendet `/etc/ldap/ldap.conf`:

```text
BASE dc=srv,dc=lcd
URI ldaps://server1.srv.lcd:636
TLS_CACERT /usr/local/share/ca-certificates/openldap-lab-ca.crt
TLS_REQCERT demand
```

Der Testbenutzer `testuser1` wurde erfolgreich abgefragt.

## 5. TLS-Zwang

Vor der Änderung wurde geprüft, dass keine `olcSecurity`-Richtlinie vorhanden ist.

Die aktivierte Konfiguration lautet:

```text
olcSecurity: simple_bind=128
```

Die Änderung wurde über `ldif/05-tls-enforcement.ldif` durchgeführt.

## 6. Sicherheitstests

Die Sicherheitstest wurden erfolgreich durchgeführt. Die ausführlichen Testprotokolle und Ergebnisse sind im Abschnitt "Sicherheitstests - Ergebnisse dokumentiert.

| Test | Tatsächliches Ergebnis |
|---|---|
| Passwort-Bind ohne TLS | Abgewiesen |
| Passwort-Bind über LDAPS | Erfolgreich |
| Passwort-Bind über StartTLS | Erfolgreich |

## 7. Sicherheitshinweise

- Private Schlüssel und Passwörter gehören nicht in Git.
- Die Zertifikatsprüfung bleibt aktiviert.
- Anonyme LDAP-Abfragen werden durch `simple_bind=128` nicht grundsätzlich verboten.
- Die Serverkonfiguration wurde vor der Änderung gesichert.

### Sicherheitstest: Unverschlüsselte LDAP-Anmeldung

Auf `client1.srv.lcd` wurde eine einfache LDAP-Passwortanmeldung ohne TLS mit einem absichtlich falschen Testpasswort durchgeführt.

**Ergebnis:**

```text
ldap_bind: Confidentiality required (13)
additional info: confidentiality required
```

**Bewertung:** Der LDAP-Server weist unverschlüsselte einfache Passwortanmeldungen aufgrund der aktivierten Sicherheitsrichtlinie `olcSecurity: simple_bind=128` zurück.

Der Sicherheitstest wurde erfolgreich bestanden.

## Sicherheitstests – Ergebnisse

Die Sicherheitstests wurden auf der Debian-12-Client-VM `client1.srv.lcd` (`192.168.1.223`) durchgeführt.

LDAP-Server: `server1.srv.lcd` (`192.168.1.191`)

### Test A – Unverschlüsselte LDAP-Passwortanmeldung

```bash
ldapwhoami -x \
  -H ldap://server1.srv.lcd:389 \
  -D "cn=admin,dc=srv,dc=lcd" \
  -w "ABSICHTLICH-FALSCHES-TESTPASSWORT"
```

Ergebnis:

```text
ldap_bind: Confidentiality required (13)
additional info: confidentiality required
```

**Bewertung:** Der Server weist unverschlüsselte Passwortanmeldungen korrekt zurück.

### Test B – Passwortanmeldung über LDAPS

```bash
ldapwhoami -x \
  -H ldaps://server1.srv.lcd:636 \
  -D "cn=admin,dc=srv,dc=lcd" -W
```

Ergebnis:

```text
dn:cn=admin,dc=srv,dc=lcd
```

**Bewertung:** Die verschlüsselte Anmeldung über LDAPS funktioniert.

### Test C – Passwortanmeldung über StartTLS

```bash
ldapwhoami -x -ZZ \
  -H ldap://server1.srv.lcd:389 \
  -D "cn=admin,dc=srv,dc=lcd" -W
```

Ergebnis:

```text
dn:cn=admin,dc=srv,dc=lcd
```

**Bewertung:** Die verschlüsselte Anmeldung über StartTLS funktioniert.

### Gesamtergebnis

Die Sicherheitsrichtlinie

```text
olcSecurity: simple_bind=128
```

ist aktiv und wurde erfolgreich getestet.

Unverschlüsselte einfache LDAP-Passwortanmeldungen werden abgewiesen. Verschlüsselte Anmeldungen über LDAPS und StartTLS funktionieren.

Die Richtlinie verbietet nicht grundsätzlich anonyme LDAP-Abfragen ohne TLS.
