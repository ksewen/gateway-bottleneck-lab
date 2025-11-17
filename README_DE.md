# Die zweite Testrunde

[English](./README.md) | [简体中文](./README_ZH.md)

## 🎯 Ziel

Nachdem der Engpass aus der ersten Testrunde (blockierender RestTemplate-Aufruf im Gateway) behoben wurde, sollte die
neue Durchsatzleistung des Gesamtsystems über das Gateway überprüft werden.
Das Ziel dieser Runde war:

- **die Wirkung** der Umstellung auf WebClient zu **messen**
- mögliche **weitere Engpässe** zu identifizieren
- eine neue Leistungsbasis für spätere Optimierungen zu schaffen

## Durchsatztest nach der Behebung

Nach der Umstellung auf WebClient wurde erneut ein fünfminütiger Durchsatztest über das Gateway durchgeführt:

```shell
wrk -t5 -c10 -d300s --timeout=10s --latency http://127.0.0.1:38071/service/hello
```

**Ergebnis**:

> Requests/sec: 4091.91

Originales Protokoll:

```shell
Running 5m test @ http://127.0.0.1:38071/service/hello
  5 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.47ms  772.41us  43.05ms   96.34%
    Req/Sec   822.41     55.76     0.92k    86.56%
  Latency Distribution
     50%    2.36ms
     75%    2.54ms
     90%    2.80ms
     99%    4.65ms
  1227963 requests in 5.00m, 149.90MB read
Requests/sec:   4091.91
Transfer/sec:    511.49KB
```

### Interpretation

- Der Durchsatz stieg von **~1.800 RPS** (vor der Behebung) auf **~4.091 RPS**.
- Das entspricht einer Steigerung um **über 125 %**.
- Die Latenzwerte haben sich ebenfalls **deutlich verbessert**.

Trotz der Verbesserung war der erwartete Wert (deutlich näher an der Baseline von ~13.000 RPS) noch nicht erreicht.
Daher war eine weitere Analyse notwendig, um den neuen Engpass zu finden.

## Vertiefte Analyse

Wie in der ersten Runde wurde erneut ein Profiling mit
[JProfiler](https://www.ej-technologies.com/jprofiler) durchgeführt.

Ein Snapshot der CPU-Zeitverteilung zeigte deutlich, dass **ein großer Teil der Ausführungszeit in der Logausgabe**
verbracht wurde:

![cpu-views-call-tree](https://raw.githubusercontent.com/ksewen/Bilder/main/202308161502704.png "CPU Views - Call Tree")

### Warum entsteht dieser Engpass?

Logback schreibt Logs **standardmäßig synchron**, d. h. der arbeitende Thread muss warten, bis der Logeintrag verarbeitet ist.   

Bei hoher Last könnte dies dazu führen:
- Blockierungen auf der kritischen Ausführungspfad
- unnötigen I/O-Wartezeiten
- Verlangsamung der Event-Loop
- sinkender Gesamt-Durchsatzleistung

Damit wurde klar:
**Nach der Behebung des ersten Engpasses war die synchrone Logausgabe der nächste dominierende Leistungsfaktor.**

## Behebung

Logback bietet einen [AsyncAppender](https://logback.qos.ch/manual/appenders.html#AsyncAppender),
mit dem die Logausgabe in Hintergrund-Threads ausgelagert wird.

Vorteile des AsyncAppender:
- **entlastet** den ausführenden Thread
- Logvorgänge werden **asynchron verarbeitet**
- **weniger Blockierungen** im kritischen Request-Flow
- bessere Skalierbarkeit bei hoher Last

Ein typisches Beispiel:

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE"/>
</appender>
```

Damit wandert das Schreiben der Logs aus der Hauptlogik heraus, und die Anfrageverarbeitung würde spürbar beschleunigt.

## Erwartetes Ergebnis

Nach der Aktivierung des AsyncAppender könnte:
- die Latenz weiter **sinken**
- der Durchsatz weiter **steigen**
- die Blockierungszeit im Gateway deutlich **geringer** werden

Die tatsächliche Wirkung wird in der [**nächsten Testrunde**](https://github.com/ksewen/gateway-bottleneck-lab/tree/0.0.3?tab=readme-ov-file) überprüft.

## 🟩 Gesamtfazit

- Die erste Korrektur (WebClient) war erfolgreich: **Durchsatz +125 %**.
- Mittels JProfiler wurde die **synchrone Logausgabe** als neuer Engpass identifiziert.
- Logback’s **AsyncAppender** bietet eine gezielte, mögliche Lösung, dessen Wirkung in der folgenden Testrunde überprüft wird.
