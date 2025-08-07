# 🛰️ LangGraph Single-Agent for PX4 + MAVSDK  
**Proposal: Skill Catalog, Tool Schemas, and ReAct-style Workflow**

This proposal defines a **single-agent, tool-calling** architecture (no DSL code generation) that controls a **PX4** UAV via **MAVSDK**. The agent operates in a **ReAct** loop (observe → reason → act), with **guarded skills** exposed as tools. Perception (vision + internal sensors) is available as JSON tools (e.g., `object_pose`, `vehicle_health`), so the agent can query **x, y, z** of targets and **internal state** deterministically.

---

## 1) Design Goals

1. **Direct tool calling** (no MiniSpec/DSL interpreter) for **fast iteration** on behaviors.  
2. **Bounded, auditable actions** via **graph policies** (timeouts, retries, safety checks).  
3. **Perception fusion**: expose **vision outputs** (x,y,z, conf) and **PX4/MAVSDK telemetry** as tools.  
4. **Compositional skills**: high-level “policies” (e.g., scan→orient→approach) implemented as **graph subloops**.  
5. **Research-oriented**: clear logs, ablation switches (e.g., depth on/off), and reproducible runs.

---

## 2) Skill Categories (Tools)

Below are **tools** the agent can call. Each tool has:
- **Name**
- **Category**
- **Schema (args / returns)**
- **Safety rules / guards**
- **MAVSDK mapping** (where applicable)

> **Notation**: JSON-like schemas; all tools must validate ranges and clamp unsafe values.

### A) Vehicle Lifecycle & Modes

1. **arm**
   - `args: {}` → `returns: {ok: bool, msg: str}`
   - Guard: preflight checks must pass; fail if EKF not ready or GPS required but poor.
   - MAVSDK: `action.arm()`

2. **disarm**
   - `args: {}` → `returns: {ok: bool}`
   - Guard: only on ground unless emergency.
   - MAVSDK: `action.disarm()`

3. **takeoff**
   - `args: {alt_m: float}` → `{ok: bool}`
   - Guard: `1.0 ≤ alt_m ≤ 30.0`; require arm + global position ok.
   - MAVSDK: `action.takeoff()` then wait altitude.

4. **land**
   - `args: {}` → `{ok: bool}`
   - MAVSDK: `action.land()`

5. **rtl**
   - `args: {}` → `{ok: bool}`
   - MAVSDK: `action.return_to_launch()`

---

### B) Low-Level Motion (Offboard / Body-frame)

6. **set_yaw**
   - `args: {deg: float}` → `{ok: bool}`
   - Guard: clamp `-180..180`, rate-limit.
   - MAVSDK: offboard yaw or `action.set_current_speed()` + yaw rate.

7. **turn_ccw / turn_cw**
   - `args: {deg: float}` → `{ok: bool}`
   - Guard: clamp `1..90` per call; compose as multiple steps.
   - MAVSDK: NED yaw update (offboard) or `action.set_current_yaw()`.

8. **move_forward**
   - `args: {cm: int}` → `{ok: bool}`
   - Guard: clamp `10..300`; require obstacle check.
   - MAVSDK: `offboard.set_velocity_body(vx,0,0,yaw_rate)` with time window.

9. **move_left / move_right / move_up / move_down**
   - `args: {cm: int}` → `{ok: bool}`
   - Guard: clamp each; check geofence/altitude bounds.
   - MAVSDK: offboard velocity or position setpoints.

10. **hold**
    - `args: {ms: int}` → `{ok: bool}`
    - Guard: clamp `0..5000`.
    - MAVSDK: maintain offboard hold (zero velocity) or `action.hold()`.

---

### C) Navigation & Paths

11. **goto_gps**
    - `args: {lat: float, lon: float, alt_m: float, yaw_deg?: float}` → `{ok: bool}`
    - Guard: valid lat/lon, FENCE; check RTL battery threshold.
    - MAVSDK: `action.goto_location()`.

12. **goto_local**
    - `args: {north_m: float, east_m: float, down_m: float, yaw_deg?: float}` → `{ok: bool}`
    - MAVSDK: offboard position NED setpoints.

13. **follow_path**
    - `args: {waypoints: [{lat,lon,alt_m}], cruise_mps?: float}` → `{ok: bool}`
    - Guard: max N waypoints; min spacing.
    - MAVSDK: `mission.upload_mission()` + `mission.start_mission()` or offboard path.

---

### D) Perception & Scene Queries (JSON-backed tools)

14. **is_visible**
    - `args: {object: str, conf_min?: float}` → `{visible: bool, conf: float}`
    - Source: vision JSON.
    - Guard: `0≤conf_min≤1` (default 0.5).

15. **object_pose**
    - `args: {object: str}` → `{x: float, y: float, z_m?: float, dist_m?: float, conf: float}`
    - Meaning: `x,y ∈ [0,1]` image coords; `z_m/dist_m` if available (stereo/ToF).
    - Source: vision JSON (fused with depth if available).

16. **scene_free_ahead**
    - `args: {range_m: float}` → `{free: bool, min_dist_m: float}`
    - Source: depth/obstacle sensor JSON.
    - Guard: clamp `0.5..10`.

17. **count_objects**
    - `args: {label: str}` → `{count: int}`

---

### E) High-Level Safety & Autonomy (Policy-level tools)

18. **safe_approach_until**
    - `args: {object: str, stop_dist_m: float, step_cm?: int}` → `{ok: bool, reached: bool, final_dist_m?: float}`
    - Guard: `0.3 ≤ stop_dist_m ≤ 3.0`; `step_cm` default 50; each step checks `scene_free_ahead`.
    - Composition: calls `object_pose`, `scene_free_ahead`, `move_forward`, `hold`.

19. **scan**
    - `args: {object: str, step_deg?: int, max_steps?: int}` → `{ok: bool, found: bool, steps: int}`
    - Guard: clamp step `5..45`, cap steps `≤ 12`.
    - Loop: `turn_cw(step)` → `is_visible` check. Stops on found or bound.

20. **orient_to_object**
    - `args: {object: str, center_band?: {min: float, max: float}, micro_deg?: int}` → `{ok: bool, centered: bool}`
    - Defaults: band `[0.4, 0.6]`, micro turn `15°`.
    - Loop: query `object_pose.x`, turn cw/ccw until centered or bound.

21. **track_object**
    - `args: {object: str, horizon_s: float}` → `{ok: bool}`
    - Keeps object centered using small yaw/strafe within safety envelope.

22. **replan**
    - `args: {reason: str}` → `{ok: bool, plan_id: str}`
    - Triggers a policy branch (e.g., widen scan band, change altitude).

---

### F) Health, Telemetry, and Logging

23. **vehicle_health**
    - `args: {}` → `{gps_ok, ekf_ok, imu_ok, magnetometer_ok, battery_pct, link_ok, flight_mode}`
    - Source: MAVSDK telemetry; JSON aggregator.

24. **battery_status**
    - `args: {}` → `{pct: float, voltage_v: float, remaining_m: float?}`

25. **log**
    - `args: {level: "INFO"|"WARN"|"ERROR", text: str}` → `{ok: bool}`

26. **capture_photo / start_video / stop_video**
    - `args: {} / {}` → `{ok: bool}`  
    - MAVSDK: `camera.take_photo()` / `camera.start_video()` / `camera.stop_video()`.

27. **emergency_brake**
    - `args: {}` → `{ok: bool}`  
    - Guard: immediately zero velocities; fall back to land if unstable.

---

## 3) ReAct-Style Workflow (LangGraph)

**State** (single JSON object):
```json
{
  "goal": "goto red cup",
  "last_detection": {"object": "red cup", "x": 0.58, "y": 0.52, "dist_m": 1.4, "conf": 0.81},
  "rotation_accum_deg": 0,
  "forward_accum_m": 0.0,
  "safety": {"battery_pct": 62, "gps_ok": true, "ekf_ok": true, "free_ahead": true},
  "limits": {"max_rotation_deg": 360, "max_forward_m": 5.0, "min_battery_pct": 25}
}
