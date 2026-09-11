# DwarfStar4 / ds4: DeepSeek V4 Flash auf dem Mac Studio M4 Max mit 128 GB

## Konsolidierte Umsetzungs- und Betriebsanleitung

**Master:** `dwarfstar4-deepseek-v4-flash-macstudio-guide-v2.md`
**Ergänzt und kritisch korrigiert aus:** `DwarfStar4-ds4_Anleitung_MacStudio_M4Max_128GB.md`
**Stand der Konsolidierung:** 07.08.2026; Upstream-Recheck 07.08.2026
**Zielsystem:** Mac Studio M4 Max `Mac16,9`, 128 GiB Unified Memory, macOS, Metal
**Projektverzeichnis:** `/Users/michaelkuebbeler/ai-workspace/localAI/ds4`
**Externe SSD:** Samsung 990 PRO 1 TB im OWC StudioStack, APFS, gemountet als `/Volumes/AIModels`; beobachtete freie Kapazität ca. 931 GiB.
**Gemessener SSD-Read:** 6,36 GB/s dezimal bzw. 5,92 GiB/s bei einem 8-GiB-`dd`-Read nach `purge`; beobachteter TB-Link: 80 Gb/s. Das ist ein Ist-Messwert dieses Laufs, aber noch kein fairer Beweis, dass die externe SSD schneller als die interne SSD ist, weil in v2 kein identischer interner A/B-Test dokumentiert ist.
**Aktuelle Modellartefakte:** Das HF-Repository führt Flash-Dateien mit dem Suffix `-0731`; der aktuelle ds4-Downloader verwendet dafür die Targets `ds4f-q2`, `ds4f-q2-q4`, `ds4f-q4` und `ds4f-dspark`.[10][11]

**Aktueller Upstream-Recheck:** Der `main`-Branch stand am 07.08.2026 bei Commit `b0309611041655f4e45671cfd9c9886aff161406`. Gegenüber dem zuvor gepinnten Stand wurden die Flash-Downloadtargets auf `ds4f-*` umbenannt und auf die `-0731`-Dateien umgestellt. Vor jedem Download deshalb zuerst `./download_model.sh --help` ausführen.[11]

> **BLUF:** Die reale SSD ist schnell genug und als Modellablage für den residenten 0731-Hybrid sinnvoll; für Q4-SSD-Streaming liegt das Modell zwingend auf der SSD, während der KV-Disk-Cache zunächst getrennt und anschließend per A/B-Test bewertet wird. Der `-0731`-Q2/Q4-Hybrid bleibt der bevorzugte Standardpfad, Q4 bleibt ein kontrolliertes Streaming-Experiment.

---

## 1. Geltungsbereich und Zielbild

Diese Anleitung beschreibt die lokale Nutzung der modellspezifischen DwarfStar4-/ds4-Engine für DeepSeek V4 Flash. ds4 ist kein allgemeiner GGUF-Runner. Es dürfen nur die im ds4-Projekt und im offiziellen Repository `antirez/deepseek-v4-gguf` vorgesehenen Artefakte verwendet werden.[1][2][4]

Das Ziel ist eine schrittweise, messbare Inbetriebnahme:

1. Voraussetzungen und Speicher prüfen.
2. Repository im bestehenden Projektverzeichnis reproduzierbar bereitstellen.
3. Metal-Binaries bauen und lokal prüfen.
4. Ohne externe SSD zunächst Q2 als Referenzpfad testen.
5. Nach Anschluss der SSD das Modellverzeichnis optional auf die SSD verlagern.
6. Danach Q2/Q4-Hybrid resident testen.
7. Q4 ausschließlich mit `--ssd-streaming` untersuchen.
8. Erst nach erfolgreicher Direktnutzung Server, KV-Disk-Cache und LiteLLM integrieren.
9. Optional MTP oder DSpark separat und nur nach einer stabilen Baseline testen.

### Nichtziele

- DeepSeek V4 PRO auf diesem 128-GB-System produktiv betreiben.
- MXFP4 vor einer nachgewiesenen Runtime-Unterstützung einsetzen.
- Q4 resident laden.
- MTP/DSpark mit SSD-Streaming kombinieren.
- Parallele Multi-Agent-Inferenz als Leistungsversprechen.
- SSD-Bandbreite oder SSD-Streaming-Performance vor Anschluss und Messung als gegeben darstellen.

---

## 2. Kritische Konsolidierung und Aktualisierungen gegenüber der Vorversion

Die neue `_v2`-Anleitung ist die argumentative Masterfassung. Die alternative Anleitung bleibt als Ergänzung für eine schrittweise Installation relevant. Die Konsolidierung übernimmt die realen SSD-Daten, korrigiert aber Schlussfolgerungen, die durch die Messmethode nicht vollständig belegt sind.

| Thema                      | Konsolidierte Bewertung                                                                                                                                                                                                                                     | Fehlertyp bzw. Risiko                                                         |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Externe SSD                | Samsung 990 PRO im OWC StudioStack ist angeschlossen; 6,36 GB/s Read sind als Messwert dieses Laufs dokumentiert                                                                                                                                            | Messwert nicht automatisch gleich fairem internen A/B-Vergleich               |
| Interne SSD                | v2 behauptet, die externe SSD sei schneller; ein identischer, cache-kontrollierter interner Read-Test ist dort nicht dokumentiert                                                                                                                           | Quellen-/Benchmarküberdehnung                                                |
| Modellablage               | Externe SSD ist für residenten Hybrid sinnvoll, weil sie internen Platz spart und die gemessene Ladezeit akzeptabel ist                                                                                                                                    | keine RAM-Entlastung durch Ablage allein                                      |
| Q4-SSD-Streaming           | GGUF muss als Streamingquelle auf der SSD liegen;`--ssd-streaming` ist zusätzlich erforderlich                                                                                                                                                           | Verwechslung von Dateipfad und Runtime-Modus                                  |
| KV-Disk-Cache              | Persistente Snapshots können bei residentem Betrieb auf die externe SSD; bei Q4-Streaming zunächst intern testen                                                                                                                                          | I/O-Contention ist Hypothese bis A/B-Test                                     |
| Repository-Pfad            | Es gilt`/Users/michaelkuebbeler/ai-workspace/localAI/ds4`                                                                                                                                                                                                 | Neue, leerzeichenfreie Pfadannahme                                            |
| Download-Targets           | Der aktuelle Upstream-Downloader verwendet`ds4f-q2`, `ds4f-q2-q4` und `ds4f-q4` für die `-0731`-Artefakte                                                                                                                                          | Vor jedem Download den Target-Stand mit`./download_model.sh --help` prüfen |
| MTP/DSpark + SSD-Streaming | Der aktuelle Runtime-Code weist`--ssd-streaming` zusammen mit `--mtp` zurück.[6][8]                                                                                                                                                                    | Harte Kompatibilitätsverletzung                                              |
| MXFP4                      | Das HF-Artefakt existiert; der aktuelle Downloader/README bietet dafür kein Target. Issue#641 dokumentiert für einen geprüften Upstream-Stand den Loader-Fehler `unsupported GGUF type 39`; Runtime-Nutzbarkeit im verwendeten Commit verifizieren.[7] | Quellenfehlzuordnung zwischen Artefakt und Runtime                            |
| `iogpu.wired_limit_mb`   | Der reale Bestand`118000` wird dokumentiert, aber nicht erhöht; Kontext und tatsächliche Memory Pressure sind die primären Stellgrößen                                                                                                               | Übersehene Variable / unnötiger Systemeingriff                              |
| Rollback                   | Kein pauschales`pkill -f omlx`, kein blindes `rm -rf`; Prozesse und Pfade vorher identifizieren                                                                                                                                                         | Kollateralschaden                                                             |

**Konfidenz:** Die lokalen Mount-, Kapazitäts- und SSD-Messwerte sind für den dokumentierten Lauf gut belegt. Die Behauptung „externe SSD schneller als interne SSD“ ist bis zum identischen A/B-Test nur plausibel. Die Upstream-Aussagen zu Targets, Flags, Build-Artefakten und harten Inkompatibilitäten sind gut belegt am gepinnten Commit.

---

## 3. Modell- und Variantenentscheidung

### 3.1 GB, GiB und `hw.memsize` nicht vermischen

Die Apple-Konfigurationsbezeichnung lautet **128 GB**. Eine rein dezimale Umrechnung von 128.000.000.000 Bytes ergibt 119,209 GiB. Das ist jedoch nicht zwingend der Wert, den macOS für den tatsächlich verbauten Unified Memory meldet: Bei einem 128-GB-Apple-Silicon-System kann `hw.memsize` beispielsweise **137.438.953.472 Bytes = 128 GiB** ausweisen.

Deshalb gilt für dieses Zielsystem: Die Hardwarekapazität wird live ermittelt; weder 119,2 GiB noch 128 GiB dürfen als frei verfügbares Modellbudget interpretiert werden.

```bash
sysctl hw.memsize | awk '{printf "hw.memsize: %d bytes = %.3f GiB\\n", $2, $2/1024^3}'
```

Für die resident laufende Inferenz sind zusätzlich macOS, WindowServer, Metal, Graph-Scratch, Aktivierungen, KV-Zustand und andere Prozesse abzuziehen. Die Modell-Dateigröße ist deshalb nur ein notwendiger, nicht hinreichender Fit-Test. Fixe Abzüge wie „8 GiB für macOS“ sind Planungsheuristiken, keine Garantie.

### 3.2 Verifizierte Hauptartefakte

Die folgenden Größen sind aus den aktuellen HF-LFS-Dateigrößen in GiB umgerechnet. Vor einem Download muss der lokale Downloader-Stand nochmals gegen `./download_model.sh --help` geprüft werden.[2][4]

| Variante                    | Offizieller Dateiname                                                                                                                  |      Größe ungefähr | Resident auf 128 GB      | Entscheidung                                                         |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ---------------------: | ------------------------ | -------------------------------------------------------------------- |
| Q2 imatrix                  | `DeepSeek-V4-Flash-IQ2XXS-w2Q2K-AProjQ8-SExpQ8-OutQ8-chat-v2-imatrix.gguf`                                                           |   86,72 GB / 80,76 GiB | Ja, mit Reserve          | Referenz, nur wenn das unversionierte Artefakt bewusst gewählt wird |
| **Q2 imatrix 0731**   | `DeepSeek-V4-Flash-IQ2XXS-w2Q2K-AProjQ8-SExpQ8-OutQ8-chat-v2-imatrix-0731.gguf`                                                      |   86,72 GB / 80,76 GiB | Ja, mit Reserve          | **0731-Baseline nach explizitem Download**                     |
| Q2/Q4-Hybrid                | `DeepSeek-V4-Flash-Layers37-42Q4KExperts-OtherExpertLayersIQ2XXSGateUp-Q2KDown-AProjQ8-SExpQ8-OutQ8-chat-v2-imatrix-fixed.gguf`      |   97,59 GB / 90,89 GiB | Wahrscheinlich, aber eng | Referenz ohne 0731-Suffix                                            |
| **Q2/Q4-Hybrid 0731** | `DeepSeek-V4-Flash-Layers37-42Q4KExperts-OtherExpertLayersIQ2XXSGateUp-Q2KDown-AProjQ8-SExpQ8-OutQ8-chat-v2-imatrix-fixed-0731.gguf` |   97,59 GB / 90,89 GiB | Wahrscheinlich, aber eng | **bevorzugter 0731-Zielpfad**                                  |
| Volles Q4 imatrix           | `DeepSeek-V4-Flash-Q4KExperts-F16HC-F16Compressor-F16Indexer-Q8Attn-Q8Shared-Q8Out-chat-v2-imatrix.gguf`                             | 164,63 GB / 153,33 GiB | Nein                     | Nur SSD-Streaming                                                    |
| Volles Q4 imatrix 0731      | `DeepSeek-V4-Flash-Q4KExperts-F16HC-F16Compressor-F16Indexer-Q8Attn-Q8Shared-Q8Out-chat-v2-imatrix-0731.gguf`                        | 164,63 GB / 153,33 GiB | Nein                     | Nur SSD-Streaming                                                    |
| MXFP4 0731                  | `DeepSeek-V4-Flash-MXFP4Experts-F16HC-F16Compressor-F16Indexer-Q8Attn-Q8Shared-Q8Out-chat-v2-mxfp4-0731.gguf`                        | 155,98 GB / 145,26 GiB | Nein                     | Derzeit nicht verwenden                                              |
| DeepSeek V4 PRO Q2          | `DeepSeek-V4-Pro-IQ2XXS-w2Q2K-AProjQ8-SExpQ8-OutQ8-Instruct-imatrix.gguf`                                                            | 464,63 GB / 432,67 GiB | Nein                     | Nichtziel                                                            |

### 3.3 Entscheidung

**Stufe 1 – Q2 resident:** geringstes Risiko, schnellster belastbarer Referenzpunkt.
**Stufe 2 – Q2/Q4-Hybrid resident:** erwarteter bester Qualitäts-/Speicherkompromiss, aber mit kleinerer Reserve.
**Stufe 3 – Q4 mit SSD-Streaming:** Qualitätsvergleich und Technikexperiment, nicht Standardbetrieb.
**MXFP4:** nicht als Standardpfad einplanen. Das Artefakt existiert, aber der aktuelle Downloader/README bietet kein Target; Issue #641 dokumentiert für einen geprüften Upstream-Stand zusätzlich den Loader-Fehler `unsupported GGUF type 39`. Erst verwenden, wenn die Runtime-Unterstützung im tatsächlich gebauten Commit nachweislich vorhanden ist.[7]

### 3.4 Kosten-Nutzen-Analyse

| Option                                      | Nutzen                                                                              | Kosten/Risiko                                                                                                                 | Urteil                    |
| ------------------------------------------- | ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| Q2 resident                                 | geringstes Setup-Risiko, hohe Geschwindigkeit, reproduzierbare Baseline             | stärkere Quantisierung                                                                                                       | notwendig als Referenz    |
| Q2/Q4-Hybrid resident                       | bessere Qualitätschance bei weiterhin residentem Betrieb                           | ca. 10 GiB weniger Reserve, direkter M4-Max-Durchsatz noch nicht gemessen                                                     | sinnvoller Zielpfad       |
| Q4 SSD-Streaming                            | höherwertige Expertenquantisierung ohne 256-GB-Maschine                            | zusätzliche SSD-Abhängigkeit, Cache-Misses, niedrigerer Decode-Durchsatz                                                    | kontrolliertes Experiment |
| MXFP4                                       | theoretisch höchste Artefakt-Treue                                                 | kein aktuelles ds4-Downloader-Target; Loader-Nutzbarkeit im verwendeten Commit offen, Issue#641 berichtet einen Loader-Fehler | derzeit nicht einplanen   |
| `iogpu.wired_limit_mb` dauerhaft erhöhen | kann bei einem konkreten Residency-Problem zusätzlichen GPU-Budget-Spielraum geben | globaler Eingriff, weniger Systemreserve, schwerer Rollback                                                                   | nicht standardmäßig     |

---

## 4. Voraussetzungen und Status vor Anschluss der SSD

### 4.1 Bereits bzw. unabhängig von der SSD prüfbar

- Mac Studio M4 Max mit 128 GB Unified Memory.
- macOS mit funktionierendem Xcode Command Line Tools-Setup.
- `git`, `curl`, `make`.
- Lokaler Projektpfad `/Users/michaelkuebbeler/ai-workspace/localAI/ds4`.
- Ausreichend freier interner Speicher für Repository, Build und gegebenenfalls Q2.
- oMLX nicht gleichzeitig mit einem großen residenten ds4-Modell betreiben.

### 4.2 Live-Status der angeschlossenen SSD

Die lokale Statusprüfung vom **07.08.2026** meldete:

```text
hw.memsize: 137438953472
SSD: Samsung SSD 990 PRO 1TB im OWC StudioStack
Mount: /Volumes/AIModels
Filesystem: APFS
Device: /dev/disk4s1
TB-Link: 80 Gb/s
Swap: total = 0.00M, used = 0.00M
```

Die externe SSD ist damit für die weiteren Phasen verfügbar. Der im Master dokumentierte 8-GiB-Lesetest erreichte 6,36 GB/s dezimal bzw. 5,92 GiB/s nach `purge`. Dieser Wert ist ein beobachteter Einzeltest, keine vollständige Sustained- oder interne A/B-Benchmarkserie.

Vor produktivem Einsatz sind Mount-Punkt, freie Kapazität und Dateipfade zu verifizieren:

```bash
export DS4_DIR="/Users/michaelkuebbeler/ai-workspace/localAI/ds4"
export DS4_SSD="/Volumes/AIModels"

diskutil info "$DS4_SSD"
df -h "$DS4_SSD"
```

Die Messung belegt, dass die SSD schnell genug für die geplante Modellablage ist. Sie belegt noch nicht, dass sie unter jeder Last oder thermisch dauerhaft schneller als die interne SSD ist.

### 4.3 Speicher prüfen

```bash
export DS4_DIR="/Users/michaelkuebbeler/ai-workspace/localAI/ds4"
export DS4_SSD="/Volumes/AIModels"

df -h "$HOME"
df -h "$DS4_SSD"
du -sh "$DS4_DIR" 2>/dev/null || true
```

Für den residenten Hybrid zählt nicht nur die freie Dateisystemkapazität, sondern die aktuelle Unified-Memory-/GPU-Budgetierung. Für den Download zählt die freie Kapazität des Zielvolumes. Beide Größen getrennt prüfen; ein schneller SSD-Read ersetzt keine Memory-Reserve.

---

## 5. Phase A – Repository und Build ohne externe SSD

### 5.1 Vorhandenes Verzeichnis sicher als Arbeitsbaum verwenden

Der Ordner enthält bereits `Documents`. Deshalb nicht blind `git clone` in den bestehenden Ordner ausführen und keine vorhandenen Dateien löschen.

```bash
export DS4_DIR="/Users/michaelkuebbeler/ai-workspace/localAI/ds4"
export DS4_REPO="https://github.com/antirez/ds4.git"

cd "$DS4_DIR"

if [ ! -d .git ]; then
  git init
fi

if git remote get-url origin >/dev/null 2>&1; then
  git remote set-url origin "$DS4_REPO"
else
  git remote add origin "$DS4_REPO"
fi

git fetch origin main
git checkout -B main origin/main

git rev-parse --show-toplevel
git rev-parse HEAD
git status --short
```

Der aktuell geprüfte Upstream-Stand ist vom 07.08.2026: Commit `b0309611041655f4e45671cfd9c9886aff161406`. Für reproduzierbare Experimente Commit, Datum und Modell-Datei gemeinsam dokumentieren. Der Branch ist schnelllebig und laut README Beta-Software.[1][11]

### 5.2 Build-Voraussetzungen

```bash
xcode-select -p
cc --version
make --version
git --version
curl --version | head -1
```

Falls die Command Line Tools fehlen:

```bash
xcode-select --install
```

Homebrew und ein globales Python-Paket sind für den normalen Flash-Download nicht zwingend erforderlich. Der aktuelle Downloader verwendet für die kleineren Flash-Dateien den im Skript vorgesehenen Downloadpfad; die HF-CLI bleibt eine optionale Ausweichmöglichkeit.[2]

### 5.3 Metal-Build

```bash
cd "$DS4_DIR"
make clean
make
```

Der aktuelle macOS-`Makefile`-Pfad erzeugt:

```text
./ds4
./ds4-server
./ds4-bench
./ds4-eval
./ds4-agent
```

Prüfen:

```bash
cd "$DS4_DIR"
for f in ds4 ds4-server ds4-bench ds4-eval ds4-agent; do
  test -x "$f" && printf 'OK  %s\n' "$f" || printf 'FEHLT  %s\n' "$f"
done

./ds4 --help
./ds4-server --help
```

`make cpu` ist ein CPU-Diagnosebuild, nicht der gewünschte Metal-Produktionspfad. Nicht ausführen, wenn das Ziel die Metal-Inferenz ist. Der aktuelle Makefile-Stand dokumentiert `make` als macOS-Metal-Build.[3]

### 5.4 Build-Akzeptanzkriterium

Weitergehen erst, wenn:

- alle fünf erwarteten Binaries vorhanden und ausführbar sind;
- `./ds4 --help` und `./ds4-server --help` ohne Absturz laufen;
- der Commit dokumentiert ist;
- keine Modell- oder SSD-Annahme in den Build-Test eingeflossen ist.

---

## 6. Phase B – Build, Modellpfad und 0731-Baseline

### 6.1 Downloader-Targets immer zuerst prüfen

```bash
cd "$DS4_DIR"
./download_model.sh --help
```

Der aktuelle Downloader definiert unter anderem:

```text
ds4f-q2
ds4f-q2-q4
ds4f-q4
ds4f-dspark
```

Entscheidend: Im aktuellen `main`-Stand sind die Flash-Targets auf die `-0731`-Dateien gemappt. Ältere Versionen dieser Anleitung verwendeten die inzwischen veralteten Namen `q2-imatrix`, `q2-q4-imatrix` und `q4-imatrix`; sie sind keine aktuellen Copy-and-paste-Targets.[2][11]

Für den gewünschten 0731-Pfad genügt im aktuellen Stand das passende `ds4f-*`-Target. Der aktive Symlink muss trotzdem nach jedem Download geprüft werden.

### 6.2 Q2-Referenz laden – nur wenn bewusst gewünscht

```bash
cd "$DS4_DIR"
./download_model.sh ds4f-q2
```

Prüfen:

```bash
ls -lh "$DS4_DIR/gguf/"
ls -l "$DS4_DIR/ds4flash.gguf"
readlink "$DS4_DIR/ds4flash.gguf"
```

Der Symlink muss auf die erwartete Q2-imatrix-Datei zeigen. Nicht nur die Dateigröße, sondern den vollständigen Dateinamen und gegebenenfalls den HF-OID dokumentieren.

### 6.3 Aktiven 0731-Modellpfad prüfen

Der aktuelle Downloader verwendet für die Flash-Targets die `-0731`-Dateien. Der exakte Dateiname wird trotzdem geprüft, weil der Symlink `ds4flash.gguf` das tatsächlich aktive Modell bestimmt.

Dateiliste ohne Modell-Download prüfen:

```bash
curl -fsSL 'https://huggingface.co/api/models/antirez/deepseek-v4-gguf/tree/main?recursive=true&expand=true' \
  | python3 -c 'import json,sys; d=json.load(sys.stdin); print("\n".join(x["path"] for x in d if x["path"].endswith(".gguf")))'
```

Der bevorzugte 0731-Hybrid lautet:

```text
DeepSeek-V4-Flash-Layers37-42Q4KExperts-OtherExpertLayersIQ2XXSGateUp-Q2KDown-AProjQ8-SExpQ8-OutQ8-chat-v2-imatrix-fixed-0731.gguf
```

Der direkte Download über das aktuelle ds4-Skript ist der Standardpfad:

```bash
cd "$DS4_DIR"
./download_model.sh ds4f-q2-q4
```

Nach Installation einer HF-CLI in einer isolierten virtuellen Umgebung ist ein manueller Download nur als alternative, dateinamengebundene Kontrolle erforderlich:

```bash
python3 -m venv "$HOME/.venvs/huggingface"
source "$HOME/.venvs/huggingface/bin/activate"
python -m pip install -U huggingface_hub hf_xet

export DS4_0731_FILE="DeepSeek-V4-Flash-Layers37-42Q4KExperts-OtherExpertLayersIQ2XXSGateUp-Q2KDown-AProjQ8-SExpQ8-OutQ8-chat-v2-imatrix-fixed-0731.gguf"
hf download antirez/deepseek-v4-gguf "$DS4_0731_FILE" \
  --local-dir "$DS4_DIR/gguf"
```

Danach den aktiven Symlink bewusst auf diese Datei setzen:

```bash
cd "$DS4_DIR"
ln -sfn "gguf/$DS4_0731_FILE" ds4flash.gguf
ls -l ds4flash.gguf
readlink ds4flash.gguf
```

Der Symlink ist der entscheidende Aktivierungsschritt. Ein 0731-Artefakt im HF-Repository allein ändert nicht das aktuell verwendete Modell.

### 6.4 Modellinspektion vor Inferenz

```bash
cd "$DS4_DIR"
./ds4 -m ./ds4flash.gguf --inspect
```

Zu prüfen:

- DeepSeek-V4-Flash-Architektur erkannt;
- Metal als Backend aktiv;
- erwartete Modellgröße und Tensorlayout;
- keine `unsupported GGUF type`-Meldung;
- keine Speicher- oder Initialisierungsfehler.

### 6.5 Erster deterministischer Q2-Test

```bash
cd "$DS4_DIR"
./ds4 \
  -m ./ds4flash.gguf \
  --ctx 16384 \
  --nothink \
  --tokens 128 \
  --temp 0 \
  -p "Erkläre in fünf Stichpunkten, wie Redis Streams funktionieren."
```

Wenn dieser Test erfolgreich ist, Kontext vorsichtig erhöhen:

```bash
./ds4 \
  -m ./ds4flash.gguf \
  --ctx 32768 \
  --nothink \
  --tokens 256 \
  --temp 0 \
  -p "Vergleiche eventgetriebene und requestbasierte Architekturen."
```

Danach Thinking separat testen:

```bash
./ds4 \
  -m ./ds4flash.gguf \
  --ctx 32768 \
  --think \
  --tokens 512 \
  -p "Analysiere die Vor- und Nachteile einer eventgetriebenen Architektur."
```

`--think-max` zunächst nicht einsetzen. Der aktuelle CLI-Code verlangt für Think Max einen deutlich größeren Kontext; bei kleinerem Kontext wird auf normales Thinking zurückgefallen oder gewarnt.[8]

---

## 7. Speicher- und Stabilitätsüberwachung

In einem zweiten Terminal:

```bash
sysctl vm.swapusage
memory_pressure
vm_stat
ps axo pid,rss,comm -r | head -20
```

Für die laufende Beobachtung:

```bash
while true; do
  printf '\n--- %s ---\n' "$(date '+%d.%m.%Y %H:%M:%S')"
  sysctl vm.swapusage
  memory_pressure 2>/dev/null | tail -1
  sleep 5
done
```

### Stop-Kriterien

Lauf sofort abbrechen, wenn mindestens eines zutrifft:

- starke oder anhaltende Memory Pressure;
- Swap-Nutzung steigt während des Basistests deutlich;
- Prozess beendet sich wegen Speicherfehler;
- System reagiert erkennbar träge;
- Modell wird nicht als erwartete ds4-Variante erkannt.

### Reihenfolge der Gegenmaßnahmen

1. Anwendung abbrechen.
2. andere große Modelle und oMLX stoppen bzw. nicht parallel starten;
3. `--ctx` reduzieren;
4. `--nothink` und kurze Ausgabe verwenden;
5. auf Q2 zurückgehen;
6. erst danach die Modellvariante wechseln.

`iogpu.wired_limit_mb` nicht als ersten Reparaturschritt verändern. Der physische Unified Memory und die tatsächliche aktuelle Speicherbelegung sind die primären Variablen.

---

## 8. Phase C – SSD-Prüfung und Modellablage

Die externe SSD ist angeschlossen und für die weiteren Phasen verfügbar. Der dokumentierte 8-GiB-Read-Test ergab 6,36 GB/s dezimal bzw. 5,92 GiB/s nach `purge` bei einem 80-Gb/s-TB-Link. Das ist ein belastbarer Einzelmesswert für die Modellablage, aber noch kein fairer Nachweis eines dauerhaften Geschwindigkeitsvorteils gegenüber der internen SSD.

### 8.1 Mount-Punkt und Dateisystem verifizieren

```bash
export DS4_SSD="/Volumes/AIModels"

diskutil info "$DS4_SSD"
df -h "$DS4_SSD"
```

Der Mount-Punkt ist aktuell beobachtet, muss aber vor jedem Start geprüft werden. Device-Nummern wie `/dev/disk4s1` sind nicht stabil und dürfen nicht fest in Skripte geschrieben werden.

### 8.2 Spotlight und Time Machine bewusst konfigurieren

```bash
mdutil -s "$DS4_SSD"
tmutil isexcluded "$DS4_SSD"
```

Falls bewusst gewünscht und freigegeben:

```bash
sudo mdutil -i off "$DS4_SSD"
sudo mdutil -E "$DS4_SSD"
sudo tmutil addexclusion "$DS4_SSD"
```

Diese Ausschlüsse sind Betriebsoptimierungen, keine Voraussetzung für ds4. Die SSD ist nicht ungeprüft zu formatieren; `diskutil eraseDisk` bleibt außerhalb des Standardpfads.

### 8.3 Optionaler A/B-Test gegen die interne SSD

Die Aussage „externe SSD schneller als interne SSD“ ist erst nach einem identischen Test auf beiden Volumes belastbar. Mindestens festhalten:

- gleiche Dateigröße und Blockgröße;
- gleicher Read-Test und gleiche Anzahl Wiederholungen;
- Cache-Zustand und Zeitpunkt von `purge`;
- Temperatur-/Dauerlastzustand;
- keine parallelen Modell-, Backup- oder Indexierungszugriffe;
- Mittelwert und Streuung statt eines Einzelwerts.

Der vorhandene 6,36-GB/s-Wert bleibt als **externer Einzelmesswert** dokumentiert. Er wird nicht als allgemeiner interner-vs.-externer Sieger ausgegeben.

### 8.4 Modellverzeichnis auf die SSD verlagern

```bash
export DS4_DIR="/Users/michaelkuebbeler/ai-workspace/localAI/ds4"
mkdir -p "$DS4_SSD/ds4/gguf" "$DS4_SSD/ds4/kv"

cd "$DS4_DIR"
if [ -e gguf ] && [ ! -L gguf ]; then
  mv gguf "gguf.before-ssd.$(date +%Y%m%d-%H%M%S)"
fi
ln -s "$DS4_SSD/ds4/gguf" gguf
ls -ld gguf
ls -la gguf/
df -h "$DS4_SSD"
```

Kein blindes `rm -rf gguf` verwenden. Das Verschieben der GGUF-Datei auf die SSD reduziert den RAM-Bedarf im residenten Modus nicht; für Q4-Streaming ist zusätzlich `--ssd-streaming` erforderlich. Der Symlink verlagert den Download-/Modellpfad, nicht den aktiven Unified-Memory-Zustand.

---

## 9. Phase D – Modell auf die SSD verlagern

Die Modellablage wird im Verzeichnis `/Volumes/AIModels/ds4/gguf` geführt. Der Projektpfad enthält jetzt kein Leerzeichen mehr:

```bash
export DS4_DIR="/Users/michaelkuebbeler/ai-workspace/localAI/ds4"
export DS4_SSD="/Volumes/AIModels"

mkdir -p "$DS4_SSD/ds4/gguf"
```

Wenn `gguf` noch ein normales Verzeichnis ist und keine wichtigen Dateien enthält:

```bash
cd "$DS4_DIR"

if [ -e gguf ] && [ ! -L gguf ]; then
  mv gguf "gguf.before-ssd.$(date +%Y%m%d-%H%M%S)"
fi

ln -s "$DS4_SSD/ds4/gguf" gguf
ls -ld gguf
```

Kein blindes `rm -rf gguf` verwenden. Nach dem Symlink:

```bash
cd "$DS4_DIR"
ls -la gguf/
df -h "$DS4_SSD"
```

Danach können weitere Modelle mit `./download_model.sh` in das verlinkte Verzeichnis geladen werden. Das alte interne Modellverzeichnis erst löschen, wenn der neue Pfad erfolgreich geprüft wurde.

### 9.5 Entscheidung: Modellgewichte oder KV-Disk-Cache auf die SSD?

Die beiden Verlagerungen lösen **unterschiedliche Probleme** und dürfen nicht gleichgesetzt werden:

| Komponente                     | Was liegt auf der SSD?                                                                                     |                                                                                                  Entlastet Unified Memory? | Bedeutung für SSD-Expert-Streaming                                                      |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------: | ---------------------------------------------------------------------------------------- |
| **GGUF-Modellgewichte**  | eigentliche Modell-Datei; bei`--ssd-streaming` werden geroutete Experten daraus bei Cache-Misses gelesen | **Ja, aber nur zusammen mit `--ssd-streaming`**: bloßes Verschieben der GGUF-Datei reduziert den RAM-Bedarf nicht | Für Q4/Modelle größer als RAM als Streamingquelle erforderlich                        |
| **KV-Disk-Cache**        | persistente Prefix-/Session-Snapshots                                                                      |                     **Nein, nicht grundsätzlich**: der aktive KV-Zustand bleibt für die Inferenz im Unified Memory | Neustart-/Session-Persistenz und Prefill-Optimierung; kein Ersatz für Gewichtsstreaming |
| **Token-/SSE-Streaming** | keine Modell- oder KV-Datei                                                                                |                                                                                                                       Nein | Nur Transport der Ausgabe; löst kein Speicherproblem                                    |

**Kernentscheidung:** Für Q4-Expert-Streaming liegt die GGUF-Modellquelle auf der externen SSD und `--ssd-streaming` muss aktiviert werden. Der Symlink von `"$DS4_DIR/gguf"` auf `"$DS4_SSD/ds4/gguf"` verlagert Download- und Modellpfad, nicht den aktiven Unified-Memory-Zustand. Nicht-geroutete Gewichte, Expert-Cache, Graph/Scratch, Aktivierungen und aktiver KV bleiben im Speicher.[1][8]

Der KV-Disk-Cache ist separat zu bewerten. `--kv-disk-dir` verlagert den aktiven KV-Zustand nicht aus dem RAM; ds4 schreibt und lädt persistente Snapshots. Der Server verwendet dafür gewöhnliche `read`/`write`-I/O und vermeidet beim Restore zusätzliche Modell-VM-Mappings.[1][9]

### 9.6 Storage-Entscheidung für die gemessene SSD

| Betriebsmodus                    | Modell-GGUF                                                                                                 | aktiver KV / KV-Snapshots                                         | Bewertung                                                                           |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Q2/Hybrid**resident**      | **externe SSD empfohlen**: 6,36 GB/s Read ist für die Modellablage ausreichend; spart internen Platz | **externe SSD empfohlen**                                   | Nach dem Residency-Build kaum Gewichts-I/O; die SSD kann den KV-Disk-Cache bedienen |
| Q4**SSD-Streaming**        | **externe SSD erforderlich**                                                                          | aktiver KV im Unified Memory; Snapshots zunächst**intern** | Gewicht-Reads haben Priorität; SSD-KV zunächst als separater A/B-Test             |
| Q4 SSD-Streaming mit zweiter SSD | SSD 1 für GGUF                                                                                             | SSD 2 für KV-Snapshots                                           | sauberste I/O-Trennung                                                              |

Die v2-Messung spricht für die externe SSD als Modellablage im residenten Standardpfad: 6,36 GB/s bedeuten rechnerisch ca. 15,4 s reine I/O für 90,9 GiB Hybrid-Gewichte. Das ist eine Ladezeitbetrachtung, keine RAM-Entlastung und keine bestätigte allgemeine Überlegenheit gegenüber der internen SSD.

Für die einzelne externe SSD ist die empfohlene Reihenfolge:

1. 0731-GGUF auf der externen SSD ablegen und per `gguf`-Symlink aktivieren.
2. Residenten Hybrid ohne laufende Gewichts-I/O betreiben.
3. Im residenten Betrieb KV-Snapshots auf derselben SSD aktivieren.
4. Für Q4-Streaming KV-Snapshots zunächst intern ablegen.
5. Externen KV-Disk-Cache im Q4-Streaming anschließend als A/B-Test gegen den internen KV-Pfad messen.

Die Contention-Annahme im Q4-Pfad ist eine konservative Hypothese, bis ein realer A/B-Test mit identischem Prompt-/Session-Szenario, gleichem Cachebudget und gleicher thermischer Situation vorliegt.

### 9.7 Modellverlagerung und Verlinkung

Wenn `gguf` noch ein normales Verzeichnis ist und keine wichtigen Dateien enthält:

```bash
cd "$DS4_DIR"
mkdir -p "$DS4_SSD/ds4/gguf"

if [ -e gguf ] && [ ! -L gguf ]; then
  mv gguf "gguf.before-ssd.$(date +%Y%m%d-%H%M%S)"
fi

ln -s "$DS4_SSD/ds4/gguf" gguf
ls -ld gguf
```

Danach zeigt der Modellpfad für den Downloader und für `ds4flash.gguf` auf die SSD. Vor dem Löschen des gesicherten internen Verzeichnisses muss mindestens ein `ls`, ein `df` und ein erfolgreicher Modellpfad-Check durchgeführt werden.

Die bestehende allgemeine Zielaufteilung bleibt damit präzisiert:

| Betriebsmodus      | Modellgewichte                                                          | KV-Disk-Cache                                                  |
| ------------------ | ----------------------------------------------------------------------- | -------------------------------------------------------------- |
| Resident Q2/Hybrid | **externe SSD empfohlen**; nach dem Laden kaum laufende SSD-Reads | **extern empfohlen**; intern als A/B-Vergleich           |
| Q4 SSD-Streaming   | **SSD erforderlich**                                              | **zunächst intern empfohlen**; extern nur nach I/O-Test |

Die aktuelle ds4-Dokumentation beschreibt SSD-Streaming als residenten Nicht-Routed-Anteil plus SSD-Laden gerouteter Experten bei Cache-Misses; sie beschreibt den Disk-KV-Cache separat als persistente Session-/Prefix-Snapshots.[1]

---

## 10. Hauptszenarien: resident, automatisch gestreamt und Budget-Streaming

Die drei Profile verwenden dieselbe ds4-Binary, dieselbe `-0731`-Q2/Q4-Hybrid-GGUF und denselben externen Modellpfad. Nur die Runtime-Optionen und der KV-Disk-Cachepfad werden geändert.

### Szenario 1 – Q2/Q4-Hybrid vollständig resident

```text
GGUF-Modell:        /Volumes/AIModels/ds4/gguf
Modellgewichte:     vollständig im Unified Memory
Aktiver KV-Cache:   Unified Memory
KV-Disk-Snapshots:  /Volumes/AIModels/ds4/kv
SSD-Streaming:      aus
```

```bash
export DS4_DIR="/Users/michaelkuebbeler/ai-workspace/localAI/ds4"
export DS4_SSD="/Volumes/AIModels"
mkdir -p "$DS4_SSD/ds4/kv"

cd "$DS4_DIR"
./ds4-server \
  -m ./ds4flash.gguf \
  --host 127.0.0.1 \
  --port 8000 \
  --ctx 32768 \
  --kv-disk-dir "$DS4_SSD/ds4/kv" \
  --kv-disk-space-mb 8192 \
  --chdir "$DS4_DIR"
```

`--kv-disk-dir` speichert persistente Prefix-/Session-Snapshots. Der aktive KV-Zustand bleibt im Unified Memory.

### Szenario 2 – Q2/Q4-Hybrid mit automatischer SSD-Budgetierung

```text
GGUF-Modell:        externe SSD
Nicht-geroutete Gewichte: resident
Geroutete Experten:        dynamischer RAM-Cache plus SSD-Nachladen
Aktiver KV-Cache:          Unified Memory
KV-Disk-Snapshots:         zunächst interne SSD
```

```bash
export DS4_DIR="/Users/michaelkuebbeler/ai-workspace/localAI/ds4"
export DS4_SSD="/Volumes/AIModels"
mkdir -p "$HOME/.ds4/server-kv"

cd "$DS4_DIR"
./ds4-server \
  -m ./ds4flash.gguf \
  --ssd-streaming \
  --host 127.0.0.1 \
  --port 8000 \
  --ctx 16384 \
  --kv-disk-dir "$HOME/.ds4/server-kv" \
  --kv-disk-space-mb 8192 \
  --chdir "$DS4_DIR"
```

Die automatische ds4-Budgetierung berechnet den residenten Anteil, Bytes pro Expert, Expert-Cache und Prefill-Reserve aus dem konkreten GGUF und dem Metal-Working-Set. Sie ist der erste Streamingtest.

### Szenario 3 – Q2/Q4-Hybrid mit explizitem Expert-Cache-Budget

```bash
cd "/Users/michaelkuebbeler/ai-workspace/localAI/ds4"

./ds4-server \
  -m ./ds4flash.gguf \
  --ssd-streaming \
  --ssd-streaming-cache-experts 48GB \
  --host 127.0.0.1 \
  --port 8000 \
  --ctx 16384 \
  --kv-disk-dir "$HOME/.ds4/server-kv" \
  --kv-disk-space-mb 8192 \
  --chdir "/Users/michaelkuebbeler/ai-workspace/localAI/ds4"
```

`48GB` wird als ungefähr 48 GiB verarbeitet und bezeichnet nur das Budget für den gerouteten Expert-Cache. Es ist kein Gesamtlimit für den ds4-Prozess. Zusätzlich benötigen Unified Memory:

```text
R + E + H + B + K(ctx)
```

mit:

- `R`: residenter Modellanteil, insbesondere nicht-geroutete Gewichte;
- `E`: dynamischer Expert-Cache;
- `H`: Prefill-Reserve, einschließlich der vom Code berücksichtigten vollständigen Streaming-Layer;
- `B`: Graph-Scratch, Aktivierungen, Prefill-Buffer und Runtime-Overhead;
- `K(ctx)`: aktiver KV-/Kontextzustand.

Ein allgemeines `--memory-limit`-Flag gibt es nicht. `--kv-disk-dir` ersetzt den aktiven KV-Cache nicht. Für ein Ziel von höchstens 70 GiB ohne aktiven KV ist grob zu prüfen:

```text
R + E + H + B ≤ 70 GiB
```

Die Größe `R` lässt sich nicht zuverlässig aus der GGUF-Gesamtgröße ableiten. Sie muss aus dem normalen Streaming-Startlog und dem konkreten Cacheplan übernommen werden. Für 100–110 GiB inklusive aktivem KV gilt:

```text
R + E + H + B + K(ctx) ≤ 100–110 GiB
```

Kontext und Expert-Cache deshalb schrittweise testen:

```text
32K → 64K → 128K → 256K → 500K
```

und beim expliziten Budget:

```text
32GB → 40GB → 48GB
```

### Gemeinsame Installation und Betriebsmodus-Schalter

Es ist keine zweite Modellinstallation erforderlich. Die Varianten unterscheiden sich nur durch die Runtime-Flags.

```bash
# Szenario 1: resident
DS4_KV_DIR="/Volumes/AIModels/ds4/kv" \
  "/Users/michaelkuebbeler/ai-workspace/localAI/ds4/DwarfStar4-Mode-Switch.command" ds4

# Szenario 2: automatische Budgetierung
DS4_STREAMING=1 DS4_CACHE_EXPERTS="" \
DS4_KV_DIR="$HOME/.ds4/server-kv" \
  "/Users/michaelkuebbeler/ai-workspace/localAI/ds4/DwarfStar4-Mode-Switch.command" ds4

# Szenario 3: explizites 48-GiB-Expert-Cache-Budget
DS4_KV_DIR="$HOME/.ds4/server-kv" \
  "/Users/michaelkuebbeler/ai-workspace/localAI/ds4/DwarfStar4-Mode-Switch.command" ds4-stream48
```

Im interaktiven Menü entspricht Option 2 dem residenten Profil und Option 3 dem expliziten 48-GB-Streamingprofil.

---

## 11. Phase E – `-0731`-Q2/Q4-Hybrid resident

Nach erfolgreichem Build und Q2-Basistest wird das aktuelle Target `ds4f-q2-q4` für den 0731-Hybrid verwendet. Abschnitt 6.3 prüft anschließend den exakten Dateinamen und den aktiven Symlink.

Prüfen:

```bash
cd "$DS4_DIR"
ls -lh "$DS4_DIR/gguf/"
ls -l "$DS4_DIR/ds4flash.gguf"
readlink "$DS4_DIR/ds4flash.gguf"
```

Erster residenter Test mit 32K:

```bash
./ds4 \
  -m ./ds4flash.gguf \
  --ctx 32768 \
  --nothink \
  --tokens 128 \
  --temp 0 \
  -p "Führe einen kurzen technischen Funktionstest durch."
```

Nach erfolgreichem Test den empfohlenen Server-/Bewertungskontext von 131072 schrittweise prüfen:

```bash
./ds4 \
  -m ./ds4flash.gguf \
  --ctx 131072 \
  --nothink \
  --tokens 256 \
  --temp 0 \
  -p "Vergleiche eventgetriebene und requestbasierte Architekturen."
```

Die Zielrechnung aus v2 verwendet 90,9 GiB Hybrid-Gewichte, 6,0 GiB Graph/Aktivierungen, den beobachteten 118000-MiB-Wired-Rahmen und einen modellabhängigen KV-Bedarf. Die resultierende Reserve ist eine Planungsschätzung; der reale Startlog, Memory Pressure und Swap sind maßgeblich. Der `-0731`-Hybrid ist als Zielmodell plausibel, der direkte Durchsatz auf Michaels Mac Studio bleibt empirisch zu messen.[5]
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## 12. Phase F – Q4 ausschließlich mit SSD-Streaming

### 12.1 Vorbedingungen

- SSD angeschlossen und Mount-Punkt verifiziert.
- Q2 oder Hybrid stabil getestet.
- mindestens 200 GiB freie SSD-Kapazität für Modell, temporäre Dateien und KV-Cache.
- keine parallele große oMLX-Instanz.
- SSD-Streaming als Experiment akzeptiert.

### 12.2 Q4 laden

```bash
cd "$DS4_DIR"
./download_model.sh ds4f-q4
```

Der aktuelle Downloader lädt damit die aktuelle Q4-imatrix-0731-Datei. Der Q4-0731-Pfad bleibt wegen der Modellgröße ein reines SSD-Streaming-Experiment.[2][4]

### 12.3 Automatisches SSD-Streaming zuerst

```bash
cd "$DS4_DIR"
./ds4 \
  -m ./ds4flash.gguf \
  --ssd-streaming \
  --ctx 16384 \
  --nothink \
  --tokens 128 \
  --temp 0 \
  -p "Führe einen kurzen Stabilitätstest des Q4-SSD-Streaming-Betriebs durch."
```

Nicht sofort mit einem manuell optimierten Cache starten. Der aktuelle ds4-Code kann explizite Cachebudgets nach Kontext- und Working-Set-Accounting begrenzen; der Startlog ist maßgeblich.[1][8]

### 12.4 Manuelles Expert-Cache-Budget erst nach erfolgreichem Start

```bash
./ds4 \
  -m ./ds4flash.gguf \
  --ssd-streaming \
  --ssd-streaming-cache-experts 32GB \
  --ctx 16384 \
  --nothink \
  --tokens 128 \
  --temp 0 \
  -p "Teste einen begrenzten Q4-Expert-Cache."
```

`32GB` ist ein Budget für geroutete Experten, kein Gesamtbudget des Prozesses. Ein reiner Zahlenwert wie `4000` bezeichnet eine Zahl von Expertenslots und ist semantisch etwas anderes.[1]

### 12.5 SSD-Streaming beobachten

Erst nach Ermittlung des tatsächlichen Devices:

```bash
iostat -w 1
```

Ein niedriger momentaner Durchsatz beweist nicht automatisch, dass Streaming deaktiviert ist: Page-Cache-Hits können physische I/O temporär reduzieren. Maßgeblich sind Startlog, Modellpfad, Cacheplan und reproduzierbare Laufzeitmessung.[1]

### 12.6 Harte Ausschlussregel für MTP/DSpark

Nicht verwenden:

```bash
./ds4 \\
  -m ./ds4flash.gguf \\
  --ssd-streaming \\
  --mtp ./gguf/DeepSeek-V4-Flash-DSpark-support-0731.gguf \\
  --dspark
```

Der aktuelle Runtime-Code verweigert `--ssd-streaming` zusammen mit `--mtp`; Issue #596 dokumentiert denselben Fehlertext.[6][8]

---

## 13. Optional: MTP oder DSpark resident

Diese Komponenten sind Support-/Speculative-Decoding-Artefakte, keine eigenständigen Chatmodelle.

### 13.1 Legacy-MTP

```bash
cd "$DS4_DIR"
./download_model.sh mtp

./ds4 \
  -m ./ds4flash.gguf \
  --mtp ./gguf/DeepSeek-V4-Flash-MTP-Q4K-Q8_0-F32.gguf \
  --mtp-draft 2 \
  --ctx 32768 \
  --temp 0 \
  --tokens 256 \
  -p "Teste die optionale MTP-Unterstützung."
```

Der aktuelle Downloader hält das Legacy-MTP-Ziel nicht in der neuen `ds4f-*`-Namensfamilie; für DeepSeek Flash 0731 ist stattdessen der DSpark-Target `ds4f-dspark` vorgesehen. Legacy-MTP nur verwenden, wenn `./download_model.sh --help` das Target im konkret ausgecheckten Commit weiterhin anbietet; der aktuelle 0731-Standard ist `ds4f-dspark`. Die HF-Datei ist rund 3,81 GB groß.[2][4]

### 13.2 DSpark

Der aktuelle Downloader-Target heißt **`ds4f-dspark`**. Die alten Bezeichnungen `dspark` und `dspark-support` sind keine aktuellen Targets.

```bash
cd "$DS4_DIR"
./download_model.sh ds4f-dspark

./ds4 \
  -m ./ds4flash.gguf \
  --mtp ./gguf/DeepSeek-V4-Flash-DSpark-support-0731.gguf \
  --dspark \
  --ctx 32768 \
  --temp 0 \
  --tokens 256 \
  -p "Teste DSpark im residenten Flash-Betrieb."
```

DSpark benötigt Greedy-Decoding bzw. `--temp 0`, weil gesampelte Decodierung die DSpark-Vorschläge nicht nutzt. Das Support-GGUF ist laut aktuellem HF-Eintrag rund 5,99 GB groß.[1][2][4]

### 13.3 Reihenfolge

```text
Q2 resident
→ Q2/Q4-Hybrid resident
→ optional Q2 + MTP oder DSpark
→ Q4 mit SSD-Streaming
```

Nicht:

```text
Q4 + SSD-Streaming + MTP/DSpark
```

---

## 14. Lokaler Server

Erst nach erfolgreichem CLI-Test starten. Der aktuelle Server hat standardmäßig `127.0.0.1`, Port `8000`, Modell `ds4flash.gguf` und Kontext `32768`; explizite Angaben bleiben für reproduzierbare Betriebsprofile empfehlenswert.[1][9]

### 14.1 Q2 oder Hybrid resident

Im residenten Betrieb wird die externe SSD als Modellablage und für den persistenten KV-Disk-Cache verwendet. Die Modellgewichte werden nach dem Residency-Build nicht fortlaufend von der SSD gelesen; dadurch ist die gemeinsame Ablage in diesem Betriebsmodus zunächst plausibel.

```bash
cd "$DS4_DIR"
mkdir -p "$DS4_SSD/ds4/kv"

./ds4-server \
  --model ./ds4flash.gguf \
  --host 127.0.0.1 \
  --port 8000 \
  --ctx 32768 \
  --kv-disk-dir "$DS4_SSD/ds4/kv" \
  --kv-disk-space-mb 8192 \
  --chdir "$DS4_DIR"
```

Für einen isolierten Vergleich kann `--kv-disk-dir "$HOME/.ds4/server-kv"` auf dem internen Laufwerk verwendet werden.

### 14.2 API prüfen

In einem zweiten Terminal:

```bash
curl -i http://127.0.0.1:8000/v1/models
```

Minimaler Chat-Test:

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model":"deepseek-v4-flash",
    "messages":[{"role":"user","content":"Antworte mit genau einem Wort: Test"}],
    "temperature":0,
    "max_tokens":32
  }'
```

SSE-/Token-Streaming testen:

```bash
curl -N http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model":"deepseek-v4-flash",
    "messages":[{"role":"user","content":"Erkläre den Unterschied zwischen Q2 und Q4."}],
    "temperature":0,
    "max_tokens":128,
    "stream":true
  }'
```

`stream: true` ist Token-/SSE-Streaming. Es ist nicht dasselbe wie `--ssd-streaming` und verändert nicht den Speicherbedarf der Modellgewichte.

### 14.3 KV-Disk-Cache

Der KV-Disk-Cache persistiert Prefix-/Session-Zustände über Session-Wechsel und Neustarts. Er ist vom Expert-Streaming unabhängig, aber der physische Speicherort ist betriebsmodusabhängig:

- residenter Hybrid: externe SSD zunächst empfohlen;
- Q4-SSD-Streaming: interne Ablage zunächst empfohlen;
- danach A/B-Test extern gegen intern.

Für den residenten Hybrid läuft der Server mit der externen SSD als KV-Ziel:

```bash
mkdir -p "$DS4_SSD/ds4/kv"

./ds4-server \
  --model ./ds4flash.gguf \
  --ctx 32768 \
  --kv-disk-dir "$DS4_SSD/ds4/kv" \
  --kv-disk-space-mb 8192 \
  --chdir "$DS4_DIR"
```

Für Q4-Streaming zunächst stattdessen:

```bash
mkdir -p "$HOME/.ds4/server-kv"
./ds4-server \
  --model ./ds4flash.gguf \
  --ssd-streaming \
  --ctx 16384 \
  --kv-disk-dir "$HOME/.ds4/server-kv" \
  --kv-disk-space-mb 8192 \
  --chdir "$DS4_DIR"
```

Erst nach einer separaten A/B-Messung darf der Q4-KV-Pfad auf `"$DS4_SSD/ds4/kv"` gelegt werden. Die KV-Dateien enthalten rekonstruierbare Prompt-/Sessiondaten und sind deshalb datenschutzrelevant. Verzeichniszugriff begrenzen und Cache bei Bedarf löschen:

```bash
rm -rf "$HOME/.ds4/server-kv"/*
```

### 14.4 Mehrere Sessions

Der aktuelle Server unterstützt `--batched-session N`. Dies ist kein Freibrief für parallele Last: alle KV-Sessions müssen zusätzlich in den Speicher passen. Erst nach einer Einzel-Session-Baseline und einer Speicherrechnung aktivieren.[1]

---

## 15. LiteLLM-Integration erst nach Direktprüfung

Direkt zuerst prüfen:

```bash
curl -i http://127.0.0.1:8000/v1/models
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"Test"}],"max_tokens":16}'
```

Erst danach in LiteLLM ergänzen:

```yaml
model_list:
  - model_name: deepseek-v4-flash-local
    litellm_params:
      model: openai/deepseek-v4-flash
      api_base: http://127.0.0.1:8000/v1
      api_key: dsv4-local
      timeout: 600
      stream_timeout: 600
```

Danach über LiteLLM testen:

```bash
curl -s http://127.0.0.1:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"deepseek-v4-flash-local","messages":[{"role":"user","content":"Antworte mit genau einem Wort: Test"}],"max_tokens":16}'
```

Den Namen `deepseek-v4-flash-local` nur verwenden, wenn er tatsächlich in der laufenden LiteLLM-Konfiguration eingetragen ist. Keine Platzhalter wie `$LITEL..._KEY` in ein ausführbares Kommando übernehmen.

### Anthropic-/Claude-Code-Pfad

Der ds4-Server dokumentiert einen Anthropic-kompatiblen Endpoint. Die Client-Integration wird erst nach erfolgreichem `/v1/messages`-Test eingerichtet. Für einen lokalen Wrapper sind mindestens Base-URL, lokales Auth-Token, Modellname und lange Idle-Timeouts zu setzen. Client-spezifische Umgebungsvariablen vor Verwendung gegen die installierte Claude-Code-Version prüfen; sie sind keine ds4-Upstream-Schnittstelle.

---

## 16. Performance- und Qualitätsmessung

### 16.1 Status der Messungen

Auf dem Zielsystem wurde ein nativer SSD-Lesetest durchgeführt: 8 GiB, `dd`, Read nach `sudo purge`, 6,36 GB/s dezimal bzw. 5,92 GiB/s. Das ist ein Ist-Messwert dieses Laufs. Ein identischer interner A/B-Test, mehrere Wiederholungen, eine längere thermische Sustained-Messung und ein echter Q4-SSD-Streaming-Lauf sind in v2 jedoch nicht dokumentiert.

Noch nicht als Zielsystem-Messung belegt sind:

- dauerhafte Sustained-Read-Leistung über längere Zeit;
- direkter Vergleich mit der internen Apple-SSD unter identischen Bedingungen;
- Q4-SSD-Streaming auf diesem Mac Studio;
- Q2/Q4-Hybrid-Durchsatz auf diesem Mac Studio;
- tatsächlicher Einfluss des KV-Disk-Caches im vorhandenen Agenten-Stack.

### 16.2 Reproduzierbarer Storage-A/B-Test

Die Aussage „externe SSD schneller als interne SSD“ darf erst nach einem identischen A/B-Test als Ergebnis übernommen werden. Beide Tests müssen dieselbe Dateigröße, Blockgröße, Wiederholungszahl und Cache-/Thermikprozedur verwenden. Ein möglicher, nicht automatisch auszuführender Testaufbau:

```bash
export DS4_SSD="/Volumes/AIModels"
export EXT_TEST="$DS4_SSD/ds4-read-test.bin"
export INT_TEST="$HOME/ds4-read-test.bin"

# Einmalige Testdateien; ausreichend freien Speicher vorher prüfen.
dd if=/dev/zero of="$EXT_TEST" bs=1m count=8192

dd if=/dev/zero of="$INT_TEST" bs=1m count=8192

sync
sudo purge
/usr/bin/time -p dd if="$EXT_TEST" of=/dev/null bs=1m count=8192

sudo purge
/usr/bin/time -p dd if="$INT_TEST" of=/dev/null bs=1m count=8192

rm -f "$EXT_TEST" "$INT_TEST"
```

Für eine belastbare Entscheidung mindestens drei Wiederholungen pro Medium, gleiche Reihenfolge in Gegenproben, Temperaturbeobachtung und einen längeren Lauf ergänzen. `purge` ist keine perfekte Cache-Isolation; deshalb sind die Ergebnisse als Systemmessung mit dokumentiertem Cache-Zustand zu verstehen, nicht als physikalisch reiner NAND-Benchmark.

### 16.3 Historische Referenzwerte

Issue #437 berichtet auf einem M4 Max mit 128 GiB unter definierten Bedingungen unter anderem q2 resident mit 22,80 tok/s sowie q4 SSD-Streaming mit Cachebudget b80 und 11,00 tok/s.[5] Diese Zahlen sind eine historische Referenzmessung einer anderen Umgebung bzw. eines anderen Upstream-Zustands. Sie dürfen nicht als Messung des Zielsystems ausgegeben werden.

### 16.4 Reproduzierbarer lokaler Benchmark

Nach erfolgreichem Basistest:

```bash
cd "$DS4_DIR"
./ds4-bench \
  -m ./ds4flash.gguf \
  --prompt-file speed-bench/promessi_sposi.txt \
  --ctx-start 8192 \
  --ctx-max 8192 \
  --gen-tokens 128
```

Für jeden Lauf dokumentieren:

- Datum und Uhrzeit;
- Commit von ds4;
- exakter GGUF-Dateiname und OID, sofern verfügbar;
- resident oder `--ssd-streaming`;
- Cachebudget;
- Kontextgröße;
- `--think`/`--nothink`;
- Temperatur und Ausgabelimit;
- Prefill- und Decode-Rate;
- Speicherpressure, Swap und beobachtete Fehler;
- bei SSD-Betrieb Mount-Punkt, Device und I/O-Messung.

### 16.5 Qualitätsvergleich

Q2 und Hybrid nicht nur nach tok/s bewerten. Gleiche Prompts, gleiche Samplingparameter und gleiche Seeds verwenden. Für die Entscheidung zählen:

1. technische Korrektheit;
2. Tool-Call-Zuverlässigkeit;
3. Code- und Änderungsqualität;
4. Wiederholbarkeit;
5. erst danach Geschwindigkeit.

---

## 17. `iogpu.wired_limit_mb`: bewusst kein Standardrezept

Die Masteranleitung empfiehlt eine dauerhafte Erhöhung auf `114688`. Diese Empfehlung wird in der konsolidierten Fassung nicht als Pflicht übernommen.

Der aktuelle ds4-Code liest `iogpu.wired_limit_mb` als explizit gesetztes GPU-Budget und kann damit seine Memory-Guard-Budgetierung beeinflussen. Daraus folgt aber nicht, dass ein Wert von 114688 für jedes Single-Model-Setup erforderlich oder sicher ist.[8]

Die lokale Statusprüfung vom 07.08.2026 meldete außerdem:

```text
iogpu.wired_limit_mb: 118000
iogpu.dynamic_lwm: 1
```

Diese Werte werden in der Anleitung dokumentiert, aber nicht als Empfehlung interpretiert. Für `iogpu.*` liegen hier keine belastbaren öffentlichen Apple-Spezifikationen zu empfohlenen Werten vor.

### Sichere Policy

1. Aktuellen Wert dokumentieren und unverändert lassen.
2. Erst einen normalen Metal-Build und einen kleinen Q2-Test ausführen.
3. Bei einem konkreten Speicherfehler den Startlog und die aktuelle Speicherbelegung sichern.
4. Prüfen, ob der Fehler wirklich aus dem GPU-Budget und nicht aus Modellgröße, Kontext, Konkurrenzprozess oder falschem Artefakt stammt.
5. Falls ein temporärer Test fachlich gerechtfertigt ist, nur für diesen Lauf setzen:

```bash
sudo sysctl iogpu.wired_limit_mb=114688
sysctl iogpu.wired_limit_mb
```

6. Nach dem Test auf den Systemdefault zurücksetzen oder rebooten; keinen persistenten LaunchDaemon ohne dokumentierten Bedarf installieren.

**Kosten-Nutzen:** Der mögliche Nutzen ist ein größerer expliziter GPU-Budgetrahmen. Der Preis ist weniger Reserve für macOS und ein systemweiter, schwerer nachvollziehbarer Eingriff. Für den ersten Betrieb überwiegt das Risiko den Nutzen.

---

## 18. Troubleshooting

### `command not found: make`

```bash
xcode-select --install
```

Danach neues Terminal öffnen und `make --version` prüfen.

### Falsche Modellversion geladen

```bash
./download_model.sh --help
ls -lh gguf/
readlink ds4flash.gguf
```

Targetname und finalen Dateinamen getrennt prüfen. `-0731` nicht hineininterpretieren.

### `unsupported GGUF type 39 (MXFP4)`

MXFP4 nicht weiter testen. Q2, Hybrid oder Q4-K verwenden. Issue #641 dokumentiert den aktuellen Loader-Gap.[7]

### `--ssd-streaming is not compatible with --mtp yet`

MTP/DSpark entfernen oder SSD-Streaming deaktivieren. Diese Kombination ist im aktuellen Runtime-Code hart abgewiesen.[6][8]

### Modell lädt nicht wegen Speicherbedarf

In dieser Reihenfolge:

```text
Kontext reduzieren
→ --nothink und kurze Ausgabe
→ konkurrierende Modelle stoppen
→ Hybrid testen
→ Q2 als Referenz
```

Nicht zuerst `iogpu.wired_limit_mb` dauerhaft verändern.

### Server startet, Client verbindet sich nicht

```bash
lsof -iTCP:8000 -sTCP:LISTEN -P -n
curl -i http://127.0.0.1:8000/v1/models
```

Wenn aus einem anderen Arbeitsverzeichnis gestartet wird, `--chdir "$DS4_DIR"` setzen.

### SSD-Messung oder Streaming zeigt unerwartete Werte

Die externe SSD meldet in v2 einen Einzelwert von 6,36 GB/s Read. Für die Diagnose den tatsächlichen Mount-Punkt und Device-Namen dynamisch ermitteln; `/dev/disk4` ist nicht dauerhaft garantiert. Einen niedrigen momentanen `iostat`-Wert nicht isoliert als Beweis gegen aktives Streaming interpretieren: Startlog, Modellpfad, Cacheplan und wiederholbare Laufzeitmessung gemeinsam bewerten.

---

## 19. Rollback

### ds4-Prozesse beenden

Vor dem Beenden identifizieren:

```bash
pgrep -af 'ds4-server|/ds4( |$)' || true
```

Dann den konkreten Prozess kontrolliert beenden, statt pauschal jedes ähnlich benannte Programm zu töten.

### KV-Cache löschen

```bash
rm -rf "$HOME/.ds4/server-kv"/*
```

Bei SSD-Verwendung erst den tatsächlichen Pfad prüfen:

```bash
printf '%s\n' "$DS4_SSD/ds4/kv"
```

### Modellpfad zurücksetzen

Wenn `gguf` ein Symlink ist:

```bash
cd "$DS4_DIR"
ls -ld gguf
rm gguf
```

Das löscht nur den Symlink, nicht die Modelldateien. Danach kann das zuvor gesicherte interne Verzeichnis zurückbenannt werden.

### SSD-Ausschlüsse zurücknehmen

Nur wenn sie zuvor auf genau diesem Volume gesetzt wurden:

```bash
sudo mdutil -i on "$DS4_SSD"
sudo tmutil removeexclusion "$DS4_SSD"
```

### GPU-Wired-Limit

Keinen persistenten LaunchDaemon installieren. Ein temporär gesetzter Wert gilt bis Neustart bzw. bis zum expliziten Zurücksetzen nach erfolgreicher Prüfung. Vor dem Zurücksetzen den aktuellen Wert dokumentieren.

---

## 20. Endgültige empfohlene Reihenfolge

### Bereits erfolgt bzw. belegt

```text
1. SSD identifiziert und als APFS-Volume /Volumes/AIModels gemountet
2. 80-Gb/s-TB-Link verifiziert
3. externer 8-GiB-Read-Test mit 6,36 GB/s dokumentiert
4. internen Vergleich als offene A/B-Messung markiert
5. Storage-Entscheidung getrennt nach Modell-GGUF und KV-Snapshots festgelegt
```

### Als Nächstes: residenter 0731-Hybrid

```text
6. ds4-Commit und Metal-Build verifizieren
7. 0731-Hybrid per HF-Dateiname laden bzw. vorhandene Datei prüfen
8. gguf-Verzeichnis auf /Volumes/AIModels/ds4/gguf verlinken
9. ds4flash.gguf explizit auf den -0731-Hybrid setzen
10. --inspect und kurzen 32K-CLI-Test ausführen
11. 131072-Kontext schrittweise testen
12. Swap, Memory Pressure und Startlog dokumentieren
13. KV-Disk-Cache auf der externen SSD im residenten Betrieb aktivieren
14. lokalen Server direkt testen
15. LiteLLM und Agenten integrieren
```

### Später: Q4-SSD-Streaming

```text
16. 0731-Q4-Datei auf derselben externen SSD ablegen
17. Q4 ausschließlich mit --ssd-streaming starten
18. automatisches Expert-Cache-Budget und tatsächliche SSD-I/O dokumentieren
19. KV-Snapshots zunächst intern ablegen
20. externen KV-Disk-Cache als separaten A/B-Test gegen den internen Pfad messen
21. erst danach eine dauerhafte Co-Location von Gewichten und KV entscheiden
```

---

## 21. Akzeptanz-Checkliste

### Speicherplanung für die drei Hauptszenarien

- [ ] Szenario 1 resident ohne `--ssd-streaming` getestet.
- [ ] Szenario 2 mit `--ssd-streaming` und automatischer Budgetierung getestet.
- [ ] Szenario 3 mit `--ssd-streaming --ssd-streaming-cache-experts 48GB` getestet.
- [ ] Für Szenario 2/3 ein normaler Startlog mit dem gewünschten `--ctx` gespeichert; `--inspect` allein reicht für die Speicherplanung nicht.
- [ ] Im Startlog residenter Modellanteil, Expertengröße, Expert-Cache, Prefill-Reserve und KV-/Buffer-Bedarf dokumentiert.
- [ ] Für das Ziel `R + E + H + B <= 70 GiB` keine GGUF-Gesamtgrößen-Subtraktion als Beweis verwendet.
- [ ] Für das Ziel `R + E + H + B + K(ctx) <= 100–110 GiB` den aktiven KV für jede Kontextstufe real gemessen.
- [ ] Kontextstufen 32K → 64K → 128K → 256K → 500K nur bei stabiler Memory Pressure getestet.

### Grundvoraussetzungen und residenter Betrieb

- [ ] SSD identifiziert, gemountet und Dateisystem geprüft.
- [ ] freier interner und externer Speicher geprüft.
- [ ] ds4-Commit dokumentiert.
- [ ] fünf Metal-Binaries gebaut.
- [ ] `./ds4 --help` erfolgreich.
- [ ] `./ds4-server --help` erfolgreich.
- [ ] externer 8-GiB-Read-Test mit Bedingungen dokumentiert.
- [ ] interner A/B-Vergleich als offen oder durchgeführt dokumentiert.
- [ ] 0731-Hybrid-Dateiname und HF-OID geprüft.
- [ ] `gguf`-Symlink auf `/Volumes/AIModels/ds4/gguf` geprüft.
- [ ] `ds4flash.gguf` zeigt auf den 0731-Hybrid.
- [ ] `--inspect` erfolgreich.
- [ ] kurzer 32K-CLI-Test erfolgreich.
- [ ] 131072-Kontext schrittweise getestet oder als offen markiert.
- [ ] kein kritischer Memory Pressure-/Swap-Anstieg.

### Residenter Betrieb

- [ ] Hybrid resident getestet.
- [ ] während der Generierung keine relevante Gewichts-I/O erwartet bzw. beobachtet.
- [ ] KV-Cache auf der externen SSD getestet.
- [ ] KV-Cache-Datenschutz bewertet.
- [ ] direkter Server-Endpoint getestet.
- [ ] LiteLLM erst danach integriert.

### Q4-SSD-Streaming

- [ ] Q4-0731 auf der externen SSD abgelegt.
- [ ] Q4 nur mit `--ssd-streaming` getestet.
- [ ] automatisches Expert-Cache-Budget und Startlog dokumentiert.
- [ ] MTP/DSpark nicht mit SSD-Streaming kombiniert.
- [ ] KV-Snapshots zunächst intern getestet.
- [ ] externer KV-Pfad im A/B-Test gegen intern verglichen.
- [ ] SSD-I/O nicht als alleiniger Streamingnachweis verwendet.

---

## 22. Quellen

[1] DwarfStar ds4 README, aktueller Recheck-Commit `b0309611041655f4e45671cfd9c9886aff161406`: https://raw.githubusercontent.com/antirez/ds4/b0309611041655f4e45671cfd9c9886aff161406/README.md[2] Download-Skript mit Target-zu-Datei-Mapping, aktueller Recheck-Commit: https://raw.githubusercontent.com/antirez/ds4/b0309611041655f4e45671cfd9c9886aff161406/download_model.sh[3] macOS-Makefile, aktueller Recheck-Commit: https://raw.githubusercontent.com/antirez/ds4/b0309611041655f4e45671cfd9c9886aff161406/Makefile[4] Offizielle Hugging-Face-Dateiliste und LFS-Größen: https://huggingface.co/api/models/antirez/deepseek-v4-gguf/tree/main?recursive=true&expand=true[10] Aktuelle Hugging-Face-Dateiansicht mit den `-0731`-Artefakten: https://huggingface.co/antirez/deepseek-v4-gguf/tree/main[11] Upstream-Recheck `main` am 07.08.2026, Commit `b0309611041655f4e45671cfd9c9886aff161406`: https://github.com/antirez/ds4/commit/b0309611041655f4e45671cfd9c9886aff161406[5] M4-Max-128-GiB-SSD-Streaming-Benchmark, historische Referenz: https://github.com/antirez/ds4/issues/437[6] Explizite Inkompatibilität `--ssd-streaming` + `--mtp`: https://github.com/antirez/ds4/issues/596[7] MXFP4-Loader-Lücke, `unsupported GGUF type 39`: https://github.com/antirez/ds4/issues/641[8] ds4-Core mit Runtime- und Speicherprüfungen, aktueller Recheck-Commit: https://raw.githubusercontent.com/antirez/ds4/b0309611041655f4e45671cfd9c9886aff161406/ds4.c[9] ds4-Server mit Optionen und KV-Cache-Konfiguration, aktueller Recheck-Commit: https://raw.githubusercontent.com/antirez/ds4/b0309611041655f4e45671cfd9c9886aff161406/ds4_server.c

> **Qualitätshinweis:** Diese Anleitung enthält keine erfundenen SSD-Messwerte. Die externe SSD war zum Konsolidierungszeitpunkt der v1 nicht angeschlossen. Für v2 ist ein externer 8-GiB-Read mit 6,36 GB/s dokumentiert; ein identischer interner A/B-Test, thermische Sustained-Messung und echter Q4-Streaming-Lauf bleiben offen.

**Erstellt: 03.08.2026 · aktualisiert/konsolidiert: 07.08.2026 · Zielsystem: Mac Studio M4 Max `Mac16,9`, 128 GiB · Engine: DwarfStar/ds4, Beta**
