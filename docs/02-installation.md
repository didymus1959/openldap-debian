# 02 – Installation von OpenLDAP

## Ziel

Installation und Überprüfung eines OpenLDAP-Servers unter Debian 12 oder Debian 13.

## 1. Systeminformationen

Betriebssystem: Debian GNU/Linux 12 oder 13

Version überprüfen:

```bash
cat /etc/os-release
```

## 2. Installation

Paketlisten aktualisieren:

```bash
sudo apt update
```

OpenLDAP installieren:

```bash
sudo apt install slapd ldap-utils
```

## 3. Grundkonfiguration

Konfigurationsdialog starten:

```bash
sudo dpkg-reconfigure -plow slapd
```

Beispielkonfiguration:

| Parameter | Wert |
|---|---|
| DNS-Domain | srv.lcd |
| Organisation | LDAP Testumgebung |
| Basis-DN | dc=srv,dc=lcd |
| Administrator-DN | cn=admin,dc=srv,dc=lcd |

Das Administratorpasswort wird nicht dokumentiert.

**Hinweis:** Eine erneute Konfiguration kann vorhandene LDAP-Daten verändern oder ersetzen. Vorher eine Sicherung erstellen.

## 4. Dienststatus

```bash
sudo systemctl status slapd
```

Erwarteter Zustand: `active (running)`.

## 5. Funktionstest

```bash
ldapsearch -x -H ldap://localhost \
  -s base -b "" namingContexts
```

Erwartete Ausgabe:

```text
dn:
namingContexts: dc=example,dc=test
```

## 6. Wichtige Verzeichnisse

| Pfad | Bedeutung |
|---|---|
| /etc/ldap/ | LDAP-Konfigurationsdateien |
| /etc/ldap/slapd.d/ | Dynamische Serverkonfiguration |
| /var/lib/ldap/ | LDAP-Datenbank |

## 7. Ergebnis

Der OpenLDAP-Server ist installiert und kann lokal abgefragt werden.

Im nächsten Kapitel wird die LDAP-Verzeichnisstruktur konfiguriert.
