### Fehlerbehebung: CA-Zertifikat nicht lesbar

**Fehlermeldung:**

```text
TLS: could not use CA certificate file
Error while reading file. (-64)
ldap_start_tls: Connect error (-11)
```

**Ursache:**

Das Verzeichnis `/etc/ldap/tls` besitzt die Berechtigungen `750` und gehört `root:openldap`.

Der normale Benutzer kann deshalb das CA-Zertifikat nicht lesen.

**Lösung:**

Das öffentliche CA-Zertifikat wird in den systemweiten Zertifikatsspeicher kopiert:

```bash
sudo install -o root -g root -m 0644 \
  /etc/ldap/tls/ca.crt \
  /usr/local/share/ca-certificates/openldap-lab-ca.crt

sudo update-ca-certificates
```

**Funktionstest:**

```bash
LDAPTLS_CACERT=/usr/local/share/ca-certificates/openldap-lab-ca.crt \
ldapwhoami -x -ZZ -H ldap://server1.srv.lcd
```

Erwartete Ausgabe: `anonymous`.

Die privaten Schlüssel bleiben weiterhin geschützt unter `/etc/ldap/tls/`.
