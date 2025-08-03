# 🧠 Finite State Machine (FSM) Design for F1TENTH Roboracer

## 📌 Overview

This document defines the proposed Finite State Machine (FSM) structure for the F1TENTH autonomous vehicle. It governs the high-level behavior based on inputs from planning, perception, and control modules. The design is modular, safety-aware, and scalable to accommodate future race strategies or failure recovery logics.

---

## 🎯 Objectives

- Define a deterministic and modular FSM for core driving behavior.
- Support normal operation and failure recovery.
- Maintain safety via manual override logic.
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
| `RECOVERY` | Attempting to recover from fault by reinitializing subsystems or re-localizing. |

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
| `TRACKING` | `OBSTACLE_AVOIDANCE` | Basic obstacle detected requiring lateral avoidance |
| `OBSTACLE_AVOIDANCE` | `TRACKING` | Avoidance maneuver complete |
| `TRACKING` | `IDLE` | Manual stop command (`/stop_flag` or user interrupt) |
| `TRACKING` | `RECOVERY` | System fault detected |
| `PLANNING_OVERTAKE` | `RECOVERY` | Planning failure or system fault |
| `EXECUTE_OVERTAKE` | `RECOVERY` | Execution failure or system fault |
| `OBSTACLE_AVOIDANCE` | `RECOVERY` | Avoidance failure or system fault |
| `RECOVERY` | `TRACKING` | Recovered successfully |
| `RECOVERY` | `IDLE` | Recovery unsuccessful - manual intervention required |

---

## 🗺️ FSM State Transition

                              +--------+
                             |  IDLE  |<---------------------------+
                             +--------+                           |
                                  |                               |
                       /start_flag received           manual stop |
                                  v                    command    |
                            +-----------+                         |
                            |  STARTUP  |                         |
                            +-----------+                         |
                                  |                               |
                       system checks passed                       |
                                  v                               |
                       +-------------------+                      |
                       |  WAIT_FOR_PATH    |                      |
                       +-------------------+                      |
                                  |                               |
                           path received                          |
                                  v                               |
                             +--------+---------------------------+
                             |TRACKING|
                             +--------+
                             /   |   \
              obstacle detected  |    \ basic obstacle
                           v     |     v
                +----------------+ +------------------+
                |PLANNING_OVERTAKE| |OBSTACLE_AVOIDANCE|
                +----------------+ +------------------+
                          |                 |
              valid path computed    avoidance complete
                          v                 |
                +------------------+        |
                |EXECUTE_OVERTAKE  |        |
                +------------------+        |
                          |                 |
                maneuver complete           |
                          v                 v
                      +--------+<-----------+
                      |TRACKING|
                      +--------+
                          |
                 fault detected
                          v
                      +--------+
                      |RECOVERY|
                      +--------+
                       /      \
             success /        \ failure
                   v           v
              +--------+   +--------+
              |TRACKING|   |  IDLE  |
              +--------+   +--------+

---

### References:
![Behavior-Planning-by-Finite-State-Machine repo](https://github.com/A2Amir/Behavior-Planning-by-Finite-State-Machine)  
![Nav2 Project Documentation](https://docs.nav2.org/)  
![ROS Navigation Project](https://github.com/ros-navigation)  
![Autoware Documentation](https://autowarefoundation.github.io/autoware-documentation/main/)  
![Finite-State Machine Wiki](https://en.wikipedia.org/wiki/Finite-state_machine)  
![Automata Theory Wiki](https://en.wikipedia.org/wiki/Automata_theory)
