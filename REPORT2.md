# REPORT 2 — GLM-5.3-Flash-Q4_K SSD-Streaming: pread-Bündelung (Teil 2) und MTP-Union (Teil 1)

Worktree: `/Users/michaelkuebbeler/ai-workspace/localAI/ds4-kernel-async`
Branch: `exp/kernel-glm-async-selected` (Basis `df9f2cf`)
Stand: 15.09.2026 · Protokoll: `--ssd-streaming --ctx 4096 --tokens 128 --temp 0`, Prompt erzeugt 128 echte Tokens

**Ergebnis in drei Sätzen: Die Experten-Union über Draft+Verify-Tokens wird bereits heute gemeinsam
bedient — ein Layer-Durchlauf, ein deduplizierter Ladevorgang (gemessen: 13,14 unique Experten von 16
Slots je Layer-Pass), es gibt nichts zu implementieren. Der MTP-Verlust von 15 % ist mit der
Accept-Rate und der head+draft-Phase vollständig erklärt, nicht mit per-Token-Lookups. Für Teil 2
sind sowohl „Teilreads parallel" als auch „Reads früher anstoßen" widerlegt: die drei Teilreads eines
Experts sind bereits parallel (Nebenläufigkeit 2,98 im Busy-Fenster), das Zielgerät sättigt bei
Queue-Tiefe 1–2 (4,92 GiB/s bei 4,5 MiB, flach bis QD 9), und der implementierte Frühstart-Versuch
verschlechtert um 2,4 % — er wurde zurückgenommen.**

---

## 1. Umfeld

| Punkt | Wert |
|---|---|
| Commit | Basis `df9f2cf`; neue Commits `f60ab6d` (Teil-2-Versuch), `84003bc` (Diagnostik), `3a9f312` (Revert des Versuchs), `5e8e350` (Diagnostik-Ausgabe) |
| Build | `make -j8` fehlerfrei/warnungsfrei nach jedem Patch (5 Binaries: `ds4`, `ds4-server`, `ds4-bench`, `ds4-eval`, `ds4-agent`) |
| Backup | Branch `backup-before-pread2-20260915-181058`; Binaries `/tmp/ds4{,-server,-bench}.bin.prea2-df9f2cf`, SHA-256 `/tmp/prea2-backup-sha.txt` |
| Server-Status | Port 8008 während **aller** Läufe frei (`lsof -nP -iTCP:8008 -sTCP:LISTEN` leer). Kein `ds4-server` gestartet/gestoppt |
| Modell | `/Volumes/AIModels/ds4/gguf/GLM-5.3-Flash-Q4_K.gguf` (190.875.526.464 B), `/dev/disk7s1` |
| Host | Mac Studio M4 Max, 128 GiB, macOS 27.0 |
| Läufe | 8 CLI-Läufe à 128 Token, strikt sequenziell, nie zwei gleichzeitig |
| Hard Rules | eingehalten: kein `git push`, kein Issue/PR, keine Änderung außerhalb des Worktrees, keine `AGENTS.md`, kein `rm -rf`, keine Installation, keine Modell-/Symlink-/KV-Operation |

## 2. Messreihe Baseline vs. Fix (je 2 Läufe, nur der zweite ist vergleichbar)

| Lauf | Binary | Konfiguration | prefill t/s | **decode t/s** | Gate `cache aging tokens` | Output | Byte-Gate (`cmp` vs 584 B) |
|---|---|---|---|---|---|---|---|
| B1 | `ds4.bin.prea2-df9f2cf` (Original) | §9 | 2,42 | **6,24** | 127 | 584 B | **PASS** |
| B2 | Original | §9 | 2,57 | **6,24** | 127 | 584 B | **PASS** |
| N1 | V1 (Commit `f60ab6d`) | §9 | 2,59 | **6,08** | 127 | 584 B | **PASS** |
| N2 | V1 | §9 | 2,58 | **6,10** | 127 | 584 B | **PASS** |
| M0 | Endstand (`3a9f312` + Diagnostik) | §9 + `DS4_METAL_MOE_PATH_PROFILE=1` | 2,58 | **6,25** | 127 | 584 B | **PASS** |
| F1 | Endstand | §9 (ohne Profil-Env) | 2,49 | **6,25** | 127 | 584 B | **PASS** |
| MTP1 | Endstand | §9 + `--mtp --mtp-timing` | 2,58 | **5,10** | 127 | 584 B | **PASS** |
| MTP2 | Endstand | §9 + `--mtp` | 2,51 | **5,12** | 127 | 584 B | **PASS** |

**Bewertung:** V1 = −2,4 % (6,09 vs. 6,24 Mittel) → verworfen. Endstand = 6,25 t/s, identisch zur
Baseline ⇒ Diagnostik verhaltensneutral, Revert vollständig, Byte-Gate in **allen 8 Läufen** PASS.
`--mtp` = −18 % (5,11 vs. 6,25) — reproduziert die bekannten −15 % und verschärft sie leicht.

Rohdaten: `/tmp/ds4-ab-{B1,B2,N1,N2,M0,F1,MTP1,MTP2}.{out,err}`,
Gerätebenchmark: `/tmp/io-bench3.py`, Trace: `/tmp/ds4-kernel-io.txt`.

## 3. Teil 2 — pread-Bündelung/Parallelität im Expert-Load-Pfad

### 3a. Strukturanalyse (Code-Anker, `ds4_metal.m`)

| Anker (`ds4_metal.m`, Endstand dieses Branches) | Bedeutung |
|---|---|
| `ds4_gpu_glm_routed_moe_one_tensor` (38411) | Einzel-Token-Decode: Peek je Experte, dann `load_selected_missing` oder Deferred-Split |
| `ds4_gpu_stream_expert_cache_begin_selected_load` (16813) | Early-Load: peek → **inline** `F_RDADVISE` (3 Aufrufe/Expert) → Slot-Buffer → Task-Aufbau → `pread_pool_begin` (asynchron) |
| `ds4_gpu_stream_expert_cache_load_selected_missing_with_source` (17109) | Synchroner Ladepfad, gleiche Reihenfolge |
| `ds4_gpu_stream_expert_pread_into` (13688) | Liest eine Range vollständig in den Slot-Puffer |
| `ds4_gpu_stream_expert_pread_pool_{worker,begin,wait}` (13771 / 13871 ff.) | 9 Threads, Task-Queue, `n_workers = min(9, n_tasks)` |

**Warum drei Reads je Expert:** Ein GLM-Expert belegt 13,5 MiB verteilt auf drei **nicht
zusammenhängende** Tensoren (`ffn_gate_exps`, `ffn_up_exps`, `ffn_down_exps`, je 4,5 MiB je Expert,
Tensorbasen ~1,2 GiB auseinander). Die im Auftrag vermutete „Zerlegung eines 13,5-MiB-Slabs in ~4,5-MiB-
Teilreads" ist also kein Zerlegungsfehler, sondern die Tensorstruktur.

**Warum 1,39:** Die drei Reads eines Experts werden als drei Tasks an den Pool gegeben und laufen
parallel. Im Phase-0-Trace (Lauf C, Decode) beträgt die Nebenläufigkeit **in den belegten Fenstern
2,98** (= 3 Tensoren) — die 1,39 sind das Verhältnis Σ Read-Dauer / Wandzeit, also ein
**Auslastungs-/Duty-Cycle-Maß** über die gesamte Decode-Zeit (43,3 % der Zeit ist überhaupt ein Read
in flight). Ein Parallelitätsmangel des Pools liegt nicht vor; mehr als 3 gleichzeitige Reads kann
eine einzelne Layer-Auswahl nicht erzeugen (mittlere fehlende Experten: 0,92 je Layer×Token).

| Größe (Decode, Trace Lauf C) | Wert |
|---|---|
| Reads | 14.718 (3 × fehlende Experten) |
| Bytes | 64,68 GiB = 0,51 GiB/Token |
| Wandzeit | 20,23 s → **3,20 GiB/s aggregiert** |
| Device-Busy (Union der pread-Intervalle) | 43,3 % |
| Nebenläufigkeit: Mittel / im Busy-Fenster | 1,28 / **2,98** |
| Read-Dauer p50 / Mittel | 1,34 ms / 1,78 ms je 4,5 MiB |

### 3b. Gerätemessung (neu; read-only, `/tmp/io-bench3.py`)

Verfahren: `O_RDONLY` + `F_NOCACHE` + `F_RDAHEAD=0`, frische, disjunkte 1-GiB-Regionen je Konfiguration
(Cursor `/tmp/io-bench-cursor`), 4-KiB-alignierte Zufallsoffsets, wiederverwendete Zielpuffer wie im
Expert-Cache. Keine Schreibzugriffe, keine Adminrechte.

| Read-Größe | QD 1 | QD 2 | QD 3 | QD 6 | QD 9 |
|---|---|---|---|---|---|
| 4,5 MiB (Ist-Fall) | **4,92** | 5,12 | 4,68 | 4,76 | 4,71 GiB/s |
| 1 MiB | 3,30 | — | — | 5,44 (QD4) | — |
| 13,5 MiB | **5,50** (2,39 ms) | — | — | 5,95 (QD4) | — |
| sequenziell 512 MiB (4-MiB-Chunks) | **7,63** | — | — | — | — |

- Die p50-Latenz skaliert **linear** mit der Queue-Tiefe (0,89 ms → 8,84 ms von QD1 auf QD9):
  das Gerät ist ab **QD ≈ 1–2 gesättigt**. Zusätzliche Nebenläufigkeit im Ladepfad kann den Durchsatz
  nicht erhöhen — sie erhöht nur die Latenz der einzelnen Reads.
- Größere Reads wären nur minimal besser (13,5 MiB QD1: 5,50 vs. 3 × 4,5 MiB QD3: 4,68 GiB/s), und
  sie sind strukturell nicht möglich (nicht zusammenhängende Tensoren).
- Konsequenz für das Auftragsziel: 0,52 GiB/Token ÷ 4,9 GiB/s ⇒ **109 ms/Token** ist die
  I/O-Untergrenze bei perfekter Überlappung (Ceiling ≈ 9,2 t/s). Gemessen werden 162 ms/Token ⇒
  67 % Geräte-Effizienz; der Rest ist die serielle Kette Router → Laden → MoE pro Layer, die eine
  Überlappung über Layer-Grenzen ausschließt (der Router von Layer n+1 braucht die MoE-Ausgabe von n).

### 3c. Implementierter Versuch V1 (Commit `f60ab6d`, Revert `3a9f312`)

**Idee:** Die `F_RDADVISE`-Aufrufe (3 je fehlendem Expert, gemessen 0,192 ms je Aufruf =
**22,7 ms/Token** auf dem Encode-Thread) aus dem seriellen Pfad nehmen und als Pool-Tasks *vor* die
preads legen — Read-Ahead parallel und früher, ohne Arithmetik zu berühren.

| Messgröße (Decode-Delta) | Baseline (B2) | V1 (N2) | Δ |
|---|---|---|---|
| `bind_total` (Adress-/Lade-Aufbau) | 3.162 ms | **372 ms** | **−88 %** (−22 ms/Token Host-Zeit) |
| `readahead_avg` je Aufruf | 0,192 ms | 0,724 ms | +277 % (Kernel serialisiert parallele `fcntl`) |
| `missing_bind_total` | 2.602 ms | 3.439 ms | +32 % |
| `load_pread_avg` | 2,86 ms | 3,87 ms | +35 % |
| **decode** | **6,24 t/s** | **6,09 t/s** | **−2,4 %** |

**Interpretation:** Der Versuch war mechanisch erfolgreich (Host-Zeit aus dem Pfad) und trotzdem
langsamer. Damit ist belegt: **Der Host-Anteil des Ladepfads liegt nicht auf dem kritischen Pfad**
(er wird hinter der I/O versteckt), während der *zeitliche Vorlauf* des `F_RDADVISE` vor den preads
echten Geräte-Nutzen hat (vgl. die frühere Probe `READAHEAD=0`: −2,8 %). V1 hat den Vorlauf auf ~0
verkürzt und damit genau den Nutzen entfernt, den es retten wollte. Änderung zurückgenommen.

### 3d. Beantwortung der Teil-2-Fragen

| Auftragsfrage | Antwort (gemessen) |
|---|---|
| Warum liegt die Nebenläufigkeit bei 1,39? | Duty-Cycle-Maß, nicht Parallelitätsmangel: im Busy-Fenster sind es 2,98 (= 3 Tensoren eines Experts). |
| Warum zerfällt ein 13,5-MiB-Slab in ~4,5-MiB-Reads? | Strukturell: 3 getrennte Tensoren (gate/up/down) à 4,5 MiB, nicht zusammenhängend. |
| Teilreads eines Slabs parallel? | Sind sie bereits. Mehr Queue-Tiefe bringt nichts (Gerät sättigt bei QD 1–2). |
| Größere Read-Requests? | Nicht möglich (Offsets ~1,2 GiB auseinander); hypothetischer Gewinn ≤ 17 % Gerätedurchsatz, real 0. |
| Reads früher anstoßen? | Der Frühstart-Mechanismus (Early-Load + Read-Ahead) ist bereits der beste der getesteten Varianten; V1 (−2,4 %), `EARLY_LOAD=0` (−2,3 %), `READAHEAD=0` (−2,8 %) sind alle schlechter. |
| **Ziel „I/O-Anteil deutlich unter heute, Decode messbar über 6,2 t/s"?** | **Nicht erreicht und mit diesem Pfad nicht erreichbar.** Der Pfad ist byte-limitiert, nicht verwaltungs-limitiert. |

## 4. Teil 1 — MTP/Speculative: Experten-Union über Draft+Verify-Tokens

### 4a. Issue `antirez/ds4#495` (gelesen; Body + 7 Kommentare, letzter 02.08.2026)

- Der Guard im Upstream verbietet `--ssd-streaming` zusammen mit `--mtp-model` (separates MTP-GGUF);
  er verhindert **nicht** den eingebauten GLM-MTP-Pfad (`--mtp`), der hier läuft (5,1 t/s).
- Community-Befunde: Die Streaming-Selected-Expert-Schicht ist batchfähig (Prefill nutzt sie mit
  dedupliziertem Union-Set); der Verifier ist prefill-förmig; nach Bypass des Guards läuft
  `--mtp --ssd-streaming` token-identisch, ist aber auf disk-bound Rigs **netto langsamer**.
- Quantifizierung aus dem Issue (GLM-5.2, IQ2_XXS, CUDA): benachbarte Positionen teilen nur ~30 % der
  gerouteten Experten; „a 2-token verify batch costs ~1.85 evals even with union loads". Genau dieses
  Muster bestätigt meine Messung auf GLM-5.3-Q4/Metal (dort ~18 % Überlappung).

### 4b. Code-Kette des GLM-MTP-Verify (Anker in diesem Stand)

```
ds4_session_glm_spec_cycle_impl        ds4.c:73402   Draft-Token d, toks[2] = {first_token, d}
  └─ glm_graph_forward_tokens(..., 2)  ds4.c:73471   (glm53; alternativ forward_indexed_tokens)
       └─ glm_graph_encode_ffn_batch   ds4.c:50837   (Batch-FFN für n_tokens Zeilen)
            └─ glm_graph_routed_moe_batch_dispatch   ds4.c:48765
                 └─ ds4_gpu_glm_routed_moe_batch_tensor      ds4_metal.m:39768 (impl) / 40300
                      └─ use_stream_expert_addr_table (Bedingung u.a. n_tokens > 1)  ds4_metal.m:39972 / 40041
                           └─ ds4_gpu_stream_expert_cache_prepare_selected_batch   ds4_metal.m:17687
                                → Dedup über alle n_tokens*n_selected IDs (seen/frequency/unique_ids)
                                → EIN Ladevorgang + EINE Adresstabelle pro Layer
```
Einzel-Token-Decode dagegen: `glm_graph_encode_sparse_ffn_one` → `glm_graph_routed_moe_one_dispatch`
(ds4.c:48614) → `ds4_gpu_glm_routed_moe_one_tensor` (ds4_metal.m:38411) = Peek/Load **je Token**.

### 4c. Messnachweis (neu, env `DS4_METAL_MOE_PATH_PROFILE=1`; Commits `84003bc`, `5e8e350`)

Zähler an den MoE-Einstiegspunkten und in `prepare_selected_batch`; verhaltensneutral (siehe Lauf M0/F1:
6,25 t/s und Byte-Gate identisch zur Baseline).

| Lauf | Phase | one_calls | batch_calls | batch_tokens | batch_stream_calls | union_calls | union_unique_experts | unique je Union |
|---|---|---|---|---|---|---|---|---|
| M0 (§9, ohne MTP) | Prefill | 0 | 42 | 1.806 | 42 | 42 | 4.969 | **118,31** (von 344 Slots) |
| M0 | Decode | **5.334** (42×127) | 0 | 0 | 0 | 0 | 0 | — |
| MTP2 (`--mtp`) | Decode (Differenz total−prefill) | 1.136 | **3.108** (=42×74) | **6.216** (=2×3.108) | **3.108** | **3.108** | 40.822 | **13,14** (von 16 Slots) |

**Antwort auf die Teil-1-Frage: Ja — die Experten-Union über Draft+Verify-Tokens wird heute bereits
gemeinsam bedient.** Jeder der 3.108 Layer-Durchläufe des Verifiers (74 Zyklen × 42 Layer) geht durch
den Streaming-Batch-Pfad, ruft **einmal** `prepare_selected_batch` und lädt **einmal** die
deduplizierte Union beider Zeilen; die per-Token-Pfade (`one_calls`) stammen ausschließlich aus den
einfachen Schritten und den Draft-Durchläufen. Es gibt **nichts zu implementieren**; die im Auftrag
genannte Erwartung („MTP von −15 % auf deutlich positiv") trifft daher nicht ein.

### 4d. Warum MTP trotzdem −18 % kostet (Zyklen-Log, `--mtp-timing`)

| Größe | Wert |
|---|---|
| Zyklen / Accept | 74 Zyklen, 52 akzeptiert → **Accept-Rate 0,70**, 1,73 committete Token je Zyklus |
| `verify2` je Zyklus | 282,9 ms (p50 275,8) → **6,73 ms je Layer-Pass** (2 Zeilen) |
| Einzel-Token-Layer-Pass (Baseline) | 3,86 ms (= 162 ms / 42 Layer) |
| Union-Ersparnis je Zeile | 6,73 / (2 × 3,86) = **0,87×** (also nur −13 %, weil sich zwei Zeilen nur ~18 % Experten teilen) |
| `head+draft` je Zyklus | 39,1 ms (p50 23,6) = **12 % der Zykluszeit** |
| **Rechnung je committetem Token** | verify 282,9 / 1,73 = **164 ms ≈ Baseline 162 ms** (Ersparnis exakt durch die Accept-Rate aufgezehrt) + head+draft 22 ms ⇒ **186 ms/Token = 5,4 t/s** (gemessen 5,10/5,12) |

**Break-even-Bedingung:** Der Verifier müsste je Zeile ≤ ~0,50× eines Einzel-Token-Passes kosten
(oder die Accept-Rate deutlich höher liegen). Physikalisch ist das nicht erreichbar, weil zwei
benachbarte Zeilen nur ~18 % ihrer Experten teilen: die Bytes/Token sinken bestenfalls auf 0,925×.
Der zweite Kostenblock — die head+draft-Phase — ist strukturell erklärbar: der Draft-Durchlauf nutzt
die **letzte Modell-Layer** (`il = DS4_N_LAYER - DS4_N_NEXTN_PREDICT`, ds4.c:51518) inklusive deren
gerouteter Experten über den SSD-Streaming-Pfad, also einen zusätzlichen Layer-Pass je Zyklus.

### 4e. Empfehlung Teil 1

- `--mtp` bei GLM-5.3-Q4-K + `--ssd-streaming` **aus lassen** (Switcher-Default `DS4_GLM_MTP=0` ist
  inhaltlich korrekt). Kein Union-Code nötig.
- Wenn MTP später nochmals geprüft werden soll, ist der einzige klar umrissene Angriffspunkt die
  **head+draft-Phase (22 ms/Token = 12 %)** — konkret: die MTP-Layer (letzte Layer, 288 Experten
  ≈ 3,8 GiB) resident halten, statt sie je Zyklus über den Streaming-Pfad zu bedienen.
- Für Upstream (nur vorbereitet, **nicht gepostet** — Hard Rule): Beleg, dass der Metal-GLM-Pfad die
  Union bereits nutzt (13,14 unique/16 Slots, verify = 1,74× eines Einzel-Passes), und dass der
  MTP-Verlust auf der Accept-Rate plus head+draft beruht, nicht auf per-Token-Lookups. Damit ist die
  Frage aus Issue #495 für Metal präziser beantwortbar als im Thread bisher geschehen.

## 5. Diffs und Build

| Commit | Inhalt | Wirkung |
|---|---|---|
| `f60ab6d` | `ds4_metal.m`: `readahead_only`-Flag im pread-Task, `ds4_gpu_stream_expert_pread_task_run()`-Helfer (beide Worker nutzen ihn), Read-Ahead-Tasks vor den pread-Tasks in beiden Ladepfaden, Task-Kapazität 3×→6× je Selected-Slot | gemessen **−2,4 %** |
| `84003bc` | `ds4_metal.m`: env-gated Zähler + Ausgabe des routed-MoE-Pfads (`DS4_METAL_MOE_PATH_PROFILE`) | verhaltensneutral (M0: 6,25 t/s) |
| `3a9f312` | Revert von `f60ab6d` | Endstand wieder 6,25 t/s |
| `5e8e350` | `ds4_metal.m`: zusätzlicher `atexit`-Dump der Zähler (der MTP-Loop erreicht die Memory-Report-Snapshots nicht) | verhaltensneutral (F1: 6,25 t/s, Byte-Gate PASS) |

`make -j8` nach jedem Schritt fehler- und warnungsfrei; deutlich erkennbar an den unveränderten
Laufzeiten (6,24/6,24 → 6,25/6,25) und dem durchgehend grünen Byte-Gate.

## 6. Abweichungen vom Auftrag und Begründung

1. **Zusatzmessung „Gerätemessung" (read-only).** Sie entscheidet, ob Teil 2 überhaupt einen Hebel
   hat; ohne sie wäre die Aussage „mehr Parallelität hilft nicht" nicht belegbar. Keine Schreibzugriffe,
   keine Adminrechte, keine Modell-/KV-Operation.
2. **Zwei `--mtp`-Läufe** (einer zusätzlich mit `--mtp-timing`). Regel „kein `--mtp` ohne
   MTP-Fragestellung" ist eingehalten — Teil 1 ist genau diese Fragestellung.
3. **V1 implementiert und wieder verworfen.** Der Auftrag erlaubt „ggf."; das negative Ergebnis ist
   belastbar (2 Läufe je Variante, Byte-Gate durchgehend PASS) und im Branch als Commit + Revert
   nachvollziehbar. Der Revert ist sauber (Endstand messgleich zur Baseline).
4. **`iostat`-Monitoring nicht möglich** (`iostat: could not record 'disk7'`, ohne Root) — daher keine
   separate Geräte-Byte-Messung während eines Laufs; die Applikationszahlen (`miss_pread`,
   Trace) wurden stattdessen verwendet.
5. **Teil-2-Ziel nicht erreicht, Teil-1-Union nicht implementiert** — beides nicht als „nicht
   geschafft", sondern als gemessen begründete Verwerfung (siehe §3d, §4c).

## 7. Offene Punkte, Fazit und Empfehlung

1. **Ziel ≥ 10 t/s bleibt auf diesem Pfad unerreichbar.** Harte Zahlen: 0,51–0,52 GiB/Token,
   Gerätedurchsatz 4,92 GiB/s bei 4,5-MiB-Random-Reads (sättigt bei QD 1–2) ⇒ 109 ms/Token
   I/O-Untergrenze = 9,2 t/s Ceiling, und das nur bei perfekter Überlappung, die die serielle
   Layer-Abhängigkeitskette nicht zulässt. Gemessen: 162 ms/Token (6,25 t/s) = 67 % Geräte-Effizienz.
2. **Der einzige belastbare Hebel bleibt das Lesevolumen je Token** — bestätigt durch den früheren
   Q2-Test (dasselbe Binary: 21,67 t/s streaming / 28,63 t/s resident vs. 6,25 t/s Q4). Optionen mit
   Datenlage: GLM-Q2, DeepSeek-MXFP4-Streaming (15,56 t/s), höhere Hit-Rate. Cache-Budget-Sweeps sind
   widerlegt (48/56/85 GB schlechter als Auto 70,1 GiB).
3. **Teil 1 abgeschlossen:** Union vorhanden und wirksam, kein Implementierungsbedarf; MTP bleibt bei
   Q4-Streaming deaktiviert. Nächster sinnvoller MTP-Schritt wäre die Resident-Haltung der
   MTP-Layer (head+draft = 12 % der Zykluszeit) — als eigener, kleiner Auftrag.
4. **Aufräumen:** Worktree enthält die Commits `f60ab6d`, `84003bc`, `3a9f312`, `5e8e350` sowie
   `FORTSCHRITT2.md`/`REPORT2.md`; Produktions-Klon `/Users/michaelkuebbeler/ai-workspace/localAI/ds4`
   unverändert (kein Commit, kein Push, Port 8008 frei, kein Server gestartet/gestoppt).
