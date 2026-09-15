# FORTSCHRITT Kernel-Auftrag (GLM-5.3-Q4 SSD-Streaming-Decode)

Worktree: /Users/michaelkuebbeler/ai-workspace/localAI/ds4-kernel-async
Branch: exp/kernel-glm-async-selected (Basis main 7a8ca68)
Backup-Branch: backup-before-kernel-async-20260915-143809
Binary-Backups: /tmp/{ds4,ds4-server,ds4-bench}.bin.orig-7a8ca68-kernel (SHA-256: /tmp/kernel-backup-sha.txt)
Gestartet: 15.09.2026 14:26 · Stand: 15.09.2026 15:05

## Log
| Zeit | Schritt | Befehl/Datei | Ergebnis |
|---|---|---|---|
| 14:26 | Worktree + Kontextdateien (START-AUFTRAG, KERNEL-AGENT-PROMPT, MESSREIHE, MESSPLAN-POTENZIALE, ROLLBACK, ISSUE-ENTWURF) | - | Auftrag und Hard Rules erfasst |
| 14:38 | Umfeldcheck | `lsof -nP -iTCP:8008`, `df`, `diskutil info` | Port 8008 frei, RAM 128 GiB (95 % frei), Modell 190.875.526.464 B auf /dev/disk7s1 (APFS, Soild State, PCI-Express) |
| 14:38 | Backup §8 Regel 2 | `git branch backup-before-kernel-async-*`, `cp ds4{,-server,-bench}` | PASS (Branch + 3 Binaries + SHA-256) |
| 14:39 | SSD-Rohbandbreite, nur lesend, F_NOCACHE, disjunkte frische Regionen | `/tmp/ssd-bw3.py` | 1 Stream **5,72 GiB/s**, 2 Streams 15,9, 4 Streams 11,1, 8 Streams 21,0 GiB/s (streuen stark, Page-Cache-Effekte; belastbar ist nur der 1-Stream-Wert: 5,47/5,72 GiB/s in zwei Messungen) |
| 14:40 | **Baseline-Reproduktion** im Worktree, Original-Binary | §9-Protokoll + `DS4_METAL_GPU_BUSY_PROFILE=1` | 6,41 t/s, `tokens=127`, 584 B **byte-identisch** |
| 14:42 | Phase-0-Instrumentierung (`ds4_metal.m`, env-gated) | `patch`, `make -j8` | Build ok, verhaltensneutral |
| 14:45 | **Lauf A** Phase 0 | §9 + `DS4_METAL_PHASE0_PROFILE=1` | 6,20 t/s, Gates PASS |
| 14:45 | **Lauf B** Phase 0 + Encoder-Timeline | + `DS4_METAL_ENCODER_TIMELINE` | 6,04 t/s (−2,6 % Sampling-Overhead), 164.403 Timeline-Zeilen |
| 14:47 | I/O-Intervalltrace ergänzt | `patch`, `make -j8` | Build ok |
| 14:47 | **Lauf C** Phase 0 + I/O-Trace | + `DS4_METAL_PHASE0_IO_TRACE` | 6,22 t/s, Gates PASS, 29.643 Reads |
| 14:50 | Feinzerlegung der MoE-Host-Kette | `patch`, `make -j8` | Build ok |
| 14:51 | **Lauf D** Feinzerlegung | §9 + Phase 0 | 6,21 t/s, Gates PASS, 584 B identisch |
| 14:52 | Commit Phase-0-Instrumentierung | `git commit` | 73be92c |
| 14:53 | **Probe-Matrix** Phase 1 (nicht in §4 ausgeschlossene I/O-Knöpfe) | `/tmp/probe-io.sh` | alle 584 B PASS, Raten siehe REPORT.md §2b |
| 15:05 | I/O-/Kostenanalyse + Kandidatenbewertung | `/tmp/io-analyze.py`, `/tmp/tl-analyze.py` | siehe REPORT.md |

## Phase-0-Kernzahlen (Decode, 127 Token, je Lauf byte-identisch zu 584 B)

| Posten | ms/Token | Anteil | Quelle |
|---|---|---|---|
| Gesamt (Wand) | 162,0–162,6 | 100 % | `wall_since_prev_print` / t-s |
| GPU-Ausführung (Union der CB-Timestamps) | 49,5–50,4 | ~31 % | `gpu_busy_union_delta` |
| Host-Warte in `waitUntilCompleted` | 64,6–65,4 | ~40 % | `host_wait_delta` |
| davon reine Latenz (Warte − GPU) | ~15 | ~9 % | Differenz |
| Host: Warten auf fehlende Experten (SSD-pread) | 48,4 | 30 % | `moe_load_ms` |
| Host: Early-Load/RDADVISE-Aufbau („bind") | 25,2 | 16 % | `bind_total` |
| Host: Cache-Peek | 0,13 | 0,1 % | `moe_peek_ms` |
| Host: Adress-Slot-Install | 0,36 | 0,2 % | `moe_slot_ms` |
| Host: MoE-Encode/Tail im Adressblock | 0,002 | ~0 % | `moe_tail_ms` |

Reproduzierbarkeit A↔C↔D: Wand ±0,3 %, GPU-Busy ±1,8 %, Host-Warte ±1,1 % → **±5 %-Gate PASS**.
Command Buffer im Decode: 8982 → **70,7 CB/Token** (≈1,7 je Layer); `gpu_busy_union == gpu_span_sum`
⇒ keine überlappenden Command Buffer, GPU-Arbeit vollständig serialisiert.

## I/O-Trace (Lauf C)
- Decode: 15.095 Reads, 66,34 GiB (0,52 GiB/Token) in 20,93 s → **3,17 GiB/s** aggregiert.
- Device nur **43,5 % der Decode-Zeit belegt** (56,5 % ohne aktiven Read); Nebenläufigkeit im Mittel 1,39
  (in den belegten Fenstern 3,2), Max 9.
- Read-Dauer p50 3,78 ms bei 4,50 MiB → **~1,19 GiB/s je Read**.

## Offen
- REPORT.md
- Kein Engine-Kandidat implementiert — Begründung in REPORT.md §5 (Machbarkeitsmessung nach §6/MESSPLAN:
  „erst Machbarkeitsmessung, dann Code"; sie fällt negativ aus).
- Phase 3 (Vision-/Q2-Regressionsgegenprobe 32K-Kontext) nicht erreicht, da Phase 1 nicht bestanden.
