# 🧠 MAVSDK Skill Set Reference

This document describes the **core skill abstractions** for drone control, structured to support an LLM-based Domain-Specific Language (DSL) runtime. Skills are categorized into:

- 🔹 **Low-Level Drone Skills** — atomic drone commands (e.g. move, rotate, check visibility).
- 🔸 **High-Level Reasoning Skills** — compound behaviors that combine low-level commands to achieve goals (e.g. scan, approach, orient).

---

## 🔸 High-Level Skills (Composite Tasks)

These skills perform **multi-step reasoning or movement**, often using conditionals or sequential planning.

| Abbr | Skill Name     | Arguments                | Definition Syntax                                 | Description |
|------|----------------|--------------------------|---------------------------------------------------|-------------|
| `s`  | `scan`         | `object_name: str`       | `8{?iv($1)==True{->True}tc(45)}->False;`          | Rotate the drone by 45° increments until the object becomes visible or a full circle is completed. |
| `sa` | `scan_abstract`| `question: str`          | `8{_1=p($1);?_1!=False{->_1}tc(45)}->False;`      | Use the LLM to probe for an abstract description (e.g. “car”) and rotate until found. |
| `o`  | `orienting`    | `object_name: str`       | `4{_1=ox($1);?_1>0.6{tc(15)};?_1<0.4{tu(15)};_2=ox($1);?_2<0.6&_2>0.4{->True}}->False;` | Align the drone with the horizontal center of the object in view (X ∈ [0.4, 0.6]) |
| `a`  | `approach`     | _none_                   | `mf(100);`                                        | Move forward a fixed distance toward a target object. |
| `g`  | `goto`         | `object_name: str`       | `orienting($1);approach();`                       | Composite skill: First orient to object, then approach. |

---

## 🔹 Low-Level Skills (Atomic Control)

These are **primitive operations** to control drone movement, sensory input, or logic flow.

### 🚁 Movement & Orientation

| Abbr | Skill Name      | Arguments                | Description |
|------|------------------|--------------------------|-------------|
| `mf` | `move_forward`   | `distance: int`          | Move forward by specified distance (cm). |
| `mb` | `move_backward`  | `distance: int`          | Move backward by specified distance. |
| `ml` | `move_left`      | `distance: int`          | Strafe left. |
| `mr` | `move_right`     | `distance: int`          | Strafe right. |
| `mu` | `move_up`        | `distance: int`          | Ascend. |
| `md` | `move_down`      | `distance: int`          | Descend. |
| `tc` | `turn_cw`        | `degrees: int`           | Rotate clockwise by given degrees. |
| `tu` | `turn_ccw`       | `degrees: int`           | Rotate counterclockwise. |
| `mi` | `move_in_circle` | `cw: bool`               | Move in a circular path (CW or CCW). |

### 🧠 Perception & Logic

| Abbr | Skill Name     | Arguments                | Description |
|------|----------------|--------------------------|-------------|
| `iv` | `is_visible`   | `object_name: str`       | Check if object is visible in current frame. |
| `ox` | `object_x`     | `object_name: str`       | Get horizontal position of object [0,1]. |
| `oy` | `object_y`     | `object_name: str`       | Get vertical position of object [0,1]. |
| `ow` | `object_width` | `object_name: str`       | Get width of detected object [0,1]. |
| `oh` | `object_height`| `object_name: str`       | Get height of detected object [0,1]. |
| `od` | `object_dis`   | `object_name: str`       | Get distance to object in cm. |
| `p`  | `probe`        | `question: str`          | Ask LLM for knowledge-based answer (used in abstract scan). |

### 🧰 Utility

| Abbr | Skill Name  | Arguments            | Description |
|------|-------------|----------------------|-------------|
| `d`  | `delay`     | `milliseconds: int`  | Wait before next action (e.g. stabilize). |
| `tp` | `take_picture`| _none_             | Capture image from RGB camera. |
| `l`  | `log`       | `text: str`          | Print message to console (useful for debugging/notifications). |
| `rp` | `re_plan`   | _none_               | Trigger replanning when current path fails. |

---

## ✅ Usage Examples

```plaintext
// Orient and approach a person
g("person")

// Scan clockwise to find a car
s("car")

// Perform abstract scan for something resembling "red object"
sa("red object")

// Log a success message after detection
l("Object located and reached.")

// Delay for stabilization
d(500)
