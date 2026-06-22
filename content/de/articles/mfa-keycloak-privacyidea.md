---
title: "MFA einführen, ohne Bestehendes zu zerstören: eine Architekturperspektive"
date: 2026-06-22
description: "Die Einführung von MFA ist der erste sichtbare Baustein eines Identitätsinfrastruktur-Projekts"
---

## Ein Problem, das sich nicht mit einem einfachen Plugin lösen lässt

Vielleicht kommt Ihnen diese Situation bekannt vor. Ihr Authentifizierungssystem basiert auf einem OpenLDAP-Verzeichnis, das seit Jahren im Einsatz ist. Es betreibt Ihre Nextcloud-Instanz, vielleicht auch weitere interne Anwendungen. Alles funktioniert. Niemand möchte daran etwas ändern.

Und dann kommt die Frage, die in Führungsrunden immer wieder auftaucht: "Warum ist MFA noch nicht überall aktiviert?"

Die ehrliche Antwort lautet oft: Weil niemand weiß, wie man das umsetzt, ohne alles zu gefährden.

Gestohlene Zugangsdaten sind mittlerweile der häufigste Einstiegspunkt bei Sicherheitsvorfällen. Ein Passwort allein, egal wie komplex, reicht nicht mehr aus, um einen Zugang zu schützen. Cyber-Versicherer wissen das, Auditoren wissen das, und immer mehr Kunden und Partner fragen explizit danach, bevor sie einen Vertrag unterschreiben. Die Multi-Faktor-Authentifizierung (MFA) ist längst keine Kür mehr – sie ist zur Grunderwartung geworden.

Die Frage ist nicht, ob man es tun muss. Die Frage ist, wie man es tut, ohne alles andere zu riskieren.

## Drei Wege, von denen nur einer auf die Zukunft vorbereitet

Betrachtet man die Möglichkeiten, MFA in eine bestehende Infrastruktur einzuführen, stößt man in der Regel auf drei Wege.

**Migration zu einer All-in-One-Identitätslösung.** Das verspricht eine moderne, vollständig integrierte Plattform, die alles abdeckt. In der Praxis ist es jedoch ein langes, teures und riskantes Projekt: Das Verzeichnis muss migriert werden, jede Anwendung neu konfiguriert, Teams müssen geschult werden – und man muss eine Übergangsphase akzeptieren, in der vieles schiefgehen kann.

**Ein MFA-Plugin für eine einzelne Anwendung hinzufügen.** In unserem Fall bietet PrivacyIDEA ein natives Plugin für Nextcloud an. Es ist schnell, kostengünstig und funktioniert. Aber es löst das Problem nur für eine einzige Anwendung. Wenn morgen ein VPN, eine weitere Webanwendung oder der Zugang zu Servern geschützt werden muss, beginnt man wieder von null – mit einem anderen Werkzeug, einer anderen Logik, einem weiteren Wartungsaufwand.

**Eine zentrale Identitätsschicht hinzufügen, ohne das Bestehende anzufassen.** Auf den ersten Blick die am wenigsten naheliegende Option, da sie die Einführung neuer Komponenten erfordert. Aber es ist die einzige Option, die weder das Verzeichnis noch die Anwendungen verändert – und die das Fundament legt, um den Rest der Infrastruktur später ebenfalls abzusichern.

Wir haben uns für den dritten Weg entschieden. Das ist der Kern dieses Artikels: warum wir diese Entscheidung getroffen haben, wie sie technisch umgesetzt wird, und was sie einmal etabliert ermöglicht.

## Die Architektur: zwei klar getrennte Rollen

Die wichtigste Entscheidung in diesem Projekt war nicht die Wahl eines Werkzeugs, sondern die Entscheidung für eine **klare Trennung der Zuständigkeiten**. Statt nach einem einzigen Produkt zu suchen, das alles erledigt, haben wir zwei Komponenten zwischen Verzeichnis und Anwendungen eingeführt – jede mit einer präzisen Aufgabe.

### Keycloak als zentraler Durchgangspunkt

Die erste Komponente ist **Keycloak**, eingesetzt als *Identity Provider*. Seine Aufgabe: zum einzigen Punkt zu werden, durch den jede Authentifizierung der Organisation läuft. Keycloak verbindet sich mit dem bestehenden OpenLDAP-Verzeichnis – ohne es zu verändern – und ist nun derjenige, der Anmeldeanfragen entgegennimmt, anstatt dass dies jede Anwendung einzeln tut.

Konkret bedeutet das: Nextcloud prüft ein Passwort nicht mehr selbst gegen das Verzeichnis. Diese Prüfung wird an Keycloak delegiert. Für die Endnutzer ändert sich äußerlich nichts. Für die Organisation ändert sich strukturell alles: Es gibt jetzt eine einzige Stelle, an der über die Authentifizierungsrichtlinie entschieden wird, statt einer Regel pro Anwendung.

### PrivacyIDEA, der Spezialist für den zweiten Faktor

Die zweite Komponente ist **PrivacyIDEA**. Hier lohnt sich Präzision, denn das ist oft eine Quelle von Verwechslungen: PrivacyIDEA ist **kein** Identity Provider. Es ist ein **Token-Server**. Seine Rolle beschränkt sich auf eine Sache – aber die erfüllt er sehr gut: das Registrieren eines zweiten Faktors (OTP-Token, mobile App usw.) für jeden Benutzer im Verzeichnis und das Prüfen des bei der Anmeldung eingegebenen Werts.

Keycloak selbst bietet nativ einen Mechanismus zur Token-Registrierung an. Warum also nicht einfach dabei bleiben? Weil seine Möglichkeiten auf einfache Szenarien begrenzt bleiben. PrivacyIDEA hingegen ist darauf ausgelegt, **Authentifizierungs-Workflows** zu verwalten – und genau das rechtfertigt seine Rolle in der Architektur, anstatt sich allein auf Keycloak zu verlassen.

## Authentifizierung als Prozess, nicht als Schalter

Authentifizierung ist kein binärer Vorgang. Sie ist eine Kette von Prüfungen, die je nach Bedarf der Organisation sehr einfach bleiben oder sehr komplex werden kann. Dieser Punkt wird häufig übersehen, wenn MFA wie ein einfacher Schalter behandelt wird, den man umlegt – während es sich in Wirklichkeit um einen zu entwerfenden Workflow handelt.

Im einfachsten von uns umgesetzten Szenario:
- Keycloak prüft das Passwort direkt gegen das OpenLDAP-Verzeichnis.
- PrivacyIDEA prüft parallel nur den Wert des Einmalcodes (OTP).

Jede Komponente erledigt eine Aufgabe – und erledigt sie gut. Die Architektur erlaubt aber mehr: Das Passwort kann auch an PrivacyIDEA weitergeleitet werden, das es dann als Auslöser für komplexere Szenarien nutzen kann – etwa einen anderen Faktor je nach Benutzerprofil zu verlangen, die Richtlinie an den Anmeldekontext anzupassen oder mehrere Prüfungen zu verketten. Diese Flexibilität bietet der native Mechanismus von Keycloak allein derzeit nicht.

Ein Punkt, der bei MFA-Einführungen oft übersehen wird: Was passiert mit einem Benutzer, der sich zum ersten Mal anmeldet, ohne bereits einen zweiten Faktor registriert zu haben? Diesen Fall muss die Architektur explizit behandeln – der Workflow muss das Fehlen eines Tokens erkennen und einen Registrierungsprozess auslösen, statt den Benutzer schlicht zu blockieren. Ein Detail, das aber oft darüber entscheidet, ob eine MFA-Einführung reibungslos verläuft oder eine Flut von Support-Tickets erzeugt.

Zu diesem Zeitpunkt musste keine der bestehenden Anwendungen grundlegend neu konfiguriert werden. OpenLDAP bleibt unverändert das Referenzverzeichnis. Nextcloud funktioniert weiterhin normal – lediglich neu verbunden mit Keycloak statt direkt mit dem Verzeichnis.

## Warum diese scheinbar "schwerere" Architektur tatsächlich die richtige Entscheidung ist

Hier zeigt sich die eigentliche Architektenlogik. Ein speziell für Nextcloud entwickeltes MFA-Plugin hätte das unmittelbare Problem schneller gelöst. Aber es hätte auch die Tür zu allem verschlossen, was nach der Einführung von MFA natürlicherweise folgt: die Absicherung weiterer Anwendungen, weiterer Zugriffsarten – mit derselben Konsistenz.

Indem wir Keycloak ins Zentrum gestellt haben, haben wir nicht nur Nextcloud geschützt. Wir haben einen zentralen Durchgangspunkt geschaffen, der die Sicherheitsanforderungen der kommenden Jahre aufnehmen kann, ohne neu aufgebaut werden zu müssen. Das ist der Unterschied zwischen einer punktuellen Lösung und einer Investition in die Infrastruktur – und genau diese Abwägung muss ein Architekt explizit treffen, bevor der erste Stein gelegt wird, statt sie erst zu entdecken, wenn ein zweiter Bedarf entsteht.

### OpenID Connect: die Tür zu Webanwendungen öffnen

Sobald Keycloak etabliert ist, eröffnet sich eine neue Möglichkeit: jede mit **OpenID Connect** (OIDC) kompatible Webanwendung anzubinden.

OpenID Connect ist ein offenes Protokoll – ähnlich wie HTTP es für das Web ist –, das definiert, wie eine Anwendung die Authentifizierung ihrer Benutzer an einen externen Identity Provider delegiert. Es basiert auf drei Elementen:

- Der **OpenID Provider (OP)**: das ist Keycloak. Er authentifiziert den Benutzer und stellt ein Token aus.
- Der **ID Token**: ein Token im JWT-Format, das bestätigt, dass der Benutzer authentifiziert wurde, und einige seiner Informationen transportiert.
- Die **Relying Party (RP)**: die Client-Anwendung – diejenige, die der Benutzer tatsächlich nutzen möchte. Sie vertraut dem von Keycloak ausgestellten Token, anstatt die Authentifizierung selbst zu übernehmen.

Was dieses Protokoll für eine Organisation wertvoll macht, ist das, was es nicht verändert. Eine über OpenID Connect angebundene Anwendung behält ihre eigene Konfiguration, ihre eigenen Benutzergruppen, ihre eigenen internen Berechtigungen. Nur die Authentifizierung wird delegiert. Während die Integration eines neuen Authentifizierungssystems normalerweise Wochen dedizierter Entwicklung erfordert, lässt sich eine OIDC-kompatible Anwendung in der Regel innerhalb weniger Stunden an Keycloak anbinden.

### RADIUS: dieselbe Logik für VPNs und Server

Die Web-Authentifizierung ist nicht das einzige Feld, das abzusichern ist. VPN-Zugänge und Verbindungen zu virtuellen Maschinen basieren in der Regel auf einem anderen Protokoll: **RADIUS**.

Dieselbe Identitätsarchitektur, die für Web-MFA aufgebaut wurde, lässt sich natürlich auf diese Anwendungsfälle ausdehnen. Das Prinzip bleibt gleich: ein zentraler Authentifizierungspunkt, der ein Passwort und einen zweiten Faktor prüfen kann – unabhängig vom Kanal, sei es eine Webseite, eine VPN-Verbindung oder ein Serverzugang.

Hier zahlt sich die anfängliche Investition wirklich aus. Die Organisation setzt nicht ein MFA-Tool für Nextcloud, ein weiteres für das VPN und ein drittes für Server ein. Sie setzt eine einzige Authentifizierungsrichtlinie um, die überall dort konsistent angewendet wird, wo sie benötigt wird.

## Das Fazit

Dieses Projekt war nie wirklich ein MFA-Projekt. Es war ein Identitätsinfrastruktur-Projekt, dessen erster sichtbarer Baustein MFA war.

Diese Unterscheidung ist entscheidend, wenn es um die Bewertung der Arbeit geht: Ein MFA-Projekt endet, sobald der zweite Faktor funktioniert. Ein Identitätsinfrastruktur-Projekt erzeugt weiterhin Wert – mit jeder neu angebundenen Anwendung, jedem neu abgesicherten Zugang –, ohne dass man jedes Mal wieder bei null anfangen muss.

Genau das ist die Art von Architektur, die wir entwerfen und umsetzen können: von einem bestehenden System ausgehen, ohne es zu zerstören, und darauf eine Identitätsschicht aufbauen, die die Sicherheitsanforderungen der kommenden Jahre aufnehmen kann.

Wenn sich Ihre Organisation in dieser Situation befindet – ein OpenLDAP, eine Nextcloud oder ein anderes bestehendes System, und ein MFA-Bedarf, der wie der erste einer langen Reihe von Sicherheitsanforderungen aussieht – lassen Sie uns sprechen. Genau bei solchen Projekten unterstützen wir Organisationen.
