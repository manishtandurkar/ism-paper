# StorageLab: Technical Details for an IEEE Access Paper

This document collects everything about StorageLab that a paper needs: what the system is, how it is built, the exact models and algorithms, the correctness argument for the RAID engine, measured results, and limitations. Every formula and number here was taken from the code (commit 76ffc3a) or measured with `paper/measurements.py`. Nothing is estimated from memory.

Repository: `../Information-Storage-Management-EL/` (all paths below are relative to it). Source of truth: `modules/` (calculations), `backend/app/` (API), `frontend/src/` (UI), `tests/` and `backend/tests/` (validation).

---

## 1. One-paragraph summary

StorageLab is a hardware-independent, web-based laboratory for teaching Information Storage Management. It combines five analytical models (RAID capacity and failure states, single-disk M/M/1 performance, backup strategy storage and RPO/RTO compliance, synchronous and asynchronous replication, and likelihood-impact risk) with a block-level RAID engine. The engine stripes a real user file across one file per simulated disk using RAID 0, 1, 5, 6 or 10. RAID 5 uses rotating XOR parity. RAID 6 adds a Reed-Solomon Q syndrome over GF(2^8). Disk failure is modelled by deleting a disk's file. The engine then reads in degraded mode, rebuilds onto a hot spare with streamed progress, and proves recovery end to end by comparing the SHA-256 of the reassembled file with the original. Every result carries its units, stated assumptions, and a rule-based plain-language explanation generated from the same formulas. An exhaustive fault-injection campaign over 1,058 failure combinations across 14 array configurations found zero silent corruptions. The engine's recoverability predicate agreed with the observed outcome in every case.

---

## 2. Problem and motivation (paper Section I)

- Storage concepts such as parity, degraded mode, rebuild, queueing delay, RPO/RTO and replication lag are taught with static diagrams. Students rarely see them act on real data because enterprise arrays, SAN fabrics and multi-site links are expensive and unavailable in a classroom.
- Production tools such as Linux md and hardware controllers are not built for teaching. They hide the block layout, need root access or real block devices, and give no explanation of why a state was reached.
- Calculators (capacity or IOPS calculators) give a number with no derivation and no way to observe failure.
- Gap: no single, lightweight tool lets a learner (a) compute textbook metrics with visible formulas and (b) see the same concepts execute on real bytes, with a checkable correctness proof (a hash match), on a laptop and without special privileges.

Research questions the paper can frame:
1. Can a pure-software, file-backed RAID engine demonstrate RAID 0/1/5/6/10 failure and recovery faithfully enough to be verifiable, meaning byte-exact recovery within tolerance and correctly reported data loss beyond it?
2. Can analytical storage models be presented so that every output is traceable to a formula, its units, and its assumptions?
3. Is the system light enough to run interactively on commodity hardware?

---

## 3. Contributions (candidate list)

1. A block-level, file-backed RAID engine (`modules/raid_engine.py`, about 440 lines) implementing RAID 0, 1, 5 (rotating XOR parity), 6 (P + Reed-Solomon Q over GF(2^8), polynomial 0x11D, generator 2) and 10. It supports degraded reads, hot-spare rebuild streamed as NDJSON, and SHA-256 end-to-end verification.
2. A closed-form treatment of every RAID 6 two-erasure case (data+data, data+P, data+Q, P+Q). All cases are checked by exhaustive fault injection.
3. An integrated set of textbook storage models (M/M/1 disk, backup storage and RPO/RTO, replication latency and lag, likelihood-impact risk). Each is exposed as a REST endpoint returning values, units, assumptions and a deterministic explanation.
4. A layered architecture (models, thin API, presentation-only UI) in which the same calculation functions drive two front ends (React and Streamlit) and a test suite.
5. A validation methodology: exhaustive enumeration of failure subsets, a recoverability predicate checked against observed behaviour, and hand-calculated formula tests. The result was 1,058/1,058 agreement and zero silent mismatches.

Wording note for the paper: the UI labels the explanation panels "Explainable AI". They are deterministic, rule-based templates, not machine learning. In an IEEE paper, call them a "rule-based explanation layer" or "formula-grounded explanations". Do not claim AI or ML, or reviewers will object.

---

## 4. System architecture (paper Section III)

### 4.1 Layers

```
Browser (React 19 + TypeScript, Vite, Recharts)
   |  JSON over HTTP (REST), NDJSON stream for rebuild
FastAPI routers  backend/app/routers/*.py     (Pydantic validation, units, assumptions, explanation)
   |  plain Python calls
Calculation modules  modules/*.py              (all formulas; no web or UI imports at module level)
   |                         |
SQLite via SQLAlchemy      workspace/<array id>/  disk_0.bin ... disk_{n-1}.bin, spare_k.bin, meta.json
(saved scenarios)          (live RAID engine)

Alternate front end: Streamlit app.py -> the same modules (show_*() functions import Streamlit lazily)
```

Design rules enforced in the code:
- Every formula lives in `modules/`. Routers only validate, call, and annotate. React pages contain no formulas.
- Each module's Streamlit renderer (`show_raid`, `show_disk`, ...) imports Streamlit inside the function. The calculation functions can therefore be imported and tested without any UI dependency.
- Validation runs twice: Pydantic bounds at the API edge (`backend/app/schemas.py`) and semantic checks in the module (minimum disks, even counts, utilization below 1). A module `ValueError` is mapped to HTTP 422 by a global exception handler (`backend/app/main.py`).

### 4.2 Technology stack (versions as measured on the evaluation machine)

| Layer | Technology |
|---|---|
| Language (backend) | Python 3.15.2 |
| Numerics | NumPy 2.5.5, pandas 3.1.1 |
| API | FastAPI 0.165.0, Pydantic 2.14.4, Uvicorn |
| Persistence | SQLAlchemy 2.1.9, SQLite (PostgreSQL possible through `STORAGELAB_DATABASE_URL`) |
| Front end | React 19.3, TypeScript 7.0, Vite 8.3, Recharts 3.11, React Router 7, TanStack Query 5, lucide-react |
| Alternate UI | Streamlit 1.67.0, Plotly |
| Testing | pytest 8+, httpx (API tests), Vitest 5, Testing Library, jsdom |
| Deployment | Docker (python:3.12-slim API image, nginx front-end image), docker-compose with a health check and a named SQLite volume |

Code size (tracked files): Python 2,677 lines, TypeScript/TSX 2,242 lines, CSS 1,887 lines.

### 4.3 REST API

| Method and path | Purpose |
|---|---|
| GET /api/health | Liveness |
| POST /api/raid/calculate | Capacity, efficiency, fault tolerance, write penalty, illustrative layout |
| POST /api/raid/simulate-failure | State after a set of failed disks, accessibility, remaining tolerance, rebuild time, cross-level comparison |
| POST /api/raid/xor | Interactive XOR parity recovery of one of three binary blocks |
| POST /api/raid/array | Create a live array from a base64 file (level, disks, stripe unit, spares) |
| GET /api/raid/array/{id} | Layout with a real byte preview per block, state, events |
| POST /api/raid/array/{id}/fail/{disk} | Delete a disk file (fault injection) |
| GET /api/raid/array/{id}/verify | Read (degraded if needed) and compare SHA-256 |
| POST /api/raid/array/{id}/rebuild/{disk} | Rebuild onto a hot spare; NDJSON progress stream |
| DELETE /api/raid/array/{id} | Remove the array directory |
| POST /api/disk/analyze | M/M/1 disk metrics and a 91-point response curve |
| POST /api/backup/compare | Full, incremental and differential comparison and a recommendation |
| POST /api/replication/compare | Selected mode and a side-by-side synchronous/asynchronous comparison |
| POST /api/risk/calculate | Risk score, category and mitigation |
| /api/scenarios (CRUD), /{id}/clone, /compare, /export/csv | Saved experiments in SQLite |

Every analytical response has the shape `{ ...values, units: {...}, assumptions: [...], explanation: {summary, formula, contributing_factors[], assumptions[]} }`.

### 4.4 Persistence model

Table `scenarios`: `id`, `name` (≤200), `module_type` (≤50), `inputs` (JSON), `results` (JSON), `assumptions` (JSON list), `tags` (JSON list), `created_at`, `updated_at` (UTC). Scenario comparison accepts 2 to 3 unique ids. CSV export streams all rows.

### 4.5 Front end

Routes: `/` overview, `/raid` (two tabs: capacity calculator and live array), `/disk`, `/backup`, `/replication`, `/risk`, `/scenarios`, `/methodology`. Charts: RAID efficiency bar chart, disk response time against utilization with a reference line at the 70% knee and a dot at the current operating point. Each module has a collapsible quick-start guide and an explanation panel. Light and dark themes. The layout is checked at 390 px phone width with no horizontal scrolling.

The live array panel reads the rebuild response with `ReadableStream.getReader()`, splits NDJSON lines, and updates a progress bar. The final line carries the new array layout.

---

## 5. Block-level RAID engine (paper Section IV, the core technical contribution)

File: `modules/raid_engine.py`.

### 5.1 Configuration space and limits

| Parameter | Allowed values |
|---|---|
| Levels | RAID 0, 1, 5, 6, 10 |
| Disks n | RAID 0 ≥ 2, RAID 1 = 2 exactly, RAID 5 ≥ 3, RAID 6 ≥ 4, RAID 10 ≥ 4 and even; all ≤ 8 |
| Stripe unit b | 512, 1024, 4096, 16384, 65536 bytes |
| Hot spares | 0 to 2 |
| File size | 1 byte to 5 MiB |

### 5.2 On-disk representation

- Each array gets a 12-hex-character id (`uuid4().hex[:12]`) and a directory `workspace/<id>/`.
- `disk_i.bin` holds `rows × b` bytes: the concatenation of the disk's blocks, row by row.
- `spare_k.bin` is pre-allocated as zero bytes of the same size.
- `meta.json` holds the level, n, b, rows, original size, the original SHA-256, spares total and used, and an event log.
- Failure state is not stored. It is derived: a disk has failed if and only if its file does not exist. The physical state is therefore the single source of truth.
- Path safety: ids must fully match `[0-9a-f]{12}` and the directory must contain `meta.json`. Anything else returns 404, which prevents path traversal.

### 5.3 Striping

Let k be the number of data blocks per row: RAID 0 k = n, RAID 1 k = 1, RAID 5 k = n−1, RAID 6 k = n−2, RAID 10 k = n/2.

rows = ⌈size / (k·b)⌉. The file is zero-padded to rows·k·b, reshaped into rows·k blocks of b bytes (a NumPy `uint8` view), and each row's k blocks are encoded into n blocks.

Role assignment for row r (`row_roles`):
- RAID 0: every disk holds data block index = disk.
- RAID 1 and 10: disk d holds data block ⌊d/2⌋. Even disks are the primary and odd disks the mirror, so pairs are (0,1), (2,3), ...
- RAID 5: P is on disk r mod n.
- RAID 6: P is on disk r mod n and Q on disk (r+1) mod n.
- In RAID 5 and 6 the data indices 0..k−1 fill the remaining disks in ascending disk order.

This is a simple rotating-parity layout. It is not identical to Linux md's left-symmetric layout. Say so in the paper.

### 5.4 Parity mathematics

P (RAID 5 and 6): $P = D_0 \oplus D_1 \oplus \dots \oplus D_{k-1}$, bytewise.

Q (RAID 6), over GF(2^8) with reduction polynomial $x^8+x^4+x^3+x^2+1$ (0x11D) and generator g = 2:

$$Q = \bigoplus_{i=0}^{k-1} g^{i} \cdot D_i$$

Implementation:
- Log and antilog tables are built once: `GF_EXP` has 512 entries (doubled to avoid a modulo), `GF_LOG` has 256. Generation: repeated left shift, XOR with 0x11D on overflow.
- `gf_scale(block, c)` multiplies a whole block by a constant in one vectorised NumPy expression, `EXP[LOG[block] + LOG[c]]`, applied to non-zero bytes only. There is no per-byte Python loop.
- `gf_inverse(v) = EXP[(255 − LOG[v]) mod 255]`.

### 5.5 Reconstruction (`recover_row`)

Given the stored blocks of one row with some missing (None):

1. If the missing set is not recoverable, raise `DataLossError`.
2. Collect the surviving data blocks, P and Q.
3. Solve for the lost data indices:

| Lost in the row | Method |
|---|---|
| One data block, P present | $D_x = P \oplus \bigoplus_{i \ne x} D_i$ |
| One data block, P lost (Q present) | $D_x = g^{-x} \cdot (Q \oplus \bigoplus_{i\ne x} g^i D_i)$ |
| Two data blocks x, y | $P_{xy} = P \oplus \bigoplus_{i\notin\{x,y\}} D_i = D_x \oplus D_y$, $Q_{xy} = Q \oplus \bigoplus_{i\notin\{x,y\}} g^i D_i = g^x D_x \oplus g^y D_y$, so $D_x = (Q_{xy} \oplus g^y P_{xy}) \,/\, (g^x \oplus g^y)$ and $D_y = P_{xy} \oplus D_x$ |
| Only P and/or Q lost | No data is lost; recompute them |
| Mirror (RAID 1 or 10) | Copy from the surviving pair member |

4. Re-encode the full row from the recovered data. The rebuild then takes the block for the target disk.

The denominator $g^x \oplus g^y$ is non-zero for x ≠ y because g = 2 is a primitive element and k ≤ 6 < 255, so the two-erasure system always has a solution.

Because parity rotates, a single failed physical disk holds data in some rows and P or Q in others. The engine therefore solves each row independently. This is why RAID 6 with two failed disks hits all four erasure cases across the rows.

### 5.6 Recoverability predicate

```
is_recoverable(level, n, M):
  RAID 0      -> M is empty
  RAID 1, 10  -> no mirror pair (2j, 2j+1) is fully contained in M
  RAID 5      -> |M| <= 1
  RAID 6      -> |M| <= 2
```
The array state is Healthy if M is empty, Degraded if M is recoverable, and Failed otherwise. The guaranteed tolerance used in the UI is 0, 1, 1, 2, 1 for RAID 0, 1, 5, 6, 10. RAID 10 can survive up to n/2 failures if each is in a different pair.

### 5.7 Operations

- `create_array`: validate, pad, encode all rows into an (n, rows, b) array, write each disk with `ndarray.tofile`, allocate spares, record the SHA-256.
- `fail_disk`: `unlink` the disk file and append an event.
- `read_file`: check recoverability, load the surviving disks (`np.fromfile`), recover each row, emit the data blocks in index order, and truncate to the original size.
- `verify`: return the state, missing disks, the original and read SHA-256 values, whether they match, data loss, and a message ("Read in degraded mode: Disk 2 reconstructed on the fly").
- `rebuild_disk`: validate eagerly (the disk must be missing, the set recoverable, a spare available), then return a generator. The generator writes the recovered row blocks into the spare file, yields (done, total) after each row, renames the spare to `disk_i.bin` when finished, and marks the spare as used. Because the rename happens only after the last row is written, an interrupted rebuild never presents a half-written disk as healthy.
- API streaming: `progress_steps` throttles to about 50 updates. The router spreads an artificial 2.5 s across the rows (`REBUILD_SECONDS`) so the progress bar is visible. State this in the paper: measured engine times (Section 8.2) exclude this delay.
- `layout`: the role, label (D, D′, P, Q) and first 4 bytes in hex of each block for up to 12 rows. The UI therefore shows real stored bytes, not a mock-up.

---

## 6. Analytical models (paper Section V)

All formulas below are copied from the code. The unit conventions are listed in 6.6.

### 6.1 RAID capacity calculator (`modules/raid.py`)

| Level | Usable | Fault tolerance | Write penalty |
|---|---|---|---|
| RAID 0 | N·S | 0 | 1 |
| RAID 1 | N/2·S (N even) | mirror-dependent | 2 |
| RAID 5 | (N−1)·S, N ≥ 3 | 1 | 4 |
| RAID 6 | (N−2)·S, N ≥ 4 | 2 | 6 |
| RAID 10 | N/2·S, N ≥ 4 even | mirror-dependent | 2 |

- Efficiency = usable / raw × 100. The calculator allows 2 to 32 disks.
- States: Healthy, Degraded, Rebuilding, Failed. Rebuilding applies only when the array is Degraded and a hot spare is enabled. A Failed array is never rebuilt.
- Rebuild time (h) = (S·f in TB × 1024²) MB / throughput (MB/s) / 3600, where f is the number of failed disks and the default throughput is 200 MB/s.
- Remaining tolerance: RAID 5 and 6 subtract the failed count from 1 or 2. RAID 1 and 10 report the number of fully intact mirror pairs.
- XOR demo: P = A⊕B⊕C. A missing block is P ⊕ the survivors, and the API returns whether the recovered value equals the original.

### 6.2 Disk performance (`modules/disk_performance.py`), single-server M/M/1

- Rotational latency $T_{rot} = 30000/\text{RPM}$ ms (half a revolution).
- Transfer $T_{xfer} = (\text{block KB}/1024)/\text{rate MB/s} \times 1000$ ms.
- Service time $T_s = T_{seek} + T_{rot} + T_{xfer}$. Maximum IOPS = $1000/T_s$.
- Utilization $U = \lambda \cdot T_s$ (seconds). The input is rejected if U ≥ 1, and the error message gives the maximum stable arrival rate.
- Response time $R = T_s/(1−U)$. Queue length $L_q = U^2/(1−U)$. Little's Law $N = \lambda R$, which equals $U/(1−U)$.
- Curve: R at 91 utilization points from 5% to 95%. The UI marks the knee at 70% and the current operating point. The Streamlit version warns at U ≥ 80%.

### 6.3 Backup planner (`modules/backup.py`)

Inputs: data size D (GB), daily change c%, retained points R, throughput T (MB/s), RPO, RTO, interval. Let C = D·c/100.

| | Full | Incremental | Differential |
|---|---|---|---|
| Retained storage | R·D | D + (R−1)·C | D + Σ_{j=1}^{R−1} min(jC, D) |
| Data per backup | D | C | min((R−1)C, D) |
| Restore data | D | D + (R−1)C | D + min((R−1)C, D) |
| Restore steps | 1 | R | 2 |

- Duration (h) = GB × 1024 / T / 3600.
- RPO passes if interval ≤ RPO. RTO passes if restore time ≤ RTO.
- Recommendation: among strategies that pass both, choose the one with the least retained storage. If none passes, say so explicitly and suggest increasing frequency or throughput, or revising the objectives.

### 6.4 Replication (`modules/replication.py`)

- Propagation RTT = 2 · distance · 0.005 ms/km (about 5 µs/km in fibre). Total RTT = network RTT + propagation RTT.
- Write bits = KB × 1024 × 8. Transmit time = bits / (Mbps × 10^6) × 1000 ms.
- Link utilization ρ = IOPS × bits / 10^6 / Mbps. If ρ ≥ 1 the link is saturated and the input is rejected.
- Synchronous: response = local write + total RTT, lag = RPO = 0.
- Asynchronous: response = local write, lag = RPO = total RTT / 2 + transmit / (1 − ρ). The second term is the M/M/1 sojourn time on the link. Writes at risk = IOPS × lag.

### 6.5 Risk (`modules/risk_assessment.py`)

Score = likelihood × impact, each an integer from 1 to 5 (booleans are rejected). Bands: 1–5 Low, 6–10 Medium, 11–15 High, 16–25 Critical. Mitigations are mapped per threat (hardware failure, ransomware, unauthorized access), with a generic fallback.

### 6.6 Unit conventions (state these explicitly in the paper)

- Storage sizes are binary: 1 TB = 1024² MB in rebuild time, and 1 GB = 1024 MB in backup.
- The network is decimal: 1 Mbps = 10^6 bit/s, while write size uses 1 KB = 1024 B.
- All times are converted to consistent units before dividing (ms within a disk operation, hours for backup and rebuild).

### 6.7 Explanation layer (`modules/explainability.py`)

Each module returns `{summary, formula, contributing_factors[], assumptions[]}` built with f-strings from the same computed values. The text is therefore always consistent with the numbers. It is deterministic and reproducible, and it is not a learned model.

---

## 7. Validation methodology (paper Section VI-A)

### 7.1 Automated tests

- Python: 62 tests passing (`.venv/bin/python -m pytest -q`, 0.80 s). Frontend: 11 Vitest tests passing. The TypeScript type check and production build succeed.
- Module tests (`tests/`): RAID capacity for every level, invalid configurations, failure states, write penalty, XOR recovery, rebuild time, disk formulas against hand-calculated values, queue length, rejection of unstable utilization, backup storage and RPO/RTO, replication distance and lag units, link saturation, risk bands.
- RAID engine tests (`tests/test_raid_engine.py`):
  - Write then read returns identical bytes for every level (configurations RAID 0×3, 1×2, 5×4, 6×5, 10×4).
  - Every combination of failures up to the level's tolerance (for RAID 6, every pair) is read in degraded mode, rebuilt onto spares, and must give a matching SHA-256 and a Healthy state.
  - Failures beyond tolerance report data loss, and rebuild is refused.
  - RAID 10 survives one failure in each mirror pair.
  - Each RAID 5 parity block equals the XOR of its row's data blocks. Parity rotates.
  - GF inverse and scaling identities. Rebuild requires a spare. Configuration validation. The layout reports real bytes.
- API tests (`backend/tests/`): end-to-end demo flow (create RAID 5, fail, verify, rebuild, verify), RAID 5 versus RAID 6 under two failures, error paths, scenario CRUD, clone, compare, CSV export.

### 7.2 Exhaustive fault-injection campaign (new measurement for the paper)

Method: for each configuration, every non-empty subset of disks (2^n − 1 subsets) was failed on a fresh array holding a 16 KiB random file with a 512 B stripe unit. The SHA-256 after a read was compared with the original. Each outcome was classified as recovered, reported data loss, or silent mismatch (the read succeeded but the bytes differ). The observed outcome was also compared with `is_recoverable`.

| Configuration | Subsets | Recovered | Reported data loss | Silent mismatch | Predicate agreement |
|---|---|---|---|---|---|
| RAID 0, n=2 | 3 | 0 | 3 | 0 | 3/3 |
| RAID 0, n=4 | 15 | 0 | 15 | 0 | 15/15 |
| RAID 1, n=2 | 3 | 2 | 1 | 0 | 3/3 |
| RAID 5, n=3 | 7 | 3 | 4 | 0 | 7/7 |
| RAID 5, n=4 | 15 | 4 | 11 | 0 | 15/15 |
| RAID 5, n=6 | 63 | 6 | 57 | 0 | 63/63 |
| RAID 5, n=8 | 255 | 8 | 247 | 0 | 255/255 |
| RAID 6, n=4 | 15 | 10 | 5 | 0 | 15/15 |
| RAID 6, n=5 | 31 | 15 | 16 | 0 | 31/31 |
| RAID 6, n=6 | 63 | 21 | 42 | 0 | 63/63 |
| RAID 6, n=8 | 255 | 36 | 219 | 0 | 255/255 |
| RAID 10, n=4 | 15 | 8 | 7 | 0 | 15/15 |
| RAID 10, n=6 | 63 | 26 | 37 | 0 | 63/63 |
| RAID 10, n=8 | 255 | 80 | 175 | 0 | 255/255 |
| Total | 1,058 | 219 | 839 | 0 | 1,058/1,058 |

The recovered counts match combinatorics, which makes a good sanity check for the paper:
- RAID 5: n single failures.
- RAID 6: C(n,1) + C(n,2), for example 8 + 28 = 36 at n = 8.
- RAID 10: 3^{n/2} − 1, because each mirror pair can lose nothing, the first member, or the second member. That gives 80 at n = 8.

Conclusion: the engine never returns wrong data. It either recovers byte-exactly or reports data loss.

---

## 8. Results (paper Section VI-B)

Evaluation machine: Apple M5, 16 GB RAM, macOS (Darwin 27.2), Python 3.15.2, NumPy 2.5.5. Each value is the median of 5 runs.

### 8.1 Representative model outputs (case studies)

RAID capacity, 8 × 4 TB disks:

| Level | Usable (TB) | Efficiency (%) | Write penalty |
|---|---|---|---|
| RAID 0 | 32 | 100.0 | 1 |
| RAID 1 | 16 | 50.0 | 2 |
| RAID 5 | 28 | 87.5 | 4 |
| RAID 6 | 24 | 75.0 | 6 |
| RAID 10 | 16 | 50.0 | 2 |

Disk performance, 7200 RPM, 4 ms seek, 150 MB/s, 4 KB block (Ts = 8.193 ms, maximum 122.06 IOPS):

| λ (IOPS) | U (%) | R (ms) | Lq | N |
|---|---|---|---|---|
| 0 | 0.00 | 8.193 | 0.000 | 0.000 |
| 25 | 20.48 | 10.303 | 0.053 | 0.258 |
| 50 | 40.96 | 13.877 | 0.284 | 0.694 |
| 75 | 61.45 | 21.250 | 0.979 | 1.594 |
| 100 | 81.93 | 45.331 | 3.714 | 4.533 |
| 125 | rejected (U = 102.4%) | | | |

The step from 61% to 82% utilization doubles the response time (21.3 ms to 45.3 ms), which illustrates the knee.

Effect of spindle speed at λ = 100 IOPS (same seek, rate and block):

| RPM | Rotational latency (ms) | Ts (ms) | U (%) | R (ms) |
|---|---|---|---|---|
| 5400 | 5.556 | 9.582 | 95.8 | 229.004 |
| 7200 | 4.167 | 8.193 | 81.9 | 45.331 |
| 10000 | 3.000 | 7.026 | 70.3 | 23.625 |
| 15000 | 2.000 | 6.026 | 60.3 | 15.164 |

A 1.6 ms change in service time between 7200 and 5400 RPM produces a 5× change in response time near saturation. This is a strong figure for the paper.

Backup, 1000 GB, 5%/day, 7 retention points, 100 MB/s, daily interval, RPO 24 h, RTO 4 h:

| Strategy | Retained (GB) | Backup (h) | Restore (h) | Restore data (GB) | Steps | RPO | RTO |
|---|---|---|---|---|---|---|---|
| Full | 7000 | 2.844 | 2.844 | 1000 | 1 | Yes | Yes |
| Incremental | 1300 | 0.142 | 3.698 | 1300 | 7 | Yes | Yes |
| Differential | 2050 | 0.853 | 3.698 | 1300 | 2 | Yes | Yes |

The recommendation is Incremental, the lowest storage that meets both objectives. With RPO = 1 h and daily backups, no strategy is compliant and the system says so explicitly.

Replication, local write 2 ms, network RTT 2 ms, 1000 Mbps, 500 IOPS, 8 KB writes (ρ = 3.28%):

| Distance (km) | Synchronous response (ms) | Asynchronous response (ms) | Asynchronous lag / RPO (ms) |
|---|---|---|---|
| 0 | 4.000 | 2.000 | 1.068 |
| 10 | 4.100 | 2.000 | 1.118 |
| 100 | 5.000 | 2.000 | 1.568 |
| 500 | 9.000 | 2.000 | 3.568 |
| 1000 | 14.000 | 2.000 | 6.068 |
| 5000 | 54.000 | 2.000 | 26.068 |

Bandwidth sensitivity at 100 km for asynchronous replication: 1000 Mbps gives a lag of 1.568 ms (ρ 3.3%), 100 Mbps gives 2.475 ms (ρ 32.8%), 50 Mbps gives 5.303 ms (ρ 65.5%), and 40 Mbps gives 10.562 ms (ρ 81.9%). Below about 32.8 Mbps the link saturates and the input is rejected.

### 8.2 Engine performance (4 MiB random file, seconds, excluding the UI's artificial rebuild delay)

| Level, n | Unit | Rows | Encode | Read | Degraded read | Rebuild 1 disk | Read with 2 failed | Raw / file bytes |
|---|---|---|---|---|---|---|---|---|
| RAID 0, 4 | 4 KiB | 256 | 0.003 | 0.001 | – | – | – | 1.000 |
| RAID 0, 4 | 64 KiB | 16 | 0.002 | 0.001 | – | – | – | 1.000 |
| RAID 1, 2 | 4 KiB | 1024 | 0.005 | 0.002 | 0.003 | 0.002 | – | 2.000 |
| RAID 1, 2 | 64 KiB | 64 | 0.004 | 0.001 | 0.001 | 0.001 | – | 2.000 |
| RAID 5, 4 | 4 KiB | 342 | 0.004 | 0.001 | 0.004 | 0.003 | – | 1.336 |
| RAID 5, 4 | 64 KiB | 22 | 0.004 | 0.001 | 0.002 | 0.002 | – | 1.375 |
| RAID 6, 5 | 4 KiB | 342 | 0.019 | 0.001 | 0.017 | 0.017 | 0.025 | 1.670 |
| RAID 6, 5 | 64 KiB | 22 | 0.015 | 0.001 | 0.013 | 0.013 | 0.019 | 1.719 |
| RAID 10, 4 | 4 KiB | 512 | 0.004 | 0.001 | 0.002 | 0.002 | – | 2.000 |
| RAID 10, 4 | 64 KiB | 32 | 0.003 | 0.001 | 0.001 | 0.001 | – | 2.000 |

Observations for the discussion:
- Every operation finishes in at most 25 ms at the 5 MiB upload limit, well below interactive thresholds, on a laptop with no special privileges.
- RAID 6 encoding and recovery take about 4 to 5 times longer than RAID 5 because of the GF(2^8) multiplications (table lookups), against a single XOR for P. This mirrors the real RAID 6 compute overhead and its write penalty of 6.
- The measured raw-to-file ratio is slightly above the theoretical n/k (1.333 for RAID 5 with 4 disks, 1.667 for RAID 6 with 5 disks) because of zero padding in the last row. The padding grows with the stripe unit, which gives a concrete way to discuss internal fragmentation.
- These times are in-memory NumPy timings on page-cached files. They are not device benchmarks, and the paper must say so.

### 8.3 Figures available

Existing screenshots (`screenshots/`): 01 home, 02 RAID calculator, 03 live array (light), 04 disk, 05 backup, 06 replication, 07 risk, 08 saved experiments, 09 methodology, 10 live array (dark), 11 RAID page at phone width.

Suggested figures to plot from the tables above:
1. Architecture diagram (Section 4.1).
2. RAID 5 and RAID 6 rotating layout grid with P/Q positions (from screenshot 03, or redrawn).
3. A sequence diagram of create, fail, degraded read, rebuild (NDJSON), verify.
4. Disk response time against utilization, one curve per RPM.
5. Synchronous against asynchronous response and lag over distance.
6. Backup retained storage against retention points for the three strategies.
7. The fault-injection outcome matrix (the Section 7.2 table as a figure or table).

---

## 9. Demonstration scenario (useful as a use-case subsection)

1. Live array: RAID 5, 4 disks, 4 KiB unit, 1 spare, sample file. P rotates across the disks row by row.
2. Fail Disk 2. The state becomes Degraded and the SHA-256 still matches, because the missing blocks are reconstructed on every read.
3. Rebuild Disk 2. The progress bar streams, the spare becomes Disk 2, the state returns to Healthy, and the SHA-256 matches.
4. Fail two disks on RAID 5. The system reports DATA LOSS and refuses the rebuild. Repeat with RAID 6 on 5 disks: the SHA-256 still matches.
5. Calculator: comparison table, efficiency chart, XOR demonstration.
6. Disk: raise λ and watch the operating point pass the 70% knee.
7. Backup: RPO 1 h with daily backups fails. Switch to hourly backups and a recommendation appears.

---

## 10. Limitations and threats to validity (be upfront; reviewers will check)

Engine:
- The engine handles erasures (known-missing disks), not silent errors. It keeps no per-block checksums, so it cannot detect or locate bit rot inside a surviving disk. SHA-256 only detects end-to-end corruption.
- File-backed, not block-device-backed. It has no controller cache, write hole, partial-stripe read-modify-write, or concurrent I/O. Whole disks are loaded into memory, so scalability depends on the 5 MiB and 8-disk limits.
- The parity layout is a simple rotation, not the left-symmetric layout of Linux md or a vendor layout.
- `meta.json` has no locking. Concurrent mutations of the same array could race. This is acceptable for a single-user teaching tool but should be stated.
- The rebuild is slowed artificially in the API for visibility.

Models:
- The disk model is M/M/1 (exponential service). Real disk service is closer to a general distribution, so M/G/1 or measured traces would be more faithful. The model is a single disk with no scheduler, cache or array-level aggregation.
- The write penalty is reported but not yet combined with disk IOPS. Effective front-end IOPS = n·IOPS_disk / (r + w·penalty) is a natural extension.
- Backup: under a fixed daily change, incremental and differential restore the same data volume. They differ in steps and retained storage. RPO compliance depends only on the interval, not on the backup's duration. There is no deduplication or compression.
- Replication: one transfer per write, constant propagation speed, no protocol overhead, and the link is modelled as an M/M/1 queue.
- Risk: the 1 to 5 scores come from judgement, not measured frequencies.
- Rebuild time assumes a constant sequential throughput.

Evaluation:
- No user study has been run yet. Learning-outcome claims need one (see Section 11). Until then, limit the claims to correctness and performance.

Security:
- No authentication. CORS is restricted to localhost origins. Uploads are limited to 5 MiB. The array id is validated, which prevents path traversal. Intended for local or classroom deployment.

---

## 11. Future work (for the conclusion)

- A pedagogical evaluation: a pre-test and post-test with control and treatment groups of ISM students, plus a SUS usability questionnaire. This is the most important addition for IEEE Access acceptance.
- Per-block checksums and scrubbing, so the engine can demonstrate silent-corruption detection (error correction rather than erasure correction with RAID 6).
- Array-level IOPS using the write penalty, an M/G/1 disk model, and workload traces.
- Erasure codes beyond RAID 6 (general Reed-Solomon k+m, LRC) and distributed or declustered parity rebuild.
- Rebuild-window risk: the probability of a second failure during a rebuild, using MTTF and MTTR.
- Multi-user deployment with authentication and per-user workspaces.

---

## 12. Reproducibility

```bash
.venv/bin/python -m pytest -q                       # 62 tests
cd frontend && npm test && npm run build            # 11 tests + build
PYTHONPATH=. .venv/bin/python paper/measurements.py # every number in Sections 7.2 and 8
.venv/bin/python -m uvicorn backend.app.main:app --port 8000
cd frontend && npm run dev -- --port 5173
docker compose up --build                            # optional
```

---

## 13. Suggested paper outline (IEEE Access format)

- Title (options):
  - "StorageLab: A Verifiable, Hardware-Independent Laboratory for RAID, Disk Performance, and Data Protection Education"
  - "A File-Backed RAID 0/1/5/6/10 Engine with End-to-End Hash Verification for Storage Systems Education"
- Abstract (≤ 250 words, one paragraph, no citations): the problem, the system, the RAID 6 GF(2^8) engine, the 1,058-case campaign with zero silent errors, timings of at most 25 ms, and the explanation layer.
- Index Terms: RAID, erasure coding, Reed-Solomon codes, Galois fields, storage systems, queueing theory, backup and recovery, replication, computer science education, simulation.
- I. Introduction (Section 2 and Section 3 of this document)
- II. Related Work (see below)
- III. System Architecture (Section 4)
- IV. Block-Level RAID Engine (Section 5)
- V. Analytical Storage Models and Explanation Layer (Section 6)
- VI. Validation and Results (Sections 7 and 8)
- VII. Discussion and Limitations (Section 10)
- VIII. Conclusion and Future Work (Section 11)

Related-work themes and foundational references. These are standard, well-known works, but check each citation's exact details before using it:
- Patterson, Gibson and Katz, "A case for redundant arrays of inexpensive disks (RAID)", ACM SIGMOD, 1988.
- Chen, Lee, Gibson, Katz and Patterson, "RAID: High-performance, reliable secondary storage", ACM Computing Surveys, 1994.
- Reed and Solomon, "Polynomial codes over certain finite fields", J. SIAM, 1960.
- Plank, "A tutorial on Reed-Solomon coding for fault-tolerance in RAID-like systems", Software: Practice and Experience, 1997.
- H. Peter Anvin, "The mathematics of RAID-6" (the Linux kernel's RAID 6 paper, the source of the g = 2 and 0x11D convention).
- Little, "A proof for the queuing formula L = λW", Operations Research, 1961.
- Kleinrock, Queueing Systems, Vol. 1: Theory, Wiley, 1975.
- Ruemmler and Wilkes, "An introduction to disk drive modeling", IEEE Computer, 1994.
- Bucy, Schindler, Schlosser, Ganger et al., The DiskSim simulation environment (CMU PDL technical report, version 4.0).
- EMC Education Services, Information Storage and Management, 2nd ed., Wiley, 2012 (the course textbook; the source of the write penalties and the backup and replication definitions).
- A separate search is needed for work on simulation or visualisation tools in systems and storage education. Do not cite any of it until you have found and read it.

Positioning against these: DiskSim-class simulators are accurate but research-oriented and give no learner-facing explanations. Linux md and hardware RAID are real but opaque and need privileges. Online calculators give numbers without failure behaviour or proof. StorageLab combines executable RAID on real bytes, hash-verified recovery, formula-traceable models, and explanations in one tool that runs on a laptop.
