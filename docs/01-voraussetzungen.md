# 01 – Voraussetzungen

## Ziel

In diesem Kapitel werden die technischen Voraussetzungen für die Installation eines OpenLDAP-Servers unter Debian 12 oder Debian 13 beschrieben.

## 1. Betriebssystem

Unterstützte Systeme:

- Debian GNU/Linux 12 (Bookworm)
- Debian GNU/Linux 13 (Trixie)

Für die Installation wird ein aktuelles Debian-System mit administrativen Rechten benötigt.

Die Debian-Version lässt sich mit folgendem Befehl überprüfen:

```bash
cat /etc/os-release
```

## 2. Hardware

Für eine kleine LDAP-Testumgebung werden folgende Ressourcen empfohlen:

| Ressource | Empfehlung |
|---|---|
| CPU | 1–2 vCPUs |
| Arbeitsspeicher | 2 GB RAM |
| Festplattenspeicher | 10 GB |
| Netzwerk | Statische IP-Adresse oder feste DHCP-Zuordnung |

## 3. Netzwerk und DNS

Für die Dokumentation verwenden wir folgende Beispielwerte:

| Einstellung | Beispiel |
|---|---|
| Hostname | ldap01 |
| FQDN | ldap01.example.test |
| IP-Adresse | 192.0.2.10 |
| LDAP-Basis-DN | dc=example,dc=test |
| Administrator-DN | cn=admin,dc=example,dc=test |

Die Werte dienen als Platzhalter und müssen an die eigene Testumgebung angepasst werden.

Die verwendete Beispiel-IP ist keine tatsächlich erreichbare Serveradresse.

## 4. Benötigte Software

Für die Installation werden folgende Debian-Pakete benötigt:

- `slapd` – OpenLDAP-Server
- `ldap-utils` – LDAP-Kommandozeilenwerkzeuge

Die Installation erfolgt im nächsten Kapitel.

## 5. Administrative Berechtigungen

Für die Installation und Konfiguration werden Root-Rechte beziehungsweise `sudo` benötigt.

Die Berechtigung kann beispielsweise mit folgendem Befehl geprüft werden:

```bash
sudo -v
```

## 6. Sicherheit

Folgende Grundsätze werden berücksichtigt:

- Keine Passwörter im Git-Repository speichern
- LDAP-Zugriffe später mit TLS absichern
- Konfigurationsänderungen dokumentieren
- Vor Änderungen Sicherungen erstellen
- LDAP-Dienste nur für benötigte Netzwerke freigeben

## 7. Ergebnis

Nach Abschluss dieses Kapitels sind die Systemanforderungen, Netzwerkparameter und benötigten Softwarepakete bekannt.

Die Installation von OpenLDAP wird im Kapitel `02-installation.md` beschrieben.
