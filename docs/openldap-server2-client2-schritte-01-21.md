# OpenLDAP-Labor: server2 und client2 – Schritte 1 bis 21

**Stand:** 19. November 2026  
**Systeme:** Debian 12, OpenLDAP-Server `server2.srv.lcd` (`192.168.1.210`), LDAP-Client `client2.srv.lcd`  
**LDAP-Basis:** `dc=srv,dc=lcd`  
**LDAP-Administrator:** `cn=admin,dc=srv,dc=lcd`

> Diese Dokumentation fasst die im Labor tatsächlich ausgeführten Schritte zusammen. Die Schrittzählung ist für diese Gesamtdokumentation vereinheitlicht; im Gespräch wurden einzelne Schrittzahlen für Server- und Clientphase mehrfach verwendet. Erfolgreiche Tests sind als **bestätigt** gekennzeichnet. Nicht abgeschlossene Aktionen werden ausdrücklich benannt.
>
> **Sicherheit:** Niemals `ca.key`, `server2.key`, LDAP-Passwörter oder Konfigurations-Backups mit sensiblen Daten in Git/GitHub ablegen.

## Teil A – OpenLDAP-Server und TLS

### Schritt 1 – Server und LDAP-Basis prüfen

Auf `server2` wurde der Hostname `server2.srv.lcd` verwendet. Die LDAP-Datenbank wurde über `cn=config` geprüft:

```bash
ldapsearch -Q -Y EXTERNAL -H ldapi:/// \
  -b cn=config '(olcSuffix=*)' olcSuffix olcRootDN
```

**Bestätigt:** `olcSuffix: dc=srv,dc=lcd` und `olcRootDN: cn=admin,dc=srv,dc=lcd`.

### Schritt 2 – Verzeichnis für eigene Zertifizierungsstelle anlegen

```bash
install -d -m 700 /root/openldap-ca
```

Alle folgenden Schlüsselarbeiten erfolgen auf `server2` als `root`.

### Schritt 3 – Privaten CA-Schlüssel erstellen

```bash
openssl genrsa -out /root/openldap-ca/ca.key 4096
chmod 600 /root/openldap-ca/ca.key
```

Der CA-Schlüssel bleibt ausschließlich auf dem Server und wird nicht an Clients verteilt.

### Schritt 4 – Selbstsigniertes CA-Zertifikat erstellen

```bash
openssl req -x509 -new -sha256 -days 3650 \
  -key /root/openldap-ca/ca.key \
  -out /root/openldap-ca/ca.crt \
  -subj '/CN=OpenLDAP Lab CA'
chmod 644 /root/openldap-ca/ca.crt
openssl x509 -in /root/openldap-ca/ca.crt -noout -subject -issuer -dates
```

**Bestätigt:** `CN = OpenLDAP Lab CA`; gültig vom 19.11.2026 bis 16.11.2036.

### Schritt 5 – Privaten Serverschlüssel erzeugen

```bash
openssl genrsa -out /root/openldap-ca/server2.key 3072
chmod 600 /root/openldap-ca/server2.key
```

### Schritt 6 – Zertifikatsanfrage mit Hostnamen erstellen

Für den Server wurde eine CSR mit `CN=server2.srv.lcd` und `subjectAltName=DNS:server2.srv.lcd` erstellt. Beispiel:

```bash
openssl req -new \
  -key /root/openldap-ca/server2.key \
  -out /root/openldap-ca/server2.csr \
  -subj '/CN=server2.srv.lcd' \
  -addext 'subjectAltName=DNS:server2.srv.lcd'
openssl req -in /root/openldap-ca/server2.csr -noout -text
```

**Bestätigt:** Der SAN enthält `DNS:server2.srv.lcd`.

### Schritt 7 – Zertifikatserweiterungen festlegen

Datei mit `vim /root/openldap-ca/server2.ext` erstellen:

```ini
basicConstraints = critical,CA:FALSE
keyUsage = critical,digitalSignature,keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS:server2.srv.lcd
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
```

### Schritt 8 – Serverzertifikat signieren und überprüfen

```bash
cd /root/openldap-ca
openssl x509 -req \
  -in server2.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server2.crt -days 825 -sha256 -extfile server2.ext
openssl verify -CAfile ca.crt \
  -verify_hostname server2.srv.lcd -purpose sslserver server2.crt
```

**Bestätigt:** `server2.crt: OK`.

### Schritt 9 – Zertifikate für slapd installieren

```bash
install -d -o root -g openldap -m 750 /etc/ldap/tls
install -o root -g root -m 644 /root/openldap-ca/ca.crt /etc/ldap/tls/ca.crt
install -o root -g root -m 644 /root/openldap-ca/server2.crt /etc/ldap/tls/server2.crt
install -o root -g openldap -m 640 /root/openldap-ca/server2.key /etc/ldap/tls/server2.key
runuser -u openldap -- test -r /etc/ldap/tls/server2.key \
  && echo 'OK: OpenLDAP kann den privaten Schlüssel lesen'
```

**Bestätigt:** Der Dienstbenutzer `openldap` kann den privaten Schlüssel lesen.

### Schritt 10 – TLS in `cn=config` aktivieren

Datei `/root/openldap-ca/enable-tls.ldif`:

```ldif
dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/tls/ca.crt
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/tls/server2.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/tls/server2.key
```

```bash
ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f /root/openldap-ca/enable-tls.ldif
ldapsearch -Q -Y EXTERNAL -H ldapi:/// \
  -b cn=config -s base \
  olcTLSCACertificateFile olcTLSCertificateFile olcTLSCertificateKeyFile
ss -tlnp | grep ':636'
```

**Bestätigt:** Alle drei TLS-Dateipfade sind eingetragen; `slapd` lauscht auf Port 636.

### Schritt 11 – LDAPS und StartTLS am Server testen

```bash
LDAPTLS_CACERT=/etc/ldap/tls/ca.crt \
  ldapwhoami -x -H ldaps://server2.srv.lcd:636

LDAPTLS_CACERT=/etc/ldap/tls/ca.crt \
  ldapwhoami -x -ZZ -H ldap://server2.srv.lcd:389
```

**Bestätigt:** Beide Aufrufe liefern `anonymous`. Auch ein Administrator-Bind über LDAPS war erfolgreich (`dn:cn=admin,dc=srv,dc=lcd`).

### Schritt 12 – `cn=config` sichern

```bash
install -d -m 700 /root/ldap-backup
slapcat -n 0 -l /root/ldap-backup/slapd-config-vor-tls-zwang.ldif
chmod 600 /root/ldap-backup/slapd-config-vor-tls-zwang.ldif
ls -lh /root/ldap-backup/slapd-config-vor-tls-zwang.ldif
test -s /root/ldap-backup/slapd-config-vor-tls-zwang.ldif \
  && echo 'OK: Backup vorhanden und nicht leer'
```

**Bestätigt:** Datei vorhanden, 39 KB, Rechte `-rw-------`, nicht leer. **Nicht in Git speichern.**

### Schritt 13 – Unverschlüsselte Passwort-Binds verbieten

Gewünschte Einstellung:

```ldif
dn: cn=config
changetype: modify
add: olcSecurity
olcSecurity: simple_bind=128
```

Ein `ldapmodify` mit `add` meldete `Type or value exists (20)`, weil die Einstellung bereits vorhanden war. Die anschließende Prüfung bestätigte:

```bash
ldapsearch -Q -Y EXTERNAL -H ldapi:/// -b cn=config -s base olcSecurity
```

```text
olcSecurity: simple_bind=128
```

**Bestätigt:** Die Richtlinie ist aktiv. Sie verbietet einfache Passwort-Binds ohne ausreichenden Schutz, aber nicht grundsätzlich alle anonymen LDAP-Abfragen.

### Schritt 14 – TLS-Zwang praktisch testen

Unverschlüsselter Bind (nur mit beliebigem Testpasswort, **nicht** dem echten Adminpasswort):

```bash
ldapwhoami -x -H ldap://server2.srv.lcd:389 \
  -D 'cn=admin,dc=srv,dc=lcd' -W
```

**Bestätigt:** `ldap_bind: Confidentiality required (13)`.

Verschlüsselter StartTLS-Bind:

```bash
LDAPTLS_CACERT=/etc/ldap/tls/ca.crt \
ldapwhoami -x -ZZ -H ldap://server2.srv.lcd:389 \
  -D 'cn=admin,dc=srv,dc=lcd' -W
```

**Bestätigt:** `dn:cn=admin,dc=srv,dc=lcd`.

## Teil B – Debian-Client `client2.srv.lcd`

### Schritt 15 – Client und Netzwerk prüfen

```bash
hostname -f
getent hosts server2.srv.lcd
getent ahostsv4 server2.srv.lcd
ping -4 -c 3 server2.srv.lcd
cat /etc/os-release | grep PRETTY_NAME
```

**Bestätigt:** Client `client2.srv.lcd`, Debian 12 (Bookworm), IPv4-Auflösung `192.168.1.210`, Ping 3/3 erfolgreich. `getent hosts` zeigte zunächst auch eine IPv6-Link-Local-Adresse; die IPv4-Auflösung funktionierte separat.

### Schritt 16 – Öffentliche CA auf Client installieren

Auf `client2`:

```bash
scp root@server2.srv.lcd:/etc/ldap/tls/ca.crt \
  /usr/local/share/ca-certificates/openldap-lab-ca.crt
update-ca-certificates
openssl x509 \
  -in /usr/local/share/ca-certificates/openldap-lab-ca.crt \
  -noout -subject -issuer -dates
```

**Bestätigt:** Die Datei ist vorhanden (1846 Bytes), `subject` und `issuer` lauten `CN = OpenLDAP Lab CA`. `update-ca-certificates` meldete `0 added, 0 removed`, daher wurde die CA zusätzlich durch explizite LDAPS-Tests verifiziert. Nur das **öffentliche** CA-Zertifikat wird übertragen.

### Schritt 17 – LDAPS und StartTLS vom Client testen

```bash
LDAPTLS_CACERT=/usr/local/share/ca-certificates/openldap-lab-ca.crt \
  ldapwhoami -x -H ldaps://server2.srv.lcd:636

LDAPTLS_CACERT=/usr/local/share/ca-certificates/openldap-lab-ca.crt \
  ldapwhoami -x -ZZ -H ldap://server2.srv.lcd:389
```

**Bestätigt:** Beide Aufrufe lieferten `anonymous`.

### Schritt 18 – Standard-LDAP-Konfiguration festlegen

Zuerst Backup anlegen:

```bash
cp -a /etc/ldap/ldap.conf /etc/ldap/ldap.conf.bak
vim /etc/ldap/ldap.conf
```

Aktive Einstellungen:

```ini
BASE dc=srv,dc=lcd
URI ldaps://server2.srv.lcd:636
TLS_CACERT /usr/local/share/ca-certificates/openldap-lab-ca.crt
TLS_REQCERT demand
```

Prüfen:

```bash
grep -Ev '^[[:space:]]*(#|$)' /etc/ldap/ldap.conf
ldapwhoami -x
ldapsearch -x -LLL -b 'dc=srv,dc=lcd' -s base dn objectClass
```

**Bestätigt:** `ldapwhoami -x` liefert `anonymous`; der LDAP-Basiseintrag wurde gelesen (`top`, `dcObject`, `organization`). Diese Vorgabe betrifft OpenLDAP-Clientwerkzeuge, nicht zwangsläufig jedes Programm.

### Schritt 19 – LDAP-Benutzer, Gruppen und SSSD einrichten

Vorhandene LDAP-Einträge wurden abgefragt:

```bash
ldapsearch -x -LLL -b 'dc=srv,dc=lcd' \
  '(|(objectClass=posixAccount)(objectClass=posixGroup))' \
  dn objectClass uid cn uidNumber gidNumber
```

**Bestätigt:** `testuser1` (UID 10001, GID 10000) und Gruppe `ldapusers` (GID 10000).

Installierte Pakete: `sssd`, `sssd-ldap`, `libnss-sss`, `libpam-sss`; `sssd-tools` wurde für `sssctl` nachinstalliert.

Datei `/etc/sssd/sssd.conf` mit `vim` angelegt:

```ini
[sssd]
config_file_version = 2
services = nss, pam
domains = srv.lcd

[domain/srv.lcd]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldaps://server2.srv.lcd:636
ldap_search_base = dc=srv,dc=lcd
ldap_schema = rfc2307
ldap_tls_cacert = /usr/local/share/ca-certificates/openldap-lab-ca.crt
ldap_tls_reqcert = demand
cache_credentials = true
enumerate = false
```

```bash
chown root:root /etc/sssd/sssd.conf
chmod 600 /etc/sssd/sssd.conf
sssctl config-check
systemctl start sssd
systemctl is-active sssd
```

**Bestätigt:** `Issues identified by validators: 0`, SSSD `active (running)` mit NSS- und PAM-Prozessen.

NSS-/Gruppentests:

```bash
grep -E '^(passwd|group|shadow):' /etc/nsswitch.conf
getent passwd testuser1
getent group ldapusers
id testuser1
grep testuser1 /etc/passwd
```

**Bestätigt:** `sss` steht in `passwd`, `group`, `shadow`; `getent` findet den LDAP-Benutzer und die Gruppe, `id` zeigt `uid=10001` und `gid=10000`, aber `/etc/passwd` enthält keinen lokalen `testuser1`.

PAM-Integration wurde durch `pam_sss.so` in `common-auth`, `common-account`, `common-session` und `common-password` bestätigt. **Eine erfolgreiche Passwortanmeldung über PAM ist damit noch nicht nachgewiesen.**

### Schritt 20 – LDAP-Passwortproblem eingrenzen

Ein direkter Bind von `client2`:

```bash
ldapwhoami -x -D 'uid=testuser1,ou=people,dc=srv,dc=lcd' -W
```

lieferte **`Invalid credentials (49)`**. Ein `su - testuser1` aus der Root-Sitzung funktionierte, prüft als Root aber normalerweise **kein Passwort**. `sudo -i` wurde abgebrochen; ein Passwort-Reset mit lokalem `passwd testuser1` als Root wurde nicht unterstützt.

Auf `server2` wurde der DN bestätigt:

```bash
ldapsearch -Q -Y EXTERNAL -H ldapi:/// \
  -b 'dc=srv,dc=lcd' '(uid=testuser1)' dn
```

Eine Existenzsuche nach `userPassword` ergab keinen Treffer:

```bash
ldapsearch -Q -Y EXTERNAL -H ldapi:/// \
  -b 'uid=testuser1,ou=people,dc=srv,dc=lcd' \
  -s base '(userPassword=*)' dn
```

**Einordnung:** Das kann auf ein fehlendes Passwortattribut oder auf LDAP-Zugriffsregeln zurückgehen; es ist kein endgültiger Beweis für ein fehlendes Passwort. Ein Passwortwechsel für `testuser1` wurde **nicht als erfolgreich bestätigt**. Der Fokus wurde anschließend auf einen neuen LDAP-only-Benutzer gelegt.

### Schritt 21 – Neuen Benutzer ausschließlich in LDAP anlegen

Vorab auf `server2` geprüft, dass `uid=testuser2` und `uidNumber=10002` nicht vergeben sind:

```bash
ldapsearch -Q -Y EXTERNAL -H ldapi:/// \
  -b 'dc=srv,dc=lcd' \
  '(|(uid=testuser2)(uidNumber=10002))' dn uid uidNumber
```

**Bestätigt:** Kein Treffer.

Datei `/root/openldap-ca/testuser2.ldif` mit `vim` angelegt:

```ldif
dn: uid=testuser2,ou=people,dc=srv,dc=lcd
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
cn: Test Benutzer 2
sn: Benutzer
uid: testuser2
uidNumber: 10002
gidNumber: 10000
homeDirectory: /home/testuser2
loginShell: /bin/bash
```

Anschließend:

```bash
LDAPTLS_CACERT=/etc/ldap/tls/ca.crt \
ldapadd -x -H ldaps://server2.srv.lcd:636 \
  -D 'cn=admin,dc=srv,dc=lcd' -W \
  -f /root/openldap-ca/testuser2.ldif
```

**Bestätigt:** `adding new entry "uid=testuser2,ou=people,dc=srv,dc=lcd"`.

**Nächster, noch nicht bestätigter Teilschritt:** Passwort für `testuser2` sicher über LDAPS setzen:

```bash
LDAPTLS_CACERT=/etc/ldap/tls/ca.crt \
ldappasswd -x -H ldaps://server2.srv.lcd:636 \
  -D 'cn=admin,dc=srv,dc=lcd' -W -S \
  'uid=testuser2,ou=people,dc=srv,dc=lcd'
```

`-S` fragt interaktiv nach dem neuen Benutzerpasswort; `-W` nach dem LDAP-Administratorpasswort. **Eine erfolgreiche Ausführung dieses Befehls liegt bisher nicht vor.**

Danach auf `client2` testen:

```bash
ldapwhoami -x -D 'uid=testuser2,ou=people,dc=srv,dc=lcd' -W
getent passwd testuser2
id testuser2
grep '^testuser2:' /etc/passwd
```

Erwartet: LDAP-Bind erfolgreich, `getent`/`id` finden den Benutzer, **kein** lokaler Eintrag in `/etc/passwd`. Das Attribut `homeDirectory` erstellt noch kein reales Home-Verzeichnis. Eine spätere Linux-Anmeldung und gegebenenfalls Home-Verzeichnis-Erstellung werden gesondert getestet.

---

## Zusammenfassung des nachgewiesenen Stands

- **Server:** Eigene CA und Serverzertifikat installiert; LDAPS und StartTLS erfolgreich getestet.
- **Sicherheit:** `olcSecurity: simple_bind=128` aktiv; unverschlüsselte einfache Passwortanmeldung wird mit Fehler 13 abgewiesen.
- **Client:** Debian 12, vertrauenswürdige CA und standardmäßige LDAPS-Verbindung eingerichtet.
- **SSSD:** Läuft; Benutzer- und Gruppenauflösung sowie PAM-Einbindung nachgewiesen.
- **`testuser1`:** Benutzerauflösung funktioniert, direkter Passwort-Bind schlug mit Fehler 49 fehl.
- **`testuser2`:** **Nur in LDAP** erfolgreich angelegt; Passwortsetzung und Authentifizierung stehen noch aus.

**Hinweis:** Keine `useradd`-/`adduser`-Befehle für `testuser2` auf dem Client ausführen. Die Benutzeridentität bleibt im LDAP-Verzeichnis; SSSD stellt sie Linux zur Verfügung.
