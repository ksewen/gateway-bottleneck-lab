# Gateway Bottleneck Lab

[English](./README.md) | [简体中文](./README_ZH.md)

> Basierend auf einem realen Produktionsproblem (vereinfacht). Dieses Projekt zeigt, warum kleine, gezielte
> Performance-Tests – insbesondere auf Komponenten- oder Integrationsebene – helfen, versteckte Engpässe früh zu
> erkennen
> und kostspielige Probleme in großen Systemen zu vermeiden.

## 🎯 Ziel des Projekts

Dieses Repository stellt eine **minimal reproduzierbare Umgebung** bereit, die typische Leistungsengpässe in einer
verteilten Architektur (Gateway → Authentication → Business Logic) demonstriert.

Der Fall basiert auf einem **realen Produktionsproblem**, wurde jedoch vollständig **bereinigt**, **vereinfacht** und
**ohne vertrauliche Inhalte** nachgebaut.

Das Projekt dient als technische Referenz für Entwicklerinnen und Entwickler, um folgende Aspekte besser zu verstehen:

- warum feingranulare Performance-Tests - von **Benchmarking** von einzelner Komponente bis hin zu **Integration** mit
  verschiedenen Komponenten genauso wichtig sind wie **systemweiten Performance-Tests**.
- wie sich **reproduzierbare Testumgebungen** aufbauen lassen, um Leistungsprobleme **frühzeitig** zu entdecken.

## 🧱 Projektüberblick

Dieses Projekt bildet eine **vereinfachte**, aber **realistische** Servicekette ab, bestehend aus:

- **gateway-for-test** – API Gateway
- **auth-service-for-test** – Authentifizierungsservice
- **service-for-test** – Business Service

Alle Komponenten sind vollständig **containerisiert** und können **unabhängig voneinander** oder gemeinsam **über Docker
ausgeführt** werden.

In den Branches, die den vollständigen Beispielcode enthalten (z.B. **0.0.1**, **0.0.2** usw.), befindet sich im
Verzeichnis **resources/** eine `docker-compose.yml`, mit der die gesamte Testumgebung schnell gestartet werden kann.

Der **main**-Branch enthält ausschließlich die Projektbeschreibung und keine lauffähigen Artefakte.

Eine einfache Darstellung der Architektur:

![Architektur](https://raw.githubusercontent.com/ksewen/Bilder/main/20251116160740231.png)

### Branches

Die verschiedenen Branches enthalten unterschiedliche Zustände des Systems:

- Der Branch [**0.0.1**](https://github.com/ksewen/performance-test-example/tree/0.0.1) enthält bewusst einen
  reproduzierbaren Leistungsengpass. Die Analyse zeigt, dass die Ursache im Einsatz eines **RestTemplate mit synchronem
  HttpClient innerhalb WebFlux** liegt, was unter Last zu deutlichen Latenzen führt.
- Der Branch [**0.0.2**](https://github.com/ksewen/performance-test-example/tree/0.0.2) behebt den Engpass aus dem
  Branch [**0.0.1**](https://github.com/ksewen/performance-test-example/tree/0.0.1) und enthält eine Bewertung der
  Optimierungsergebnisse. Dabei wurde zusätzlich festgestellt, dass **Logback beim Schreiben von Logs spürbare
  Performance-Kosten verursacht**.
- Der Branch [**0.0.3**](https://github.com/ksewen/performance-test-example/tree/0.0.3) korrigiert und analysiert die im
  Branch [**0.0.2**](https://github.com/ksewen/performance-test-example/tree/0.0.2) erkannten Probleme. Er enthält zudem
  eine Lösung sowie detaillierte Informationen zur **Blockierung bei der UUID-Generierung**.

Die jeweiligen Branches enthalten ausführliche Erläuterungen zu Testmethoden und Messergebnissen.

## 🧭 Zentrale Erkenntnisse zur Performanceanalyse

Während der Arbeit an diesem Projekt hat sich klar gezeigt, dass Performanceanalysen **nicht** darauf abzielen, **ein einzig
„richtiges“ Ergebnis zu reproduzieren**. In realen Systemen hängt das Verhalten stark von der jeweiligen Umgebung ab –
etwa CPU-Leistung, Netzwerktopologie, Container-Runtime oder verwendeten Softwareversionen.

Daraus ergeben sich drei zentrale Prinzipien, die in jedem Performance- oder Bottleneck-Lab berücksichtigt werden
sollten:

### Komponenten- oder Integrationstests garantieren keine perfekte Systemperformance

Selbst wenn ein einzelner Dienst oder eine Komponente im Benchmark optimal läuft, bedeutet das nicht, dass sich
dieselbe Performance in einem vollständigen, realen System erzielen lässt.

Der Nutzen dieser feingranularen Tests liegt vor allem darin:

- **frühzeitig offensichtliche Fehler zu entdecken**, bevor sie in komplexen Systemen teuer und schwer reproduzierbar
  werden
- **Kosten zu sparen**, weil Probleme bereits auf Komponenten- oder Integrationsebene sichtbar werden
- Entwicklerinnen und Entwicklern – insbesondere jenen, die **allgemeine oder wiederverwendbare Komponenten**
  schreiben –
  ein zuverlässiges Werkzeug für Qualitäts- und Robustheitskontrolle an die Hand zu geben

Diese Tests ersetzen nicht die Systemtests, aber sie bilden die Grundlage für jede solide Performanceanalyse.

### Performance ist immer kontextabhängig

Testergebnisse unterscheiden sich je nach Umgebung oft deutlich.
Selbst kleine Änderungen – andere Hardware, angepasste Thread-Pools, veränderte Netzwerklatenzen – könnten das Verhalten
spürbar beeinflussen.

Performanceanalysen müssen daher **mehrmals** und **unter realistischen Bedingungen** durchgeführt werden, um aussagekräftig zu
sein.

### Optimierung ist immer eine Kosten-Nutzen-Abwägung

In vielen Anwendungsszenarien ist es ebenso wichtig, den Nutzen einer Optimierung mit den Kosten einer einfachen
horizontalen Skalierung oder Knoten-Erweiterung zu vergleichen.
Systemweite Performanceoptimierung kann sehr aufwendig werden:
- Aufbau einer 1:1 oder proportional skalierten **Testumgebung**
- Erstellen oder Importieren von repräsentativen **Testdaten**
- Durchführung **mehrerer Testzyklen**
- hoher **Zeit-** und **Personalaufwand** während der Analyse

Dabei sollte stets bedacht werden:

👉 **Ziel ist „ausreichende“ Performance, nicht maximale Performance.** 
Eine Optimierung ist nur dann sinnvoll, wenn ihr tatsächlicher Nutzen den technischen und betrieblichen Aufwand rechtfertigt.

Dieses Projekt zeigt genau diese **Methodik**:  
Nicht das Erreichen einer bestimmten Zahl ist das Ziel, sondern das systematische Identifizieren, Analysieren und
Beheben von Engpässen – und das realistische Übertragen dieser Vorgehensweise auf produktive Systeme.

## 🐳 Ausführung mit Docker

### Docker-Images erstellen

Im Hauptverzeichnis können die Images aller Services durch die bereitgestellten Skripte gebaut werden:

```shell
gateway-for-test/resources/scripts/build-image.sh -d .
service-for-test/resources/scripts/build-image.sh -d .
auth-service-for-test/resources/scripts/build-image.sh -d .
```

### Umgebung starten

```shell
cd resources && \
docker-compose --compatibility -f docker-compose.yml up
```

## 🧪 Funktionsprüfung

Das Gateway stellt eine einfache Test-Route bereit, die zur Funktionsprüfung aufgerufen werden kann:

```shell
curl http://127.0.0.1:38071/service/hello
```

Für weitere Tests kann die Anwendung selbstverständlich auch mit externen Tools simuliert werden, um parallele Zugriffe
oder höhere Last zu erzeugen.
Ich nutze dafür ein leichtgewichtiges Tool wie [**wrk**](https://github.com/wg/wrk), da es sich gut für einfache und
reproduzierbare Lastszenarien eignet.

Ausführliche Beispiele und Messergebnisse befinden sich in den jeweiligen Branches (z.B. **0.0.1**, **0.0.2**).
