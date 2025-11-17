# Die dritte Testrunde

[English](./README.md) | [简体中文](./README_ZH.md)

## 🎯 Ziel

Nach den Optimierungen aus der **zweiten Testrunde** sollte erneut überprüft werden, wie sich das Gesamtsystem über das
Gateway verhält.
Die Ziele dieser Runde waren:

- die **Auswirkung des AsyncAppender** realistisch zu bewerten
- **weitere mögliche** versteckte Engpässe aufzudecken
- Unterschiede zwischen diesem Labor-Setup und realen Produktionsumgebungen besser zu verstehen
- zu prüfen, ob das System nach den bisherigen Anpassungen näher an die erwartete Systemleistung herankommt

## Durchsatztest nach der Behebung und unerwartetes Ergebnis

Nach der Aktivierung des **AsyncAppender** wurde erneut ein fünfminütiger Durchsatztest ausgeführt:

**Ergebnis**:

> Requests/sec:   3919.34

Originales Protokoll:

```shell
Running 5m test @ http://127.0.0.1:38071/service/hello
  5 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.56ms  545.34us  29.72ms   92.53%
    Req/Sec   787.67     33.81     0.89k    77.66%
  Latency Distribution
     50%    2.48ms
     75%    2.69ms
     90%    2.94ms
     99%    4.09ms
  1176033 requests in 5.00m, 143.56MB read
Requests/sec:   3919.34
Transfer/sec:    489.92KB
```

Der Durchsatz lag damit ca. 4 % **unter** dem Wert der zweiten Testrunde.

### Interpretation

Obwohl der AsyncAppender theoretisch den Request-Thread entlasten sollte, hat er in diesem Setup zu einer **geringfügig
schlechteren** Performance geführt.
Eine plausible Erklärung liegt in den beschränkten CPU-Ressourcen (nur **zwei Cores** in diesem Labor-Setup), wodurch
der
zusätzliche Logger-Hintergrundthread mehr Kontextwechsel verursacht als Nutzen bringt.

In Produktionssystemen mit **mehr CPU-Kernen** wäre das Ergebnis wahrscheinlich anders.

## Vertiefte Analyse

### Reduzieren der Log-Ausgabe

Das Log-Level wurde auf **ERROR** reduziert, um möglichst wenig Einfluss durch I/O im Request-Pfad zu erzeugen.

Dies ist auch im realen Produktionsbetrieb eine übliche und praktikable Maßnahme, da moderne Microservice-Plattformen
**dynamische Log-Level-Änderungen** unterstützen.

Nach der Reduktion des Log-Levels auf ERROR ergab der erneute Test.

**Ergebnis**:

> Requests/sec:   4369.11

Originales Protokoll:

```shell
Running 5m test @ http://127.0.0.1:38071/service/hello
  5 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.31ms  761.12us  69.92ms   96.95%
    Req/Sec     0.88k    59.38     0.96k    85.50%
  Latency Distribution
     50%    2.21ms
     75%    2.36ms
     90%    2.56ms
     99%    4.36ms
  1311067 requests in 5.00m, 160.04MB read
Requests/sec:   4369.11
Transfer/sec:    546.14KB
```

Der Durchsatz **stieg** damit um etwa **6 %**.
Berücksichtigt man Messungenauigkeiten, deckt sich diese Verbesserung mit der in der
[letzten Runde](https://github.com/ksewen/gateway-bottleneck-lab/tree/0.0.2) beobachteten
CPU-Zeit-Belastung durch Logging.

Damit wurde bestätigt:

Die Reduktion der Log-Ausgabe wirkt sich direkt **positiv** auf den Durchsatz aus.

### Analyse nach Log-Optimierung

Um sicherzustellen, dass das Gateway keine weiteren strukturellen Engpässe aufweist, wurden CPU- und Thread-Profile
erneut überprüft:

![cpu-views-call-tree](https://github.com/ksewen/Bilder/blob/main/202308190029673.png?raw=true "CPU Views - Call Tree")

Beobachtungen:

- keine auffällig häufigen **BLOCK-** oder **WAIT-Zustände**
- keine Hinweise auf **Event-Loop-Blockierungen**

Damit war klar:
Das Gateway selbst wies nach der Log-Optimierung **keine offensichtlichen Bottlenecks** mehr auf.

## Weitere Erklärungen

### Verbesserung der Profiling-Strategie – Entdeckung eines neuen Engpasses

In den vorherigen Testrunden war der JProfiler-Paketfilter bewusst auf
`com.github.ksewen` eingeschränkt worden, um Zeit und Ressourcen zu sparen.

Für eine umfassendere Analyse wurde dieser Filter nun entfernt, damit der gesamte
Request-Flow — inklusive aller Bibliotheken, Framework-Komponenten und JVM-internen Abläufe —
sichtbar wird. Zusätzlich wurden Monitors & Locks aktiviert, um mögliche Blockierungen besser zu erkennen.

Erst durch diese erweiterte Sicht wurde ein zuvor verborgener Engpass sichtbar:

👉 die Blockierung bei der **UUID-Generierung**.

### Engpass bei der UUID-Generierung

Im Filter
[TracingGlobalFilter](https://github.com/ksewen/gateway-bottleneck-lab/blob/0.0.3/gateway-for-test/src/main/java/com/github/ksewen/performance/test/gateway/filter/tracing/TracingGlobalFilter.java)
wird für jede Anfrage eine neue UUID erzeugt: `UUID.randomUUID()`

Unter Linux verwendet diese Methode standardmäßig `SecureRandom`.
Wenn `SecureRandom` auf eine blockierende Quelle wie /dev/random zugreift und die Entropie knapp ist, können
Wartezeiten entstehen.

Dies wurde im Profiling sichtbar:

![Blockierung-1](https://raw.githubusercontent.com/ksewen/Bilder/main/202308201438817.png)

![Blockierung-2](https://raw.githubusercontent.com/ksewen/Bilder/main/202308201439720.png)

![Statistik der Blockierung](https://raw.githubusercontent.com/ksewen/Bilder/main/202308201439000.png)

### Behebung in diesem Labor

Um die Blockierung zu vermeiden, wurde die JVM mit folgender Option gestartet:

```shell
-Djava.security.egd=file:/dev/urandom
```

Damit nutzt `SecureRandom` eine **nichtblockierende Entropiequelle**, wodurch die UUID-Generierung stabiler wird.

### Zusammenhang mit Java-Versionen (*JEP 273*)

Dieses Verhalten hängt auch mit der **Java-Version** zusammen:

- Produktionssystem damals: **Java 8**
- Dieses Labor: **Java 17**

Mit **Java 9** wurde `SecureRandom` gemäß *JEP 273* grundlegend überarbeitet:

📌 https://openjdk.org/jeps/273

Dadurch unterscheidet sich das Verhalten der UUID-Generierung zwischen **Java 8** und **Java 17** deutlich.

> Eine vertiefte Analyse des Problems unter Java 8 sowie eine konkrete Lösung finden Sie in meinem Projekt:  
> 👉 [**uuid-benchmark**](https://github.com/ksewen/uuid-benchmark)

### Erwartetes Ergebnis

Nach der Anpassung wurde erwartet:

- stabile UUID-Generierung **ohne Blockierung**
- **geringere Latenzen** bei hoher Last
- **leichter Anstieg** des Durchsatzes

> Erklärung:
> Die endgültige Validierung bleibt jedoch **aufgrund begrenzter Ressourcen** in diesem Labor-Setup eingeschränkt.

## 🟩 Gesamtfazit

- Der AsyncAppender zeigte in dieser Runde **keine Verbesserung** und führte zu einer **Leistungsverschlechterung von 4 %**.
- Durch das Reduzieren des Log-Levels konnte der Durchsatz auf **6 % erhöht** werden.
- Das Gateway selbst weist nach den Anpassungen **keine klaren Engpässe** mehr auf.
- Durch eine verfeinerte Profilierung wurde die **UUID-Generierung** als versteckter Engpass identifiziert.
- Die Lösung mittels `egd=/dev/urandom` ist im **Java-17-Kontext** wirksam, **unterscheidet** sich aber vom
  **Java-8-Verhalten** (*JEP 273*).
- Diese Runde zeigt eindeutig, wie wichtig Profiling-Strategien, Log-Management, JVM-Wissen und Umgebungsfaktoren für
  Performanceanalysen sind.
- Und erneut bestätigt sich:
  Performance ist **kontextabhängig**, **methodisch** und **selten perfekt reproduzierbar**.