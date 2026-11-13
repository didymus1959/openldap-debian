# 03 – LDAP-Basisstruktur konfigurieren

## Ziel

Einrichtung einer LDAP-Verzeichnisstruktur auf dem Server `server1.srv.lcd` unter Debian 12 oder 13.

## 1. Serverinformationen

| Parameter | Wert |
|---|---|
| Hostname | server1 |
| FQDN | server1.srv.lcd |
| IP-Adresse | 192.168.1.191 |
| DNS-Domain | srv.lcd |
| Basis-DN | dc=srv,dc=lcd |
| Administrator-DN | cn=admin,dc=srv,dc=lcd |

## 2. LDAP-Verzeichnisstruktur

Die LDAP-Struktur besteht aus:

```text
dc=srv,dc=lcd
├── ou=people
└── ou=groups
```

Die Organisationseinheit `people` dient zur Verwaltung von Benutzerkonten.

Die Organisationseinheit `groups` dient zur Verwaltung von Gruppen.

## 3. LDIF-Datei

Die Definition der Organisationseinheiten befindet sich in:

`ldif/01-organizational-units.ldif`

## 4. LDIF-Datei auf den Server übertragen

Vom lokalen Linux-Rechner:

```bash
scp ldif/01-organizational-units.ldif \
  BENUTZER@192.168.1.191:/tmp/
```

`BENUTZER` ist durch den tatsächlichen SSH-Benutzernamen zu ersetzen.

## 5. Organisationseinheiten anlegen

Auf `server1.srv.lcd`:

```bash
ldapadd -x -H ldap://localhost \
  -D "cn=admin,dc=srv,dc=lcd" \
  -W \
  -f /tmp/01-organizational-units.ldif
```

Das Administratorpasswort wird interaktiv abgefragt.

## 6. LDAP-Struktur überprüfen

Auf dem LDAP-Server:

```bash
ldapsearch -x -H ldap://localhost \
  -b "dc=srv,dc=lcd" \
  "(objectClass=organizationalUnit)" dn ou
```

Erwartete Einträge:

```text
dn: ou=people,dc=srv,dc=lcd
ou: people

dn: ou=groups,dc=srv,dc=lcd
ou: groups
```

## 7. Netzwerk und DNS überprüfen

Namensauflösung testen:

```bash
getent hosts server1.srv.lcd
```

Erwartete IPv4-Adresse:

`192.168.1.191`

## 8. Sicherheit

Die bisherigen LDAP-Abfragen über `ldap://localhost` erfolgen lokal auf dem Server.

Für spätere LDAP-Verbindungen über das Netzwerk wird TLS eingerichtet.

Administratorpasswörter und private Schlüssel werden nicht im Git-Repository gespeichert.

## 9. Ergebnis

Die LDAP-Basisstruktur auf `server1.srv.lcd` wurde eingerichtet und überprüft.

Im nächsten Kapitel werden LDAP-Benutzer und Gruppen angelegt.
