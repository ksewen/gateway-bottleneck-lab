# Die erste Testrunde

## Baseline-Benchmark des Backend-Services

**Der erste Schritt** jeder Performanceanalyse ist die **Ermittlung der maximalen Durchsatzrate eines einzelnen
Kernservices**.
Nur wenn die Basisleistung bekannt ist, lassen sich spätere Abweichungen – z. B. über das Gateway – sauber
interpretieren.

Für diesen Baseline-Benchmark wurde folgender Befehl mittels [**wrk**](https://github.com/wg/wrk) ausgeführt:

```shell
wrk -t5 -c10 -d300s --timeout=10s --latency http://127.0.0.1:38072/hello
```

Während des Tests wurden mit docker stats die **Systemressourcen** überwacht:

```shell
docker stats ${CONTAINER}
# oder Werte protokollieren:
docker stats ${CONTAINER} | awk '{print strftime("%m-%d-%Y %H:%M:%S",systime()), $0}' >> ${FILE_PATH_AND_NAME}
```

**Ergebnis**:

> Requests/sec:  12994.82

Originales Protokoll:

```shell
Running 5m test @ http://127.0.0.1:38072/hello
  5 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     3.08ms    6.10ms  52.91ms   87.33%
    Req/Sec     2.61k   219.70     3.65k    79.65%
  Latency Distribution
     50%  583.00us
     75%  779.00us
     90%   12.39ms
     99%   26.01ms
  3899171 requests in 5.00m, 469.24MB read
Requests/sec:  12994.82
Transfer/sec:      1.56MB
```

Die CPU-Auslastung des Backend-Services lag dabei **über 100 %**, was auf eine vollständig ausgelastete Instanz
hinweist.
Damit steht die **Basisleistung: ~13 000 RPS** fest.

## Durchsatztest über das Gateway

**Im zweiten Schritt** wurde die folgende Route über das Gateway getestet:

```shell
wrk -t5 -c10 -d300s --timeout=10s --latency http://127.0.0.1:38071/service/hello
```

**Ergebnis:**

> Requests/sec:   1811.16

Originales Protokoll:

```shell
Running 5m test @ http://127.0.0.1:38071/service/hello
  5 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     6.29ms    6.75ms 206.86ms   97.07%
    Req/Sec   363.95     67.84   545.00     85.73%
  Latency Distribution
     50%    5.17ms
     75%    6.52ms
     90%    8.22ms
     99%   43.07ms
  543438 requests in 5.00m, 66.34MB read
Requests/sec:   1811.16
Transfer/sec:    226.39KB
```

CPU-Auslastung während des Tests:

- Gateway: **~65 %**
- Business-Service: **~20 %**
- Authentication-Service: **~30 %**

### Interpretation

Der Durchsatz fällt beim Zugriff über das Gateway von **~13 000 RPS** (Baseline) auf nur **~1 800 RPS**.

Da gleichzeitig die CPU-Last aller Komponente **nicht voll ausgelastet** ist, könnte dies auf einen internen
blockierenden Engpass im Gateway hindeuten, der:

- **hohe** Wartezeiten auslöst
- die Event-Loop **bremst**
- den Gesamtdurchsatz **begrenzt**

Um die genaue Ursache zu identifizieren, ist eine tiefere Analyse mithilfe Tools erforderlich.

## Warum wrk

Für diese Testreihe wurde bewusst wrk eingesetzt, weil es:

- **extrem leichtgewichtig** ist (ideal für reproduzierbare synthetische Tests)
- sehr **geringe CPU-Overheads** hat
- **präzise Latenzstatistiken** liefert (inkl. Percentiles und Verteilung)
- sich **für engpassorientierte Micro-Benchmarks eignet**
- perfekt geeignet ist, wenn man schnell **Pipeline-Verhalten** testen möchte

Gerade bei einer Performanceuntersuchung, die sich auf **Durchsatz** und **Latenzverteilung** konzentriert, eignet sich
wrk
hervorragend, ohne dabei das Testsystem mit zu viel Tool-Overhead zu beeinflussen.

## Vertiefte Analyse mit JProfiler

Da die Gateway-Auslastung kaum anstieg, wurde ein Profiling notwendig.
Hierfür wurde [**Jprofiler**](https://www.ej-technologies.com/products/jprofiler/overview.html) verwendet.  
JProfiler bietet u. a.:

- CPU-Profiling
- Thread-Profiling
- Call-Tree-Analyse
- Heap-Analyse

> Hinweis:
> Aufgrund seines Overheads darf JProfiler nicht in Produktionsumgebungen eingesetzt werden – in einem
> reproduzierbaren Testsetup wie diesem aber ideal.

### Konfiguration

Der JProfiler-Agent wurde bereits im Docker-Image des Gateways hinterlegt;
die Verbindung erfolgte **über den Port 127.0.0.1:38849**.

Zusätzlich wurde ein Call-Tree-Filter auf das Paket: `com.github.ksewen` gesetzt, da der Engpass mit hoher
Wahrscheinlichkeit dort lag.

Während des Profilings wurde erneut ein wrk-Test ausgeführt, um aussagekräftige Daten zu sammeln.

### Analyse

Der Call Tree zeigt deutlich, dass die Methode

`org.springframework.web.client.RestTemplate.postForEntity`

den größten Teil der CPU-Zeit verbraucht:

![cpu-views-call-tree](https://raw.githubusercontent.com/ksewen/Bilder/main/202308161502917.png
"CPU Views - Call Tree")

Damit bestätigt sich der Verdacht:

❗ **Im reaktiven WebFlux-Kontext wurde ein synchroner, blockierender RestTemplate mit HttpClient verwendet.**

Dies führt in einer nicht-blockierenden Reactor-Pipeline zu:

- Thread-Blockierungen
- erzwungenem Kontextwechsel
- saturierten Worker-Pools
- massiven Latenzen
- drastischem Durchsatzverlust

### Behebung

Die korrekte Lösung besteht darin, den blockierenden RestTemplate vollständig durch **WebClient** zu ersetzen, da
dieser:

- **non-blocking I/O (Reactor Netty)** verwendet
- nativ für WebFlux entworfen wurde
- die Event-Loop nicht blockiert

Damit könnten:

- die **Latenzen deutlich reduziert**
- der **Durchsatz stark erhöht**

## 🟩 Gesamtfazit

Die erste Testrunde zeigt deutlich:

- **Baseline-Benchmark (~13k RPS)** beweist, dass das Backend leistungsfähig ist.
- **Durchsatztest über das Gateway (~1.8k RPS)** zeigt einen massiven internen Engpass.
- JProfiler bestätigt den **blockierenden RestTemplate-Aufruf** als Ursache.
- Eine mögliche Maßnahme besteht im **Wechsel zum WebClient** (reaktiv / non-blocking), dessen Wirkung in den folgenden
  Branches überprüft wird.