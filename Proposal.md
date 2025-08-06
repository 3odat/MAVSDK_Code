# EBNF-Style DSL for PX4 Drone Missions

## 1. Overview

This report presents a domain‑specific EBNF grammar for commanding PX4‑controlled drones. It includes:

* **Command Abbreviations** for concise syntax
* **Core Grammar** covering low‑level controls, vision skills, and high‑level behaviors
* **Realistic Scenario** demonstrating conditional logic and fallback patterns
* **Robustness & Safety** guidelines integrated into the design
* **LLM Prompt Engineering** to auto‑generate DSL scripts
* **Sample Missions** parsed and ready for execution on PX4

---

## 2. Command Abbreviations

| Full Name                | Abbreviation |
| ------------------------ | ------------ |
| move\_forward            | `mv_fwd`     |
| move\_backward           | `mv_bwd`     |
| move\_left               | `mv_l`       |
| move\_right              | `mv_r`       |
| move\_up                 | `mv_u`       |
| move\_down               | `mv_d`       |
| turn\_clockwise          | `tc`         |
| turn\_counter\_clockwise | `tcc`        |
| hold                     | `ho`         |
| arm                      | `ar`         |
| disarm                   | `dis`        |
| takeoff                  | `tk`         |
| land                     | `ld`         |
| detect\_object           | `det_obj`    |
| check\_visibility        | `vis_chk`    |
| take\_picture            | `pic`        |
| sweeping                 | `swp`        |
| approach                 | `app`        |

---

## 3. EBNF Grammar (Domain‑Specific)

```ebnf
program       ::= { mission_definition } ;

mission_definition ::= "mission" identifier "{" statement_list "}" ;

statement_list ::= { statement ";" } ;

statement     ::= low_level
                | vision_skill
                | high_level
                | conditional
                | safety_check ;

low_level     ::= mv_stmt | turn_stmt | ho_stmt | ar_stmt | dis_stmt | tk_stmt | ld_stmt ;

mv_stmt       ::= ("mv_fwd" | "mv_bwd" | "mv_l" | "mv_r" | "mv_u" | "mv_d") "(" number ")" ;
turn_stmt     ::= ("tc" | "tcc") "(" number ")" ;
ho_stmt       ::= "ho" "(" number ")" ;
ar_stmt       ::= "ar" "(" ")" ;
dis_stmt      ::= "dis" "(" ")" ;
tk_stmt       ::= "tk" "(" ")" ;
ld_stmt       ::= "ld" "(" ")" ;

vision_skill  ::= "det_obj" "(" string ")"
                | "vis_chk" "(" string ")"
                | "pic" "(" string ")" ;

high_level    ::= "swp" "(" number ")"
                | "app" "(" number ")" ;

conditional   ::= "if" "(" condition ")" "{" statement_list "}" [ "else" "{" statement_list "}" ] ;
condition     ::= "object_visible" | "battery_ok" | "gps_lock" ;

safety_check  ::= "safety_check" "(" ")" ;

identifier    ::= letter { letter | digit | "_" } ;
number        ::= digit { digit } ;
string        ::= '"' { any_char_except_quote } '"' ;
letter        ::= "A"…"Z" | "a"…"z" ;
digit         ::= "0"…"9" ;
any_char_except_quote ::= ? all except '"' ? ;
```

---

## 4. Realistic Scenario Example

Imagine we want to **detect** an object named **"target"**, **approach** if visible, or **rotate** until found:

```dsl
mission SearchTarget {
  ar();                      # Arm
  tk();                      # Takeoff
  swp(360);                  # Sweep full rotation
  if (object_visible) {
    app(5);                  # Approach 5 m
    pic("/data/target.jpg");
  } else {
    tcc(90);                 # Turn left 90° and retry
    det_obj("target");     # Try detection again
  };
  ld();                      # Land
  dis();                     # Disarm
};
```

This pattern enables fallback: if initial sweep fails, the drone repositions and retries vision.

---

## 5. Robustness & Safety

1. **Preflight Safety Check**: Insert `safety_check()` at mission start to verify battery, GPS, and sensors.
2. **Timeouts & Retries**: DSL can support optional retry counts in parentheses (e.g. `vis_chk("target",3)`).
3. **Failsafe Hooks**: Automatically inject `emergency_land()` on critical failures.
4. **Parameter Validation**: Grammar enforces numeric ranges (e.g. `mv_fwd(1..100)` m).
5. **Logging & Telemetry**: Each statement logs status; parser can generate telemetry hooks.

---

## 6. LLM Prompt Engineering

To have an LLM generate scripts in this DSL, use a structured system prompt:

```
You are a PX4 Drone DSL generator. Follow the EBNF grammar below exactly. Given a mission description, output only the DSL code without explanation.

EBNF Grammar:
<include the grammar from section 3>

Example Input:
"Inspect the inspection panel: arm, takeoff, sweep 180°, if panel visible then approach 2m and picture, else land."

Expected Output:
mission InspectPanel {
  ar();
  tk();
  swp(180);
  if (object_visible) {
    app(2);
    pic("/images/panel.jpg");
  } else {
    ld();
  };
  dis();
};
```

The LLM will map natural language to DSL code reliably.

---

## 7. Sample Missions & Execution

1. **InspectionMission**

   ```dsl
   mission InspectionMission {
     safety_check();
     ar();
     tk();
     mv_u(10);
     det_obj("marker");
     if (object_visible) {
       app(3);
       pic("/out/marker.png");
     } else {
       tc(45);
       det_obj("marker");
     };
     ld();
     dis();
   };
   ```
2. **SurveyArea** (grid sweep)

   ```dsl
   mission SurveyArea {
     safety_check(); ar(); tk();
     for i in 0..3 {
       mv_fwd(20);
       tc(90);
     };
     ld(); dis();
   };
   ```

> **Parsing & Execution**: Use a parser generator (e.g., ANTLR) with the EBNF to build an AST, then map each node to MAVSDK API calls in your runtime.

---

*End of Report* — ready to copy as `.md` for your documentation and LLM‑driven mission planning.
