# 🧠 Simplified Finite State Machine (FSM) Design for F1TENTH Roboracer

## 📌 Overview

This document defines a **simplified and initial** Finite State Machine (FSM) structure for the F1TENTH autonomous vehicle. This barebones implementation focuses on core driving behaviors and serves as the foundation for future expansion. The design prioritizes modularity and simplicity to enable rapid prototyping and testing of essential functionality.

---

## 🎯 Objectives

- Define a minimal but functional FSM for core autonomous driving behavior.
- Establish a solid foundation for future system expansion.
- Enable rapid prototyping and testing of essential states.
- Maintain modular design for easy integration of additional states later.

---

## 🧱 Core States (Initial Implementation)

| State | Description |
|-------|-------------|
| `IDLE` | Default, passive state. Vehicle is powered but inactive, not moving, awaiting a start command to proceed. |
| `STARTUP` | Initialization phase for system checks, sensor calibration, and startup routines. Transitions to TRACKING when ready. |
| `TRACKING` | Main driving mode. Vehicle follows the planned trajectory on the map using local planner and perception feedback. |
| `OVERTAKE` | Triggered when a blocking obstacle (static or moving) is detected. Executes complete overtake maneuver and returns to TRACKING. |

---

## 🔄 Transition Rules

| From | To | Trigger |
|------|----|---------|
| `IDLE` | `STARTUP` | Start command received (e.g., `/start_flag`) |
| `STARTUP` | `TRACKING` | All initialization routines completed successfully |
| `TRACKING` | `OVERTAKE` | Blocking object detected ahead and overtaking deemed feasible |
| `OVERTAKE` | `TRACKING` | Overtake maneuver completed successfully |
| `TRACKING` | `IDLE` | Manual stop signal or emergency interrupt received |

---

## 🗺️ FSM State Transition Diagram

```
                    +--------+
                   |  IDLE  |<-----------------------+
                   +--------+                        |
                        |                           |
             start command received         manual stop/
                        |                  emergency interrupt
                        v                           |
                  +-----------+                     |
                  |  STARTUP  |                     |
                  +-----------+                     |
                        |                           |
            initialization complete                 |
                        |                           |
                        v                           |
                  +------------+                    |
                  |  TRACKING  |--------------------+
                  +------------+
                        |    ^
           obstacle detected |    | maneuver complete
                        |    |
                        v    |
                  +------------+
                  |  OVERTAKE  |
                  +------------+
```

---

## 📝 Implementation Notes

### ✅ Why the Simplified FSM (Minimization Rationale)
We intentionally selected this reduced FSM instead of the full design (see `FSM_Design_Document_Full.md`) based on classic **FSM minimization** principles: merge states that are currently indistinguishable with respect to (a) the event alphabet we handle now and (b) externally observable outputs / side‑effects. Any state whose transitions and outward behavior are equivalent (given the present system scope) becomes redundant until new differentiating conditions or outputs are introduced.

#### Event Alphabet (Current Phase)
`{ start_cmd, init_complete, obstacle_blocking, overtake_done, stop_cmd, emergency_interrupt }`

Under this limited alphabet, several states in the full FSM collapse into equivalence classes:

| Full FSM State(s) | Simplified Representative | Reason for Equivalence (Current Scope) |
|-------------------|---------------------------|----------------------------------------|
| `WAIT_FOR_PATH` + `STARTUP` (post checks) | `STARTUP` | Both only gate transition to driving; path assumed available → no distinct branching. |
| `PLANNING_OVERTAKE` + `EXECUTE_OVERTAKE` | `OVERTAKE` | No external consumer distinguishes planning vs execution; both are “handling dynamic obstacle until resolved”. |
| `OBSTACLE_AVOIDANCE` | `OVERTAKE` | Same trigger (blocking object) and termination condition (path clear) in current implementation. |
| `EMERGENCY_STOP` | `IDLE` | Both result in halted vehicle; no separate recovery channel yet. |
| `RECOVERY`, `RECOVERY_FAILED` | (omitted) | No implemented recovery routines; would be unreachable or degenerate to IDLE. |
| `FINISHED`, `SHUTDOWN` | `IDLE` | Race completion not yet producing a distinct post-finish behavior set. |

Thus the minimized deterministic automaton over the current event set has four necessary states: `IDLE`, `STARTUP`, `TRACKING`, `OVERTAKE`.

#### Benefits Now
- Smaller test matrix (4 states, 6 primary transitions) → faster validation.
- Lower cognitive load for early contributors.
- Avoids premature branching logic (planning vs executing overtake) before metrics justify it.
- Clear upgrade path: any future feature that introduces a new distinguishing trigger/output (e.g., explicit recovery attempts, multi-phase overtake planner) re-splits the merged state(s) without breaking the public interface.

#### When to Re-Expand
- Introduce asynchronous global planner delivering paths → restore `WAIT_FOR_PATH`.
- Add multi-phase overtake pipeline (cost evaluation → trajectory commit) → split `OVERTAKE` into planning/execution.
- Implement structured recovery procedures (re-localization, controller reset) → add `RECOVERY` (+ possible failure terminal state).
- Add race session lifecycle handling → reintroduce `FINISHED` / controlled `SHUTDOWN`.

For now, the simplified FSM is the minimal sufficient controller consistent with observed triggers and required outputs, aligning with FSM minimization theory while preserving extensibility.

### 🔧 Removed States for Initial Implementation
The following states from the original comprehensive design have been **temporarily removed** to focus on core functionality:
- `WAIT_FOR_PATH` - Simplified by assuming path is available during STARTUP
- `PLANNING_OVERTAKE` & `EXECUTE_OVERTAKE` - **Combined into single `OVERTAKE` state**
- `OBSTACLE_AVOIDANCE` - Merged with overtake logic
- `EMERGENCY_STOP` - Simplified to direct transition to IDLE
- `RECOVERY` & `RECOVERY_FAILED` - Will be implemented in future iterations
- `FINISHED` & `SHUTDOWN` - Simplified to IDLE transition

### 🚀 Future Expansion Considerations
- **Overtake State Split**: The current `OVERTAKE` state can be expanded into `PLANNING_OVERTAKE` and `EXECUTE_OVERTAKE` during debugging or when more granular control is needed.
- **Error Handling**: Emergency stop and recovery states will be added after core functionality is validated.
- **Path Management**: `WAIT_FOR_PATH` state can be reintroduced when global path planning becomes more complex.
- **Race Completion**: `FINISHED` and `SHUTDOWN` states for proper race completion handling.

### 🛠️ Integration Guidelines
- Keep state transitions deterministic and well-defined
- Implement state entry/exit actions for proper resource management
- Use ROS topics/services for state transition triggers
- Log all state changes for debugging and analysis
- Design interfaces to easily accommodate future state additions

---

## 🎯 Next Steps
1. **Implement Core States**: Focus on getting IDLE → STARTUP → TRACKING → OVERTAKE working reliably
2. **Test Basic Functionality**: Validate path following and simple obstacle avoidance
3. **Add Logging & Monitoring**: Implement comprehensive state change logging
4. **Expand Gradually**: Add removed states incrementally based on testing needs and system requirements

---

### References:
![Behavior-Planning-by-Finite-State-Machine repo](https://github.com/A2Amir/Behavior-Planning-by-Finite-State-Machine)  
![Nav2 Project Documentation](https://docs.nav2.org/)  
![ROS Navigation Project](https://github.com/ros-navigation)  
![Autoware Documentation](https://autowarefoundation.github.io/autoware-documentation/main/)  
![Finite-State Machine Wiki](https://en.wikipedia.org/wiki/Finite-state_machine)  
![Automata Theory Wiki](https://en.wikipedia.org/wiki/Automata_theory)
