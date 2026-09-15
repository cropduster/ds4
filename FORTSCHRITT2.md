# FORTSCHRITT 2 — GLM-5.3-Flash-Q4_K SSD-Streaming: pread-Bündelung (Teil 2) und MTP-Union (Teil 1)

Worktree: `/Users/michaelkuebbeler/ai-workspace/localAI/ds4-kernel-async`
Branch: `exp/kernel-glm-async-selected` (Basis `df9f2cf`)
Backup vor Teil 2 (§8 Regel 2): Branch `backup-before-pread2-20260915-181058`,
Binaries `/tmp/ds4{,-server,-bench}.bin.prea2-df9f2cf`, SHA-256 in `/tmp/prea2-backup-sha.txt`
Start: 15.09.2026 18:04 · Stand: 15.09.2026 18:35

Alle CLI-Läufe: `--ssd-streaming --ctx 4096 --tokens 128 --temp 0`, Prompt = 128 echte Tokens,
Port 8008 vor jedem Lauf frei, strikt sequenziell (kein Server gestartet/gestoppt).

## Log

| Zeit | Schritt | Befehl/Datei | Ergebnis |
|---|---|---|---|
| 18:04 | Auftrag + Regeln gelesen (`START-AUFTRAG-2.md`, `KERNEL-AGENT-PROMPT.md`, `REPORT.md`, `FORTSCHRITT.md`) | - | Auftrag erfasst |
| 18:06 | Issue `antirez/ds4#495` gelesen (Body + 7 Kommentare) | `gh issue view 495` | Guard betrifft nur `--mtp-model`; Community-Befunde: Streaming-Layer batchfähig, Verify prefill-förmig, netto dennoch langsam |
| 18:08 | Code-Anker Teil 2 | `ds4_metal.m` 13440–13940, 16790–17450 | 3 Read-Tasks je Expert (gate/up/down à 4,5 MiB), Pool mit 9 Threads, `F_RDADVISE` inline **vor** `pool_begin` |
| 18:09 | Phase-0-I/O-Trace neu ausgewertet | `/tmp/ds4-kernel-io.txt` (Lauf C) | Decode: 14.718 Reads, 64,68 GiB, 20,23 s → 3,20 GiB/s; 43,3 % busy; **mittlere Nebenläufigkeit im Busy-Fenster 2,98** (= 3 Tensoren), p50 1,34 ms/4,5 MiB |
| 18:10 | Gerätebenchmark (neu, read-only) | `/tmp/io-bench3.py` (F_NOCACHE, F_RDAHEAD=0, frische Regionen) | 4,5 MiB: QD1 4,92 / QD2 5,12 / QD3 4,68 / QD6 4,76 / QD9 4,71 GiB/s → **Sättigung ab QD≈1–2**; 1 MiB QD1 3,30; 13,5 MiB QD1 5,50; sequenziell 7,63 GiB/s |
| 18:11 | Backup §8 Regel 2 | `git branch backup-…`, `cp ds4*` | PASS |
| 18:13 | Teil-2-Versuch V1 implementiert: Read-ahead als Pool-Tasks (früherer Dispatch, kein serielles `fcntl` auf dem Encode-Thread) | `patch ds4_metal.m`, `make -j8` | Build ok, Commit `f60ab6d` |
| 18:14 | **A/B-Lauf B1** (Original-Binary) | `/tmp/ab-run.sh` | **6,24 t/s**, tokens=127, 584 B **PASS**, readahead_avg 0,192 ms, bind_total 3.152 ms |
| 18:15 | **A/B-Lauf B2** (Original-Binary) | dito | **6,24 t/s**, Gates PASS |
| 18:16 | **A/B-Lauf N1** (V1) | `/tmp/ab-run.sh ./ds4 N1` | **6,08 t/s**, Gates PASS, bind_total 372 ms (−88 % Host-Zeit), aber readahead_avg 0,724 ms, missing_bind 3.439 ms, load_pread_avg 3,87 ms |
| 18:17 | **A/B-Lauf N2** (V1) | dito | **6,10 t/s** → V1 = −2,4 % |
| 18:18 | V1 verworfen, Revert | `git revert f60ab6d` | Commit `3a9f312` |
| 18:20 | Teil-1-Diagnostik: routed-MoE-Pfadzähler (env-gated) | `patch ds4_metal.m` | Commit `84003bc` |
| 18:22 | **Lauf M0** (Revert + Diagnostik, ohne `--mtp`) | `DS4_METAL_MOE_PATH_PROFILE=1` | **6,25 t/s**, Gates PASS, decode `one_calls=5334` (=42×127), prefill `union_calls=42`, 4.969 unique/Layer-Pass |
| 18:24 | **Lauf MTP1** (`--mtp --mtp-timing`) | dito + `--mtp` | **5,10 t/s**, Gates PASS, 74 Zyklen, Accept 52/74, verify2 283 ms, head+draft 39 ms |
| 18:26 | Diagnostik-Ausgabe um `atexit`-Dump erweitert (MTP-Loop erreicht die Memory-Report-Snapshots nicht) | `patch`, `make -j8` | Commit `5e8e350` |
| 18:28 | **Lauf MTP2** (`--mtp`) | `DS4_METAL_MOE_PATH_PROFILE=1` | **5,12 t/s**, Gates PASS: `batch_calls=3150`, `batch_tokens=8022`, `union_calls=3150`, `unique_per_union=14,54` |
| 18:32 | **Lauf F1** (Endstand, ohne Profil-Env) | `/tmp/ab-run2.sh ./ds4 F1` | **6,25 t/s**, Gates PASS → Diagnostik verhaltensneutral, Revert vollständig |
| 18:35 | `REPORT2.md` + dieser Stand | - | Bericht |

## Kernbefunde (Kurzform)

1. **Das 13,5-MiB-Expert-Slab ist keine zusammenhängende Einheit**, sondern drei Tensoren
   (gate/up/down à 4,5 MiB) an drei Dateioffsets. Die „~3 Teilreads" sind daher strukturell korrekt
   und laufen **bereits parallel** (mittlere Nebenläufigkeit im Busy-Fenster 2,98 bei 9 Pool-Threads).
   Die Kennzahl „1,39" aus Phase 0 ist ein **Duty-Cycle-Mittel** (Σ Read-Dauer / Wandzeit), kein
   Parallelitätsmangel.
2. **Das Gerät sättigt bei QD≈1–2** (4,92 GiB/s bei 4,5-MiB-Reads, flach bis QD 9, Latenz linear).
   Mehr Nebenläufigkeit kann den Durchsatz nicht erhöhen; größere Requests sind strukturell
   unmöglich (nicht zusammenhängende Regionen).
3. **V1 (Read-ahead parallel/früher) verschlechtert um 2,4 %** — obwohl es 22 ms/Token Host-Zeit aus
   dem kritischen Pfad entfernt. Der Nutzen des `F_RDADVISE` liegt im zeitlichen Vorlauf vor den
   preads, nicht in seiner Parallelität. Zurückgenommen.
4. **Die Experten-Union über Draft+Verify-Tokens wird bereits gemeinsam bedient** (ein Layer-Pass,
   ein deduplizierter Ladevorgang, 13,14 unique von 16 Slots je Layer). Kein Implementierungsbedarf.
5. **MTP kostet 15 %**, weil die Union je Zeile nur 0,87× kostet, die Accept-Rate (0,70 → 1,73 Token
   je Zyklus) diese Ersparnis exakt aufzehrt und die head+draft-Phase 22 ms/Token (12 %) addiert.
