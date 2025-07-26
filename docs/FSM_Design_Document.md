# 🧠 Finite State Machine (FSM) Design for F1TENTH Roboracer

## 📌 Overview

This document defines the proposed Finite State Machine (FSM) structure for the F1TENTH autonomous vehicle. It governs the high-level behavior based on inputs from planning, perception, and control modules. The design is modular, safety-aware, and scalable to accommodate future race strategies or failure recovery logics.

---

## 🎯 Objectives

- Define a deterministic and modular FSM for core driving behavior.
- Support normal operation and failure recovery.
- Maintain safety via emergency override logic.
- Allow easy expansion as system complexity grows.

---

## 🧱 High-Level States

| State | Description |
|-------|-------------|
| `IDLE` | Default state. Vehicle is powered but inactive, awaiting a start command. |
| `STARTUP` | Initialization state for sensor check, system diagnostics, and component synchronization. |
| `WAIT_FOR_PATH` | Awaiting path from global planner. Can occur after startup or replanning. |
| `TRACKING` | Following a given path using local planner and perception feedback. |
| `PLANNING_OVERTAKE` | Obstacle detected — evaluating overtake feasibility. |
| `EXECUTE_OVERTAKE` | Actively executing an overtaking maneuver using alternate trajectory. |
| `OBSTACLE_AVOIDANCE` | For basic lateral avoidance (e.g., static debris). Can be merged with `PLANNING_OVERTAKE` if overtake logic is unified. |
| `EMERGENCY_STOP` | Safety-critical halt due to hardware/software fault or external kill command. |
| `RECOVERY` | Attempting to recover from fault by reinitializing subsystems or re-localizing. |
| `RECOVERY_FAILED` | Recovery failed after max attempts. Requires human intervention or reset. |
| `FINISHED` | Goal reached successfully. Enters passive state before shutdown. |
| `SHUTDOWN` | Final system shutdown. May include data logging or network disconnect. |

---

## 🔄 Transition Rules

| From | To | Trigger |
|------|----|---------|
| `IDLE` | `STARTUP` | `/start_flag` received |
| `STARTUP` | `WAIT_FOR_PATH` | All system checks passed |
| `WAIT_FOR_PATH` | `TRACKING` | Global path received |
| `TRACKING` | `PLANNING_OVERTAKE` | Obstacle detected (moving or static) |
| `PLANNING_OVERTAKE` | `EXECUTE_OVERTAKE` | Feasible overtake path computed |
| `EXECUTE_OVERTAKE` | `TRACKING` | Overtake maneuver complete |
| `TRACKING` | `FINISHED` | Goal reached or race completed |
| `FINISHED` | `SHUTDOWN` | Time/event-based trigger |
| Any | `EMERGENCY_STOP` | Critical fault, safety violation, or `/kill_switch` activated |
| `EMERGENCY_STOP` | `RECOVERY` | Recovery signal issued |
| `RECOVERY` | `TRACKING` | Recovered successfully |
| `RECOVERY` | `RECOVERY_FAILED` | Max retry limit reached |

---

## 🗺️ FSM State Transition

                              +--------+
                             |  IDLE  |
                             +--------+
                                  |
                       /start_flag received
                                  v
                            +-----------+
                            |  STARTUP  |
                            +-----------+
                                  |
                       system checks passed
                                  v
                       +-------------------+
                       |  WAIT_FOR_PATH    |
                       +-------------------+
                                  |
                           path received
                                  v
                             +--------+
                             |TRACKING|
                             +--------+
                             /        \
                obstacle detected     \ goal reached
                           v           v
                +----------------+  +-----------+
                |PLANNING_OVERTAKE| | FINISHED  |
                +----------------+  +-----------+
                          |               |
              valid path computed   end trigger
                          v               v
                +------------------+   +---------+
                |EXECUTE_OVERTAKE  |   | SHUTDOWN|
                +------------------+   +---------+
                          |
                maneuver complete
                          v
                      +--------+
                      |TRACKING|
                      +--------+
                          |
              emergency or fault detected
                          v
                +------------------+
                | EMERGENCY_STOP   |
                +------------------+
                          |
                  recovery command
                          v
                      +--------+
                      |RECOVERY|
                      +--------+
                       /      \
             success /        \ failure
                   v           v
              +--------+   +------------------+
              |TRACKING|   | RECOVERY_FAILED  |
              +--------+   +

---

### References:
![Behavior-Planning-by-Finite-State-Machine repo](https://github.com/A2Amir/Behavior-Planning-by-Finite-State-Machine)  
![Nav2 Project Documentation](https://docs.nav2.org/)  
![ROS Navigation Project](https://github.com/ros-navigation)  
![Autoware Documentation](https://autowarefoundation.github.io/autoware-documentation/main/)  
![Finite-State Machine Wiki](https://en.wikipedia.org/wiki/Finite-state_machine)  
![Automata Theory Wiki](https://en.wikipedia.org/wiki/Automata_theory)  




