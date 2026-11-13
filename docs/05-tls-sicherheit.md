### Fehlerbehebung: CA-Zertifikat nicht lesbar

**Fehler:**

```text
TLS: could not use CA certificate file
Error while reading file. (-64)
ldap_start_tls: Connect error (-11)
```

**Ursache:**

Das Verzeichnis `/etc/ldap/tls` besitzt die Berechtigungen `750` und gehört `root:openldap`. Der Benutzer `hugo` konnte deshalb das CA-Zertifikat nicht lesen.

**Lösung:**

Das öffentliche CA-Zertifikat wurde in den systemweiten Zertifikatsspeicher kopiert:

```bash
sudo install -o root -g root -m 0644 \
  /etc/ldap/tls/ca.crt \
  /usr/local/share/ca-certificates/openldap-lab-ca.crt

sudo update-ca-certificates
```

**Erfolgreicher StartTLS-Test:**

```bash
LDAPTLS_CACERT=/usr/local/share/ca-certificates/openldap-lab-ca.crt \
ldapwhoami -x -ZZ -H ldap://server1.srv.lcd
```

Ausgabe:

```text
anonymous
```

Die StartTLS-Verbindung funktioniert mit erfolgreicher Zertifikatsprüfung. Die privaten Schlüssel verbleiben im geschützten Verzeichnis `/etc/ldap/tls/`.
