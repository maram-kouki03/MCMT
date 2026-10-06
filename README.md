# MCMT Pipeline — How It Works

Multi-camera tracking for the 4-quadrant test video (`SC_mcmt_motion_kaggle.py`). Each quadrant is treated as an independent camera; this explains how a person's ID survives moving between them.

---

## 🔵 1. Per-camera detection (unchanged from single-camera)

Each camera runs its own **YOLO-pose + BoT-SORT**. This gives a **local track ID** — only meaningful inside that one camera. The same physical person gets a *different* local ID in each camera they pass through.

<span style="color:#1f6feb"><b>Core vs. Full region, per camera:</b></span>
- **Core** = the camera's strict, non-overlapping quadrant → used for drawing, zones (SELLER/entry/PEC), and counting. Nobody is drawn or counted twice.
- **Full** = Core + `OVERLAP_PX` extending into neighboring quadrants → used for detection/tracking only, so a person isn't clipped in half right at the boundary.

---

## 🟢 2. Cross-camera matching — motion prediction (PRIMARY)

When a brand-new local track appears in any camera, before touching appearance at all:

1. Every local track's **velocity** is kept live, in shared canvas coordinates (not per-camera — one shared coordinate system).
2. For the new track to even be considered a hand-off, it must be **near one of its own camera's shared edges** — a person appearing in the *middle* of a camera isn't a hand-off case.
3. Look at every *adjacent* camera's currently-live tracks. A candidate must pass **all four** of:

| Check | What it enforces | Why |
|---|---|---|
| 🟢 **Boundary proximity** | the departing track was actually near the *shared* edge when last seen | a track lost in the middle of a camera isn't a hand-off |
| 🟢 **Direction** | its velocity has a real component *through* that specific edge | excludes people merely standing near a boundary |
| 🟢 **Predicted position** | last position + velocity × elapsed time lands close to the new track | the actual "is this physically plausible" check |
| 🟢 **Claim-once** | a departing track can only confirm **one** arrival, ever | stops two different people both being matched to the same ID |

The match radius **grows slightly with elapsed time** (`TOL_GROWTH_PX_PER_SEC`) — tight and precise for fast, clean crossings; a bit more forgiving for slightly delayed detections.

---

## 🟠 3. Appearance — TIE-BREAK and FALLBACK only

- **Tie-break:** if more than one candidate passes all four checks above (rare, once direction+boundary filtering is in place), the OSNet embedding picks between *just those few* — not searched against the whole store.
- **Fallback:** if motion finds *no* candidate at all (re-entry after a longer gap, or genuinely new person), fall back to a stricter whole-gallery appearance search (`FALLBACK_SIM_THRESHOLD`).

Appearance is never the first thing checked — it only resolves ambiguity motion couldn't.

---

## 🟣 4. What this fixes vs. the earlier version

| Symptom you reported | Root cause | Fix |
|---|---|---|
| Two different people, two cameras → same ID | No boundary/direction check — any nearby prediction counted | boundary proximity + direction + claim-once |
| ID occasionally changes crossing cameras ("small cases") | Fixed-radius tolerance too strict for slightly-delayed detections, and false candidates stealing the correct match | growing tolerance + fewer false candidates competing |

---

## ⚙️ Quick config reference

| Constant | Controls | Raise it if… | Lower it if… |
|---|---|---|---|
| `OVERLAP_PX` | how far cameras see into neighbors | people still get clipped at the boundary | detection runs too slow |
| `BOUNDARY_MARGIN_PX` | how close to an edge counts as "near it" | real crossings aren't detected as near-boundary | unrelated people near the edge get drawn in |
| `MIN_EXIT_SPEED` | min speed to count as "heading out" | slow walkers near a boundary falsely match | genuine slow crossings get missed |
| `POSITION_TOLERANCE_PX` / `TOL_GROWTH_PX_PER_SEC` | match radius (base / growth over time) | real handoffs still missed | false merges still happening |
| `HANDOFF_WINDOW_SEC` | how long a prediction stays valid | people pause near a boundary before crossing | stale matches reaching across long gaps |
| `FALLBACK_SIM_THRESHOLD` | strictness of the no-motion-candidate fallback | fallback creates false merges | genuine re-entries get a fresh ID instead of reconnecting |

---

## 🔴 Known limitations — things to improve later

- 🔴 **No real camera calibration.** This is still a single video cut into quadrants, not 4 real lenses — real cameras will have different angles/lighting/scale, which will hurt the appearance matcher more than this test can show.
- 🔴 **Claimed-track flag resets on the next touch.** Once a departing track is claimed, continuing to track it normally clears that flag — fine in practice, but not airtight if a double-claim happens *after* it resumes moving.
- 🟡 **No re-identification after a long absence across non-adjacent cameras** (e.g. someone leaves via camera A, comes back 5 minutes later in camera D) — only the appearance fallback can catch this, and only within `GLOBAL_GALLERY_TTL_SEC`.
- 🟡 **`UNIFORM_BAND_LO/HI` (seller color) and all `CAMERA_*_ZONES`** are still placeholders/old-CRK values — must be recalibrated per test video for SELLER/entry/PEC logic to mean anything.
- 🟡 **Velocity estimation is a simple EMA**, not a Kalman filter — fine for roughly-straight-line walking, will lag on sharp turns right at a boundary.
- 🟡 **No person re-identification by face/clothing persistence across a full session** — if someone changes visual appearance drastically mid-video (removes a jacket), appearance-based fallback/tie-break will weaken.