# 🛰️ Language-First ReAct Agent for PX4 + MAVSDK
**A practical survey of the *skills* you actually need, when you already expose (1) vision JSON for objects {x,y,dist?} and (2) telemetry JSON as tools.**

> **Thesis**  
> In a ReAct-style LangGraph agent, **tools should control the world (actuators) or read platform state**, while *policies/skills* like `scan`, `orienting`, and `approach` are best implemented as **graph-level composites** (subloops) that *use* those tools plus your JSON data.  
> With your two data tools (vision + telemetry), many “query-style” skills become redundant.

---

## 0) TL;DR (What to keep vs. drop)

- **Keep as tools (atomic actuators & platform queries)**:  
  `arm`, `disarm`, `takeoff`, `land`, `rtl`, `hold`,  
  `move_forward`, `move_backward`, `move_left`, `move_right`, `move_up`, `move_down`,  
  `turn_cw`, `turn_ccw`, *(optional)* `set_yaw`, `goto_gps`, `goto_local`,  
  `emergency_brake`, `capture_photo`, `start_video`, `stop_video`,  
  `get_vision_json`, `get_telemetry_json`, *(optional)* `scene_free_ahead` (if your obstacle sensing isn’t trivially derived from vision JSON).

- **Implement as graph-level *composite policies*** (not tools):  
  `scan`, `scan_abstract`, `orienting`, `approach`, `goto` (policy wrapper around atomic moves + guards), `re_plan`.

- **Drop as separate tools (redundant with vision JSON)**:  
  `is_visible`, `object_x`, `object_y`, `object_width`, `object_height`, `object_dis`, `probe` *(LLM can reason directly over JSON & context)*, `delay` *(use `hold(ms)`)*, `move_in_circle` *(rare/mission-specific; emulate with turns + strafes if needed)*, `log` *(logging can be framework-native or a trivial tool—optional)*.

---

## 1) Assumptions (Your Setup)
- **Two data-access tools** are available:
  1) `get_vision_json()` → list of detections, e.g.
     ```json
     [
       {"label": "red cup", "x": 0.58, "y": 0.52, "dist_m": 1.2, "conf": 0.84},
       {"label": "person",  "x": 0.33, "y": 0.45, "conf": 0.92}
     ]
     ```
  2) `get_telemetry_json()` → platform state, e.g.
     ```json
     {"gps_ok": true, "ekf_ok": true, "battery_pct": 62, "mode": "OFFBOARD"}
     ```

- The **agent is ReAct-style** inside **LangGraph**: observe → think → choose tool → observe → …  
- **Safety constraints & validators** clamp unsafe arguments (degrees, distances, velocities, timeouts).

---

## 2) Agent Design: Tools vs. Policies

### 2.1 Atomic **Tools** (what *must* be tools)
These **change the world** or read platform state:
- **Lifecycle & Modes**: `arm`, `disarm`, `takeoff(alt_m)`, `land`, `rtl`
- **Motion (body/local frame)**:  
  `move_forward(cm)`, `move_backward(cm)`, `move_left(cm)`, `move_right(cm)`,  
  `move_up(cm)`, `move_down(cm)`, `turn_cw(deg)`, `turn_ccw(deg)`, *(optional)* `set_yaw(deg)`
- **Navigation**: *(optional)* `goto_gps(lat,lon,alt_m)`, `goto_local(north_m,east_m,down_m)`
- **Safety & Timing**: `hold(ms)`, `emergency_brake()`
- **Camera/Media**: `capture_photo()`, `start_video()`, `stop_video()`
- **Data Access**: `get_vision_json()`, `get_telemetry_json()`  
  *(optionally add `scene_free_ahead(range_m)` if your obstacle sensing needs aggregation)*
- **(Optional) log(level,text)**: You can also handle logging in-agent or via framework hooks.

> **Why**: These map directly to MAVSDK calls / platform queries and need strict schemas, bounds, and retries. They are the **stable control surface** the LLM chooses from.

---

### 2.2 Composite **Policies** (graph-level skills)
Policies are **bounded loops/conditionals** expressed in LangGraph nodes, not tools. They *consume* JSON data + atomic tools:

- **`scan(object, step_deg=30, max_steps=12)`**  
  Rotate in steps until detection appears in `vision_json`.  
  *Implements:* `for k in 1..max_steps: call turn_cw(step) → call get_vision_json() → break if found`.

- **`scan_abstract(question, step_deg=30, max_steps=12)`**  
  Same as `scan` but detection criterion uses a looser match (LLM-side pattern), or retrieves a label to look for.

- **`orienting(object, center_band=[0.4, 0.6], micro_deg=15)`**  
  While `x < 0.4` → `turn_ccw(micro_deg)`; while `x > 0.6` → `turn_cw(micro_deg)`; re-check `vision_json` each step.

- **`approach(object, stop_dist_m=0.6, step_cm=50)`**  
  While `dist_m > stop_dist_m` → check obstacle → `move_forward(step_cm)` → re-check `vision_json`; abort on risk.

- **`goto(object)`**  
  Policy wrapper: `scan(object)` → `orienting(object)` → `approach(object)`; bound total rotation and advance.

- **`re_plan(reason)`**  
  Policy branch on failure: try broader scan band, adjust altitude, or fallback to `goto_local` candidate pose.

> **Why**: You get MiniSpec-like **boundedness and determinism** without inventing a new DSL.  
> You keep **actuators as tools**, and implement **mission logic as policies** in the graph.

---

## 3) Decision Table (Your Provided Skills)

| Skill                     | Keep as Tool? | Implement as Policy? | Rationale |
|---                        |---:|---:|---|
| **scan**                  |  | ✅ | Bounded loop of `turn_*` + `get_vision_json`. |
| **scan_abstract**         |  | ✅ | Same as `scan` with looser/LLM-based target definition. |
| **orienting**             |  | ✅ | Uses `x` from vision JSON to center; iterated micro-turns. |
| **approach**              |  | ✅ | Uses `dist_m` (if available) or proxy (object size), with obstacle checks. |
| **goto**                  |  | ✅ | Composes `scan → orient → approach`. |
| **move_forward/back/…**   | ✅ |  | Atomic actuator → MAVSDK. |
| **turn_cw / turn_ccw**    | ✅ |  | Atomic rotation primitive. |
| **move_in_circle**        | ❌ | (Optional) | Rare; emulate with small `turn_* + move_forward` steps. |
| **delay**                 | ❌ |  | Use `hold(ms)` tool instead (clearer semantics). |
| **is_visible**            | ❌ |  | Redundant—agent can read detections from `get_vision_json`. |
| **object_x / y / w / h**  | ❌ |  | Redundant—fields come from `get_vision_json`. |
| **object_dis**            | ❌ |  | Redundant—use `dist_m` in `get_vision_json` if provided. |
| **probe**                 | ❌ |  | ReAct “thinking step” replaces this; avoid network LLM calls mid-flight if possible. |
| **log**                   | (Optional) |  | Prefer framework logging; a tool is fine if you want operator-visible logs. |
| **take_picture**          | ✅ |  | Camera actuator. |
| **re_plan**               |  | ✅ | A policy branch, not a single tool. |

---

## 4) Minimal Tool Schemas (examples)

```yaml
- name: takeoff
  args: { alt_m: float }       # clamp 1.0..30.0
  returns: { ok: bool, msg?: str }

- name: move_forward
  args: { cm: int }            # clamp 10..300; rate-limit consecutive calls
  returns: { ok: bool }

- name: turn_cw
  args: { deg: float }         # clamp 1..45 per call; accumulate ≤ 360 per scan
  returns: { ok: bool }

- name: get_vision_json
  args: {}
  returns:
    detections: [
      { label: str, x: float, y: float, conf: float, dist_m?: float, w?: float, h?: float }
    ]

- name: get_telemetry_json
  args: {}
  returns:
    gps_ok: bool
    ekf_ok: bool
    battery_pct: float
    link_ok: bool
    mode: str
