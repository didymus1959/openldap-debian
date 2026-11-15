# Übung 5 – LDAP-Benutzer und POSIX-Gruppen

## Ziel

In dieser Übung werden die Organisationseinheiten für Benutzer und Gruppen eingerichtet. Anschließend werden ein LDAP-Testbenutzer und eine POSIX-Gruppe angelegt.

## Umgebung

- LDAP-Server: `server1.srv.lcd`
- IP-Adresse: `192.168.1.191`
- LDAP-Basis-DN: `dc=srv,dc=lcd`
- LDAP-Administrator: `cn=admin,dc=srv,dc=lcd`

## 1. Organisationseinheiten anlegen

Für eine übersichtliche Verzeichnisstruktur werden zwei Organisationseinheiten verwendet:

- `ou=people,dc=srv,dc=lcd` – Benutzerkonten
- `ou=groups,dc=srv,dc=lcd` – Gruppen

Beispiel einer LDIF-Datei:

```ldif
dn: ou=people,dc=srv,dc=lcd
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=srv,dc=lcd
objectClass: organizationalUnit
ou: groups
```

Die Einträge können mit `ldapadd` angelegt werden. Dieser Befehl ist ein Beispiel; bereits vorhandene Einträge dürfen nicht erneut angelegt werden:

```bash
ldapadd -x -H ldap://localhost \
  -D "cn=admin,dc=srv,dc=lcd" -W \
  -f organisationseinheiten.ldif
```

## 2. POSIX-Gruppe anlegen

Die Gruppe `ldapusers` erhält die numerische Gruppen-ID `10000`.

```ldif
dn: cn=ldapusers,ou=groups,dc=srv,dc=lcd
objectClass: top
objectClass: posixGroup
cn: ldapusers
gidNumber: 10000
```

## 3. Testbenutzer anlegen

Der Benutzer `testuser1` wird mit folgenden Eigenschaften angelegt:

| Attribut | Wert |
|---|---|
| Benutzername | `testuser1` |
| UID | `10001` |
| Primäre GID | `10000` |
| Home-Verzeichnis | `/home/testuser1` |
| Login-Shell | `/bin/bash` |

Beispiel einer LDIF-Datei:

```ldif
dn: uid=testuser1,ou=people,dc=srv,dc=lcd
objectClass: top
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Test Benutzer 1
sn: Benutzer
uid: testuser1
uidNumber: 10001
gidNumber: 10000
homeDirectory: /home/testuser1
loginShell: /bin/bash
```

Das Benutzerpasswort wird nicht im Klartext in der LDIF-Datei oder im Git-Repository gespeichert.

## 4. LDAP-Einträge überprüfen

Auf dem LDAP-Server lassen sich die Einträge mit `ldapsearch` kontrollieren:

```bash
ldapsearch -x -LLL -H ldap://localhost \
  -b "ou=people,dc=srv,dc=lcd" \
  "(uid=testuser1)" \
  uid uidNumber gidNumber homeDirectory loginShell
```

Die Gruppe wird separat abgefragt:

```bash
ldapsearch -x -LLL -H ldap://localhost \
  -b "ou=groups,dc=srv,dc=lcd" \
  "(cn=ldapusers)" cn gidNumber
```

## 5. Ergebnis

Die LDAP-Verzeichnisstruktur enthält die Organisationseinheiten `people` und `groups`.

Der Testbenutzer `testuser1` besitzt die UID `10001` und verwendet die POSIX-Gruppe `ldapusers` mit der GID `10000` als primäre Gruppe.

Damit sind die Grundlagen für die spätere Integration von LDAP-Benutzern in Linux-Clients geschaffen.

## Hinweis

Die LDIF-Beispiele dokumentieren die Struktur der Übung. Sie sind nicht zur erneuten Ausführung auf dem bereits eingerichteten LDAP-Server gedacht.
