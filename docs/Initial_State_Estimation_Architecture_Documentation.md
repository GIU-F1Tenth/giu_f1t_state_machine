# F1TENTH Roboracer – State Estimation Architecture

## Overview

This document describes the design and implementation of a state estimation architecture for the F1TENTH autonomous vehicle. The system fuses data from multiple onboard sensors to produce a reliable and consistent estimate of the vehicle's pose and velocity in real time. The estimated state is a critical input to control, planning, and decision-making modules.

---

## State Variables to be Estimated

The following state variables are estimated in our current flat-track racing setup:

- **Position (x, y)** – Vehicle’s horizontal location in the odom frame (meters).
- **Orientation (yaw)** – Vehicle’s heading angle in the plane (radians).
- **Linear velocity (vₓ)** – Forward speed along the x-axis (m/s).
- **Optional lateral velocity (vᵧ)** – Side-slip velocity along the y-axis (m/s), useful for high-speed cornering analysis.
- **Yaw rate (ω_z)** – Angular velocity around the vertical axis (rad/s).

**Why not roll, pitch, vertical position, or vertical acceleration?**  
The Roboracer vehicle operates on a flat, smooth indoor track, so vertical motion is negligible. Roll and pitch are minimal in this context and do not significantly impact control. Estimating them would add noise and complexity without performance benefit at this stage.

---

## Available Sensor Inputs

| Sensor                          | Topic Name               | Message Type                  | Data Provided                                        | Frequency  | Frame ID     |
|---------------------------------|--------------------------|--------------------------------|------------------------------------------------------|------------|--------------|
| Wheel Odometry (VESC)           | `/odom`                  | `nav_msgs/Odometry`            | x, y, yaw (optional), vₓ, vᵧ (optional), yaw rate    | 50–100 Hz  | `odom`       |
| IMU (SparkFun Artemis)          | `/imu/data_raw`          | `sensor_msgs/Imu`              | angular velocity (ω), linear acceleration (aₓ, aᵧ)  | 100–200 Hz | `imu_link`   |
| LiDAR SLAM (Hokuyo UST-10LX)    | `/slam_out_pose`         | `geometry_msgs/PoseStamped`    | absolute x, y, yaw                                   | 5–10 Hz    | `map`        |
| Optional Visual Odometry (VO)   | `/vo_odom`               | `nav_msgs/Odometry`            | relative pose and velocity                          | 10–30 Hz   | `camera_link`|

---

## Selected Estimation Algorithm

We begin with an **Extended Kalman Filter (EKF)** using the `robot_localization` ROS 2 package. The EKF is well-suited for our application because:

- **Nonlinear motion handling** – It accommodates the nonlinear relationship between state variables and sensor measurements in vehicle kinematics.
- **Computational efficiency** – Lightweight enough to run in real time on our embedded hardware.
- **Sensor fusion flexibility** – Supports asynchronous inputs from multiple sensors with different rates.
- **Proven ROS integration** – Widely used and documented in robotics projects.

The modular design of our pipeline allows future upgrades or replacements — for example, pairing EKF with Visual SLAM for drift correction, or moving to a factor-graph-based approach — without requiring major changes to downstream consumers.

---

## Modular State Estimation Pipeline

The state estimation system is built as a modular ROS-based pipeline using the **Extended Kalman Filter (EKF)** implementation from the [`robot_localization`](http://docs.ros.org/en/noetic/api/robot_localization/html/index.html) package. The architecture supports asynchronous sensor updates, incorporates robust sensor fusion techniques, and outputs the estimated state at a fixed rate.

### Key Features
- **Asynchronous Sensor Handling** – Each sensor operates at its own rate; the EKF handles delayed and out-of-order messages.
- **Robust Sensor Fusion** – High-rate wheel odometry and IMU data are fused for smooth, drift-resistant local tracking, with the option to integrate low-rate global corrections from LiDAR-based SLAM.
- **Fixed-Rate Output** – The EKF publishes a fused state estimate at a constant frequency (50–100 Hz) to ensure predictable performance for downstream modules.
- **Planar Motion Mode** – Configured for two-dimensional operation, estimating only x, y, and yaw to match the vehicle’s kinematics.

### Pipeline Components
1. **Sensor Drivers**
   - **VESC 6 MkVI** → `vesc_driver` + `vesc_to_odom` → `/odom`
   - **IMU (SparkFun OpenLog Artemis)** → custom/rosserial node → `/imu/data_raw`
   - **LiDAR (Hokuyo UST-10LX)** → `urg_node` → `/scan` → (optional SLAM node) `/slam_out_pose`
   - **Camera (Intel RealSense D435)** → `realsense2_camera` (reserved for future visual odometry integration)

2. **Fusion Node**
   - `ekf_localization_node` (robot_localization package)
   - Inputs:
     - `/odom` (wheel odometry) – provides forward velocity and relative position changes
     - `/imu/data_raw` (IMU) – provides angular velocity and linear accelerations
     - `/slam_out_pose` (LiDAR pose, optional) – provides absolute position for global correction
   - Outputs:
     - `/odometry/filtered` (`nav_msgs/Odometry`)
     - TF transform `odom → base_link`

3. **Output Consumers**
   - **Control module** – reads velocity and pose for trajectory following
   - **Local planner** – uses fused pose and velocity for path tracking
   - **Finite State Machine (FSM)** – bases state transitions on accurate motion data

---

## Input and Output Interfaces

### Inputs to EKF
| Source | ROS Topic | Message Type | Variables Used |
|--------|-----------|--------------|----------------|
| Wheel Odometry (VESC) | `/odom` | `nav_msgs/Odometry` | x, y, yaw (optional), vx, vy (optional), yaw_rate |
| IMU (Artemis) | `/imu/data_raw` | `sensor_msgs/Imu` | yaw_rate, linear_accel_x, linear_accel_y |
| LiDAR Pose | `/slam_out_pose` | `geometry_msgs/PoseStamped` | x, y, yaw |
| Camera Visual Odometry (planned) | `/vo_odom` | `nav_msgs/Odometry` | x, y, yaw |

### Outputs from EKF
| ROS Topic | Message Type | Description |
|-----------|--------------|-------------|
| `/odometry/filtered` | `nav_msgs/Odometry` | Fused pose and velocity in the odom frame |
| `/tf` | TF transform | Transformation from `odom` to `base_link` |

---

## Integration with Other Modules
- **Control** – subscribes to `/odometry/filtered` for precise velocity tracking.
- **Planning** – consumes fused pose and velocity for local trajectory generation.
- **FSM** – uses the estimated state for decision-making and safety logic.
- **Visualization & Debugging** – monitored in RViz and `rqt_plot` for validation.

---

## Architecture Diagram

```mermaid
flowchart TD
  %% Simplified labels for GitHub Mermaid compatibility
  subgraph Sensors
    VESC[VESC]
    IMU[IMU]
    LIDAR[LiDAR]
    CAM[Camera]
  end

  subgraph Preprocessing
    ODOM[/odom/]
    IMUDATA[/imu_data_raw/]
    SCAN[/scan/]
    VO[/vo_odom/]
  end

  subgraph SLAM
    SLAMNODE[SLAM]
    SLAMPOSE[/slam_out_pose/]
  end

  subgraph Fusion
    EKF[EKF]
  end

  subgraph Outputs
    FILTERED[/odometry_filtered/]
    TF[tf odom->base_link]
  end

  VESC --> ODOM
  IMU --> IMUDATA
  LIDAR --> SCAN --> SLAMNODE --> SLAMPOSE
  CAM --> VO

  ODOM --> EKF
  IMUDATA --> EKF
  SLAMPOSE --> EKF
  VO --> EKF

  EKF --> FILTERED
  EKF --> TF
```

---

## Future Extensions

Planned evolution paths (deferred until the baseline EKF fusion is fully validated):

- Unscented Kalman Filter (UKF): We can later trial `ukf_localization_node` to better handle stronger nonlinear vehicle dynamics (slip, aggressive steering). Importantly, it exposes the **same ROS interfaces (topics, frames, message types)** as `ekf_localization_node`, so it is a near drop‑in replacement: the inputs (`/odom`, `/imu/data_raw`, `/slam_out_pose`, optional `/vo_odom`) and outputs (`/odometry/filtered`, TF `odom -> base_link`) remain identical. The migration effort is therefore limited mainly to parameter duplication and retuning (process / measurement covariances, sigma point spread parameters) and—if desired—expanding the state (e.g., to include lateral velocity or accelerometer biases).
- GPS Integration: Add a GNSS (ideally RTK) receiver (`nmea_navsat_driver` + `navsat_transform_node`) to provide low‑rate absolute pose / velocity for drift correction when outdoors.
- Visual Odometry (VO): Leverage the existing RealSense camera (or a dedicated tracking camera) with VO/SLAM (e.g., ORB-SLAM2, RTAB-Map, VINS-Fusion) to publish a `/vo_odom` topic for indoor / GPS-denied operation.

Global Drift Correction via SLAM Fusion: When a SLAM system (LiDAR or visual) publishes a globally referenced pose (e.g., in the `map` frame), we can (a) feed that pose directly as an additional measurement into the existing EKF/UKF instance, or (b) run a two‑stage setup: a local high‑rate EKF/UKF (wheel odom + IMU) producing a smooth short‑term estimate, whose output is then fused with low‑rate global SLAM pose in a second EKF/UKF to realign and bound drift. Because EKF and UKF share identical I/O contracts, either filter can occupy either stage without changing the surrounding nodes—only configuration and covariance tuning differ.

Each added source increases configuration and covariance tuning complexity (time sync, frame alignment, outlier rejection). We will introduce them incrementally once quantitative EKF performance metrics (pose RMSE, innovation consistency, latency) are established.

---

## References

1. Moore, T., & Stouch, D. (2016). A Generalized Extended Kalman Filter Implementation for the Robot Operating System. In: Proceedings of the 13th International Conference on Intelligent Autonomous Systems (IAS-13). DOI: 10.1007/978-3-319-48036-7_9.  
  *Describes the robot_localization package EKF/UKF implementations used in ROS.*

2. ROS Wiki – robot_localization. http://wiki.ros.org/robot_localization  
  *Official documentation of `ekf_localization_node` and `ukf_localization_node`.*

3. F1TENTH Autonomous Racing Project. https://f1tenth.org  
  *Official site describing the F1TENTH platform, its architecture, and racing applications.*

4. Thrun, S., Burgard, W., & Fox, D. (2005). Probabilistic Robotics. MIT Press. ISBN: 978-0262201629.  
  *Standard reference on Bayesian filtering, including EKF and sensor fusion in mobile robots.*

5. Sukkarieh, S., Nebot, E. M., & Durrant-Whyte, H. F. (1999). A high integrity IMU/GPS navigation loop for autonomous land vehicle applications. IEEE Transactions on Robotics and Automation, 15(3), 572–578. DOI: 10.1109/70.768177.  
  *Classical reference for sensor fusion of odometry, IMU, and GPS.*

6. Rokhman, F., & Stouch, D. (2016). The robot_localization Package of ROS. ROSCon 2016. Slides: https://roscon.ros.org/2016/presentations/robot_localization.pdf  
  *Covers usage and tuning of robot_localization in real systems.*

7. Furgale, P., Barfoot, T. D., & Sibley, G. (2013). Continuous-Time Batch Estimation Using Temporal Basis Functions. International Journal of Robotics Research, 32(5), 566–584. DOI: 10.1177/0278364913490328.  
  *Background on factor-graph / smoothing-based alternatives to EKF.*

8. Dellaert, F., & Kaess, M. (2006). Square Root SAM: Simultaneous Localization and Mapping via Square Root Information Smoothing. International Journal of Robotics Research, 25(12), 1181–1203.  
  *Factor-graph-based approach (relevant to the "Future Extensions" section).*

9. Intel RealSense SDK Documentation. https://dev.intelrealsense.com/docs/  
  *For camera-based visual odometry integration.*

10. Hokuyo UST-10LX Laser Scanner Datasheet. https://www.hokuyo-aut.jp/search/single.php?serial=166  
   *Sensor specs (for LiDAR SLAM input).*
