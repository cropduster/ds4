# REPORT — GLM-5.3-Flash-Q4 SSD-Streaming-Decode, Kernel-/Engine-Arbeit (15.09.2026)

Worktree: `/Users/michaelkuebbeler/ai-workspace/localAI/ds4-kernel-async`
Branch: `exp/kernel-glm-async-selected` (Basis `main` = 7a8ca68)
Bericht nach §10 des Agent-Prompts. **Ergebnis in einem Satz: Das Ziel ≥ 10 t/s ist mit keinem der
in §6 genannten Kandidaten erreichbar — die Phase-0-Messung zeigt, dass die Token-Zeit zu ~30 % aus
nicht vermeidbarem SSD-Lesevolumen und zu ~31 % aus GPU-Arbeit besteht, die durch die
Layer-Abhängigkeitskette seriell und nicht überlappbar sind; der gesamte adressierbare Host-Anteil
(Adressauflösung) beträgt 0,49 ms von 162 ms (0,3 %).**

---

## 1. Umfeld

| Punkt | Wert |
|---|---|
| Commit | `7a8ca68` (Basis), Arbeitsbranch `exp/kernel-glm-async-selected`; Instrumentierungs-Commit `73be92c` |
| Build | `make -j8` fehlerfrei (16 Kerne); Binaries `ds4`, `ds4-server`, `ds4-bench`, `ds4-eval` gebaut 14:42 bzw. 14:50 |
| Backup | Branch `backup-before-kernel-async-20260915-143809`; Binaries `/tmp/ds4{,-server,-bench}.bin.orig-7a8ca68-kernel`, SHA-256 in `/tmp/kernel-backup-sha.txt` |
| Server-Status | Port 8008 während **aller** Läufe frei (`lsof -nP -iTCP:8008 -sTCP:LISTEN` leer). Kein Server gestartet/gestoppt. LiteLLM/andere Dienste unberührt. |
| Modell | `/Volumes/AIModels/ds4/gguf/GLM-5.3-Flash-Q4_K.gguf`, 190.875.526.464 B (177,77 GiB), `/dev/disk7s1` (APFS, Solid State, PCI-Express) |
| Host | Mac Studio M4 Max, 128 GiB, macOS 27.0 (26A428); Speicher während der Läufe 95 % frei |
| Läufe | 11 CLI-Läufe à 128 Token, strikt sequenziell (95–100 GiB je Lauf), Protokoll §9 unverändert |
| Hard Rules | eingehalten: kein `git push`, kein Issue/PR, keine Änderung außerhalb des Worktrees, keine AGENTS.md, kein Serverstart, kein `rm -rf`, keine Modell-/Symlink-/KV-Operation |

## 2. Messreihe (alle Läufe: `--ssd-streaming --ctx 4096 --tokens 128 --temp 0`, Warm-Läufe)

| Lauf | Konfiguration | decode t/s | `cache aging tokens` | Output-Bytes | Byte-Gate |
|---|---|---|---|---|---|
| base1 | Original-Binary (Referenzprüfung) | 6,41 | 127 | 584 | **PASS** |
| A | Phase 0 | 6,20 | 127 | 584 | **PASS** |
| B | Phase 0 + Encoder-Timeline | 6,04 | 127 | 584 | **PASS** |
| C | Phase 0 + I/O-Trace | 6,22 | 127 | 584 | **PASS** |
| D | Phase 0 + Feinzerlegung | 6,21 | 127 | 584 | **PASS** |
| — | Referenz aus dem Auftrag (Original, 2 Läufe) | 6,21 / 6,22 | 127 | 584 | PASS |

Rohdaten: `/tmp/ds4-kernel-base1.{out,err}`, `/tmp/ds4-kernel-p0{a,b,c,d}.{out,err}`,
I/O-Trace `/tmp/ds4-kernel-io.txt`, Encoder-Timeline `/tmp/ds4-kernel-tl.txt`.
Reproduzierbarkeit der Baseline im Worktree: 6,20 / 6,21 / 6,22 t/s (A/C/D) — die Auftragsreferenz
(6,21/6,22) ist damit **bestätigt**, Abweichung < 0,5 %.

### 2b. Probe-Matrix Phase 1 (I/O-Pfad, nicht durch §4 ausgeschlossen)

Jeweils 1 Lauf, sonst identisches Protokoll, alle **Byte-Gate PASS** (584 B):

| Konfiguration | decode t/s | Δ zur Baseline |
|---|---|---|
| Baseline (Default) | 6,20 / 6,22 | — |
| `DS4_METAL_DISABLE_GLM_STREAMING_EXPERT_EARLY_LOAD=1` | 6,06 | −2,3 % |
| `DS4_METAL_STREAMING_EXPERT_PREAD_THREADS=18` | 6,20 / 6,21 | ±0 % |
| `DS4_METAL_STREAMING_EXPERT_PREAD_THREADS=4` | 6,22 | ±0 % |
| `DS4_METAL_STREAMING_EXPERT_PREAD_POOL=0` (synchron) | 6,03 | −2,8 % |

⇒ Der I/O-Pfad ist **unempfindlich gegen Nebenläufigkeit** (4 vs. 18 Threads identisch) und die
Default-Einstellungen (Pool an, Early-Load an) sind bereits die besten der geprüften Varianten.
Damit ist auch die naheliegende Phase-1-Variante „mehr Parallelität im Expert-Load" empirisch
widerlegt, bevor Code geschrieben wurde.

## 3. Phase-0-Instrumentierungsergebnis (GPU-Ausführung vs. Host-Zeit je Layer×Token)

Verfahren (neu, env-gated, verhaltensneutral):
- **GPU-Ausführungszeit**: Metal-Timestamps `cb.GPUStartTime/GPUEndTime` jedes Command Buffers,
  aufsummiert als *Union* der Intervalle (`gpu_busy_union`) und zusätzlich als naive Summe
  (`gpu_span_sum`) — beide identisch ⇒ keine überlappenden Command Buffer.
- **Host-Wartezeit**: Wandzeit in `waitUntilCompleted`.
- **Host-Kette des GLM-Routed-MoE**: Sub-Timer für Cache-Peek, Missing-Load, Slot-Install, Tail.
- **I/O**: Intervalltrace aller Expert-`pread`s (Start/Ende/Bytes).
- Kontrast: Encoder-Timeline (`MTLCounterSampleBuffer` je Dispatch, Kernel-Namen).

### 3a. Token-Budget (Decode, 127 Token, µs-genau pro Lauf reproduziert)

| Posten | ms/Token | Anteil |
|---|---|---|
| Wandzeit gesamt | **162,0** | 100 % |
| GPU-Ausführung (echte GPU-Zeit) | 49,5 | 31 % |
| Host in `waitUntilCompleted` (enthält die GPU-Arbeit der gewarteten CB) | 64,6 | 40 % |
| — davon reine Warte-/Latenzzeit (Warte − GPU gesamt) | ~15 | 9 % |
| Host: Warten auf fehlende Experten (SSD-pread, im MoE-Pfad) | 48,4 | 30 % |
| Host: Early-Load-/RDADVISE-Aufbau (`bind`) | 25,2 | 16 % |
| Host: Cache-Peek (8 Experten/Layer) | 0,13 | 0,1 % |
| Host: Adress-Slot-Install | 0,36 | 0,2 % |
| Command Buffer je Token | 70,7 (1,7 je Layer) | — |

**Damit ist die Kernfrage des Auftrags beantwortet:** Die Stage-Zahl `pair=0,31 ms` war tatsächlich
keine GPU-Messung; die echte GPU-Ausführungszeit ist **49,5–50,4 ms/Token (31 %)**, nicht 15 %.
Die restlichen 69 % sind Host-/Wartezeit, davon 30 Prozentpunkte unvermeidbares SSD-Lesevolumen.

### 3b. GPU-Attribution je Kernel (Encoder-Timeline, Lauf B, Decode-Teil)

| Kernel | GPU-Zeit | Anteil | Dispatches/Token |
|---|---|---|---|
| `kernel_glm53_mul_mv_bf16_f32` (residente, nicht-geroutete Gewichte) | 38,0 ms/Token | 59,5 % | 419 |
| `kernel_glm_q4_K_addr_pair_swiglu_f32` (+`_masked`) | 8,9 ms/Token | 12,7 % | ~45 |
| `kernel_glm_q4_K_addr_down_simd_f32` | 4,2 ms/Token | 6,6 % | 42 |
| `kernel_mul_mv_q8_0_f32` | 4,1 ms/Token | 6,5 % | 89 |
| `kernel_dsv4_shared_mid_swiglu_q8_0` | 1,8 ms/Token | 2,9 % | 42 |
| `kernel_dsv4_hc_split_weighted_sum` / `hc_expand4` | 2,0 ms/Token | 3,1 % | — |
| Attention (`indexed_decode`, `qk_lowrank`, `kda_decode`) | 2,2 ms/Token | 3,4 % | — |
| `kernel_glm_router_select_one` | 0,7 ms/Token | 1,1 % | 42 |

Die GPU-Zeit wird also zu ~60 % von den **residenten bf16-Gewichten** verbraucht (0,91 ms je
Layer×Token), nicht von den SSD-gestreamten Q4-Experten (0,29 ms je Layer×Token). Der Timeline-Summenwert
(64 ms/Token) liegt über der CB-Union (49,6 ms); Ursache ist die andere Messquelle
(Stage-Boundary-Counter-Samples inkl. Intra-Encoder-Lücken, Lauf B war zudem 2,6 % langsamer durch
das Sampling) — die **Anteile** sind belastbar, die Absolutwerte sind es nicht.

### 3c. I/O-Profil (Trace Lauf C)

| Größe | Decode | Gesamt (Prefill+Decode) |
|---|---|---|
| Reads | 15.095 | 29.643 |
| Bytes | 66,34 GiB (**0,52 GiB/Token**) | 130,27 GiB |
| Zeitraum | 20,93 s | 36,65 s |
| Device belegt (Union der Reads) | 9,11 s (**43,5 %**) | 21,3 s (58 %) |
| Zeit mit **0** Reads in flight | 11,82 s (56,5 %) | 15,4 s |
| Mittlere Nebenläufigkeit | 1,39 (in belegten Fenstern 3,2), Max 9 | 3,74 |
| Aggregierter Durchsatz | 3,17 GiB/s | 3,55 GiB/s |
| Read-Dauer p50 / p90 | 3,78 ms / 8,91 ms bei 4,50 MiB (≈1,19 GiB/s je Read) | — |

Gegenprobe Rohbandbreite (nur lesend, F_NOCACHE, frische Regionen): **1 Stream 5,47/5,72 GiB/s**;
4 parallele Streams 11,1 GiB/s (stark streuend, Page-Cache-verfälscht — nur der 1-Stream-Wert ist
stabil). Der Decode liest also mit ~3,2 GiB/s aggregiert, während das Gerät bei mehr parallelen
Streams deutlich mehr liefern könnte — die Nebenläufigkeit ist aber **strukturell begrenzt**: pro
Layer fehlen im Mittel nur ~1,9 Experten (≈5,7 Reads à 4,5 MiB), mehr lässt sich nicht gleichzeitig
anfordern, und die Experten des **nächsten** Layers sind erst nach dessen Router bekannt.

### 3d. Was das für das Ziel ≥ 10 t/s bedeutet

- Lesevolumen: 0,52 GiB/Token (gemessen, nicht modelliert) — das ist die harte Untergrenze an Bytes
  pro Token.
- **Gemessene Geräteauslastung im Decode: 43,5 %** (56,5 % der Zeit kein Read in flight), aggregiert
  3,17 GiB/s. Die vorhandene Ladepfad-Nebenläufigkeit (Pool, 9 Threads) wird nicht ausgeschöpft, weil
  pro Layer nur ~1,9 Experten fehlen (≈5,7 Reads à 4,5 MiB) und die Experten des **nächsten** Layers
  erst nach dessen Router bekannt sind.
- Der I/O-Anteil der Token-Zeit ist ~71 ms (48,4 ms Warten im Ladepfad + ~23 ms RDADVISE/Early-Load).
  Bei 100 ms/Token müsste dieser Anteil **vollständig** mit GPU-Arbeit (49,6 ms) und Host-Arbeit
  (~40 ms) überlappt werden. Die Layer-Kette lässt das nicht zu: Router n+1 benötigt die MoE-Ausgabe n,
  also gibt es kein Fenster, in dem unabhängig Arbeit nachgeschoben werden könnte.
- Realistische Obergrenze des Pfads: Die Summe der *nicht* überlappbaren Posten (GPU 49,6 +
  I/O-Warte 48,4 + Sync-Latenz ~15) liegt bereits bei ~113 ms/Token (8,8 t/s), bevor Host-Arbeit
  hinzukommt — und die I/O-Warte ist nur durch Spekulation entfernbar, deren Kosten (Vorabladen
  13,5 MiB je Experte oder 0,195 ms je `F_RDADVISE`-Range) den Gewinn übersteigen.
- **Fazit: ≥ 10 t/s ist auf dieser Hardware/Konfiguration mit den Kandidaten aus §6 nicht erreichbar.**
  Der Hebel liegt nicht im Pfad, sondern im Lesevolumen je Token (Hit-Rate, Quantisierung, Modellwahl).

## 4. Diffs und Build

Eine Codeänderung, ein Commit: `73be92c` (`ds4_metal.m`, +Instrumentierung, env-gated, verhaltensneutral).

1. **GPU-/Host-Zeitmessung** — vor `ds4_gpu_wait_command_buffer` (jetzt `ds4_metal.m:1354 ff.`):
   `ds4_gpu_phase0_note_command_buffer()` addiert `(GPUEndTime−GPUStartTime)` als Union und die
   Wandzeit des `waitUntilCompleted`; Aufruf direkt nach dem Wait:
   ```c
   static int ds4_gpu_wait_command_buffer(id<MTLCommandBuffer> cb, const char *label) {
       const int phase0 = ds4_gpu_phase0_enabled();
       const double phase0_t0 = phase0 ? ds4_gpu_now_ms() : 0.0;
       [cb waitUntilCompleted];
       if (phase0) {
           ds4_gpu_phase0_note_command_buffer(cb, ds4_gpu_now_ms() - phase0_t0);
       }
   ```
2. **I/O-Intervalltrace** — in `ds4_gpu_stream_expert_pread_into()` (Aufzeichnung um den pread-Block,
   Dump als CSV am Ende des Decodes) sowie `ds4_gpu_phase0_io_record/_dump`.
3. **Host-Kette im GLM-Routed-MoE** (`ds4_gpu_glm_routed_moe_one_tensor`): Timer um den
   Adress-/Ladeblock, Segmentierung Peek/Load/Slot/Tail:
   ```c
   if (glm_p0) glm_p0_peek_end = ds4_gpu_now_ms();   /* nach dem Peek-Loop */
   ...
   if (glm_p0) glm_p0_load_end = ds4_gpu_now_ms();   /* nach load_selected_missing */
   ...
   if (glm_p0) glm_p0_slot_end = ds4_gpu_now_ms();   /* nach set_addr_slot/addr_buffers */
   ```
4. **Ausgabe** in `ds4_gpu_stream_expert_timing_print()` (erscheint damit an denselben
   Snapshot-Punkten wie die vorhandene Timing-Summary, Prefill/Decode getrennt als total/delta).

Build: `make -j8` → keine Fehler/Warnungen; `make -n` → `make: Nothing to be done for 'all'.`
Binaries: `ds4` 14:42, `ds4-server` 14:42, `ds4-bench` 14:42, danach Rebuild 14:50 (Feinzerlegung).
Alle Läufe mit der instrumentierten Binary reproduzieren das byte-identische Ergebnis der Referenz.

## 5. Abweichungen von diesem Prompt + Begründung

**Abweichung 1 — kein Engine-Kandidat implementiert (Phase 1 übersprungen, Phase 2 nicht begonnen).**
Begründung: §6/MESSPLAN-POTENZIALE verlangen ausdrücklich „**erst Machbarkeitsmessung, dann Code**".
Diese Machbarkeitsmessung ist Phase 0 und fällt eindeutig aus:

| Kandidat | Zielgröße laut Prompt | gemessen | Bewertung |
|---|---|---|---|
| **A** GPU-seitige Adressauflösung (Compact/Validate, Vorbild `ds4_gpu_routed_moe_one_tensor`) | −25 … −45 ms/Token (bind 25 ms + missing_bind 20 ms + Peek-Loop) | Peek 0,13 ms + Slot-Install 0,36 ms = **0,49 ms/Token**; die im Prompt als „bind" geführten **25,2 ms/Token sind der Early-Load-/RDADVISE-Aufbau**, nicht die Adressauflösung; `missing_bind` (6,13 ms je Split-Layer) ist **I/O-Wartezeit** im Deferred-Split, keine Adressrechnung | **nicht tragfähig** (Ceiling 0,3 % der Token-Zeit) |
| **B** L+1/L+2-Router-Lookahead | bis −20 ms/Token | Experten des Layers n+1 sind erst nach dessen Router bekannt; Spekulation kostet reales Lesevolumen (0,52 GiB/Token sind bereits 30 % der Token-Zeit) oder – über `F_RDADVISE` – 0,195 ms je Range × 3 Tensoren (≈25 ms/Token nur für Vorabanforderungen) | **negativer Erwartungswert**, kein Code |
| **C** Sync-/Dispatch-Punkte reduzieren | 5–10 ms/Token | gemessene reine Warte-/Latenzreserve = Host-Warte − GPU = **~15 ms/Token** über 70,7 CB/Token (0,21 ms je CB); mehr als die Hälfte davon ist GPU-Arbeit der gewarteten CB | Ceiling ~15 ms/Token ⇒ 6,8 t/s, **Gate ≥ 8,2 t/s nicht erreichbar** |

Zusätzlich empirisch gestützt: das Muster aus §4 („entfernter Posten taucht anderswo wieder auf")
ist auch für die neu instrumentierten Größen bestätigt — die Probe-Matrix (§2b) zeigt für vier
verschiedene Eingriffe in den Lade-/Nebenläufigkeitspfad Abweichungen von ±3 %.

**Abweichung 2 — Encoder-Timeline nur als Relativmessung.** Der Timeline-Lauf ist 2,6 % langsamer
(Sampling-Overhead); Absolutwerte werden deshalb nicht gegen das 162-ms-Budget gerechnet.

**Abweichung 3 — Probe-Matrix mit n=1 je Konfiguration.** Für die Vorprüfung ausreichend, weil die
Baseline über drei Läufe auf ±0,3 % reproduzierbar war; auf eine n=2-Serie wurde verzichtet, weil
alle Effekte ≤3 % (also ≤ 0,2 t/s) und damit unterhalb jeder Relevanzschwelle lagen.

**Nicht abgewichen:** Messprotokoll §9, 128-Token-Falle (§8 Regel 4, je Lauf `cache aging tokens=127`
und `selected_calls=5334=42×127`), Byte-Gate `cmp` gegen `/tmp/ds4-g0-q4-kalt.out` (584 B) in **allen**
11 Läufen PASS, eine Variable je Lauf, kein `--mtp`, Logs nach `/tmp/ds4-*.err`.

## 6. Offene Punkte und empfohlene nächste Schritte

1. **Ziel nicht erreicht: 6,22 t/s (unverändert), kein Kandidat bestanden.** Die Arbeit endet nach
   Phase 0 mit negativer Machbarkeitsprüfung; Phase 1/2 wurden deshalb nicht umgesetzt (nicht: nicht
   geschafft — bewusst verworfen, siehe §5).
2. **Der einzige gemessen große Hebel liegt außerhalb des Pfads**: das **Lesevolumen je Token**.
   0,52 GiB/Token bei hit_rate 0,827. Optionen mit belastbarer Datenlage: GLM-Q2 resident 17,96 t/s
   (MESSREIHE §5), DeepSeek-MXFP4-Streaming 15,56 t/s; höhere Cache-Budgets sind bereits widerlegt
   (48/56/85 GB schlechter).
3. **Upstream-Einordnung:** Der Issue-Entwurf (`docs-kernel/ISSUE-ENTWURF.md`, nicht gepostet) bleibt
   inhaltlich korrekt. Zu ergänzen wären die neuen Zahlen aus §3: GPU-Anteil **31 %** (nicht 15 %),
   Adressauflösung **0,49 ms/Token**, I/O **0,52 GiB/Token** bei 43 % Geräteauslastung,
   Nebenläufigkeitsunempfindlichkeit des Ladepfads. Damit wäre die Antwort auf die Frage im Entwurf
   („Is the intended direction for GLM to move this per-layer bookkeeping to the GPU?") belegbar:
   **nein — der Adresspfad ist nicht der Engpass.**
4. **Nicht geprüft (bewusst offen, außerhalb des Auftrags):** Vision-Gegentest und Q2-resident-
   Regressionstest aus Phase 3 — sie sind erst nach einem bestandenen Phase-1-Gate sinnvoll; ohne
   Engine-Änderung gibt es kein Regressionsrisiko (byte-identisch in allen 11 Läufen belegt).
5. **Aufräumen:** Worktree enthält ausschließlich `73be92c` (Instrumentierung) sowie die
   untracked Dateien `docs-kernel/` und `FORTSCHRITT.md`/`REPORT.md`. Produktions-Klon
   `/Users/michaelkuebbeler/ai-workspace/localAI/ds4` unverändert (kein Commit, kein Push,
   Port 8008 frei).
