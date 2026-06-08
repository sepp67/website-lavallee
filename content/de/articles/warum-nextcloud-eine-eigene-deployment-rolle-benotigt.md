---
title: "Warum Nextcloud eine eigene Deployment-Rolle benötigt statt einer generischen Docker-Rolle"
date: 2026-06-08
author: "Sébastien Lavallée"
description: "Erfahrungsbericht über die Bereitstellung von Nextcloud in einer mit Ansible und Docker verwalteten Staging-Umgebung."
---

Im Rahmen der Neustrukturierung meiner Infrastruktur habe ich eine separate Staging-Umgebung aufgebaut, die vollständig von der Produktionsumgebung getrennt ist.

Das Ziel war einfach: Neue Versionen von Anwendungen zunächst in einer isolierten Umgebung testen, bevor sie in Produktion ausgerollt werden.

Die meisten meiner Anwendungen folgen demselben Muster:

* Container-Image aus der GitHub Container Registry herunterladen
* HTTP-Port veröffentlichen
* Reverse Proxy konfigurieren
* Anwendung testen

Dieses Modell funktioniert hervorragend für:

* Grav CMS
* Webcam Stream
* Facturier Demo
* Statische Websites

Dafür habe ich eine generische Ansible-Rolle erstellt:

```text
staging_docker_app
```

Diese Rolle:

1. lädt das Image herunter,
2. erstellt den Container,
3. veröffentlicht den Port,
4. startet den Dienst.

Für einfache Anwendungen ist dieser Ansatz effizient und leicht wartbar.

Nextcloud zeigte jedoch schnell die Grenzen dieses Modells auf.

## Die ursprüngliche Annahme

Zunächst ging ich davon aus, dass Nextcloud wie jede andere Anwendung behandelt werden kann.

Die Konfiguration sah wie folgt aus:

```yaml
app_name: nextcloud
app_image: ghcr.io/sepp67/ansible-role-nextcloud-stack:latest
app_container_name: nextcloud-staging
app_host_port: 18080
app_container_port: 80
```

Der Deployment-Prozess lief erfolgreich durch.

Der Container startete.

Der Reverse Proxy funktionierte.

Doch die Anwendung war nicht betriebsbereit.

## Das eigentliche Problem

Nextcloud besteht nicht aus einem einzelnen Container.

Mein Deployment umfasst:

```text
Nextcloud Stack
├── Nextcloud
├── PostgreSQL
├── Redis
└── Cron
```

Die Docker-Compose-Datei zeigt dies deutlich:

```yaml
services:
  db:
    image: postgres

  redis:
    image: redis

  app:
    image: nextcloud

  cron:
    image: nextcloud
```

Der Hauptcontainer benötigt:

* PostgreSQL für die Datenspeicherung,
* Redis für Cache und Dateisperren,
* Cron-Jobs für Hintergrundaufgaben.

Ein einzelner Container reicht daher nicht aus.

## Ein grundlegender Architekturunterschied

Meine generische Rolle basiert auf der Annahme:

```text
Eine Anwendung = Ein Container
```

Nextcloud folgt jedoch dem Modell:

```text
Eine Anwendung = Mehrere zusammenarbeitende Dienste
```

Das ist ein völlig anderes Deployment-Muster.

Versucht man beide Ansätze in einer einzigen Rolle abzubilden, entstehen:

* zahlreiche Sonderfälle,
* komplexe Bedingungen,
* schlechtere Wartbarkeit,
* geringere Lesbarkeit.

Die generische Rolle verliert dadurch ihren eigentlichen Zweck.

## Die bessere Lösung

Statt die bestehende Rolle zu verkomplizieren, habe ich eine eigene Rolle erstellt:

```text
staging_nextcloud_stack
```

Dadurch sind die Verantwortlichkeiten klar getrennt.

### Generische Rolle

```text
staging_docker_app
```

Für:

* Grav
* Webcam Stream
* Facturier Demo
* Statische Websites

Verantwortlich für:

* Image herunterladen
* Container erstellen
* Port veröffentlichen
* Dienst starten

### Nextcloud-Rolle

```text
staging_nextcloud_stack
```

Verantwortlich für:

* Persistente Verzeichnisse erstellen
* Umgebungsvariablen generieren
* Docker-Compose-Datei erzeugen
* PostgreSQL bereitstellen
* Redis bereitstellen
* Nextcloud bereitstellen
* Cron bereitstellen
* Gesamten Stack starten

## Wiederverwendung vorhandener Komponenten

Eine weitere wichtige Erkenntnis war die Vermeidung von doppelten Verantwortlichkeiten.

Meine Infrastruktur enthielt bereits die Rolle:

```text
docker_host
```

zur Installation von:

* Docker CE
* Docker Compose Plugin
* Buildx
* Containerd

Anfangs versuchte ich, Docker direkt in der Nextcloud-Rolle zu installieren.

Das führte zu unnötiger Redundanz.

Die endgültige Architektur lautet:

```text
docker_host
        │
        ▼
staging_nextcloud_stack
```

Die Docker-Installation erfolgt zentral.

Die Anwendungslogik bleibt in der jeweiligen Anwendungsrolle.

## Fazit

Automatisierung bedeutet nicht, alle Deployments identisch zu machen.

Automatisierung bedeutet, Deployments vorhersehbar, wartbar und nachvollziehbar zu gestalten.

Für einfache Anwendungen genügt eine generische Docker-Rolle.

Für komplexe Anwendungen wie Nextcloud ist eine dedizierte Deployment-Rolle die bessere Wahl.

Die wichtigste Erkenntnis lautet:

> Generische Rollen sollten generisch bleiben. Komplexe Anwendungen verdienen ihre eigene Deployment-Logik.
