# RN Autonomous - Yahboom ROSMASTER X3 Robot System

A comprehensive ROS 2-based autonomous navigation system for the Yahboom ROSMASTER X3 omnidirectional mobile robot. This system integrates simulation (Gazebo), perception (LIDAR, RGBD Camera, IMU), localization (EKF), and navigation (Nav2) capabilities.

## 🚀 Overview

**rn_autonomous** is a complete robotics stack built on ROS 2 (Jazzy) that enables the ROSMASTER X3 robot to:
- Autonomously navigate using Nav2 navigation framework
- Localize itself using Extended Kalman Filter (EKF) sensor fusion
- Move omnidirectionally with mecanum wheel control
- Perceive the environment with LIDAR and RGBD cameras
- Simulate in Gazebo for development and testing

---

## 📦 System Architecture

```mermaid
graph TB
    subgraph Perception["🎯 Perception Layer"]
        LIDAR["LIDAR Scanner<br/>scan topic"]
        CAMERA["RGBD Camera<br/>cam_1/color/*<br/>cam_1/depth/*"]
        IMU["IMU Sensor<br/>imu/data"]
    end
    
    subgraph Localization["📍 Localization Layer"]
        EKF["EKF Filter Node<br/>robot_localization"]
        AMCL["AMCL<br/>Monte Carlo<br/>Localization"]
        TF["TF: odom → base_link<br/>TF: map → base_link"]
    end
    
    subgraph Navigation["🗺️ Navigation Layer"]
        NAV2["Nav2 Stack"]
        CONTROLLER["Path Controller<br/>MPPI"]
        PLANNER["Global Planner<br/>GridBased"]
    end
    
    subgraph Control["⚙️ Control Layer"]
        MECANUM["Mecanum Drive<br/>Controller"]
        GAZEBO["Gazebo<br/>Simulator"]
    end
    
    subgraph Hardware["🤖 Hardware/Simulation"]
        WHEELS["Wheel Encoders"]
        MOTOR_REAL["Motor Control<br/>Real Robot"]
        MOTOR_SIM["Physics Engine<br/>Simulated"]
    end
    
    LIDAR --> EKF
    CAMERA --> NAV2
    IMU --> EKF
    
    EKF --> TF
    AMCL --> TF
    
    TF --> NAV2
    
    NAV2 --> CONTROLLER
    CONTROLLER --> MECANUM
    PLANNER --> NAV2
    
    MECANUM --> GAZEBO
    MECANUM --> MOTOR_REAL
    
    GAZEBO --> WHEELS
    MOTOR_REAL --> WHEELS
    
    WHEELS --> EKF
```

---

## 📁 Package Structure

### Core Packages

| Package | Purpose | Functionality |
|---------|---------|---------------|
| **rn_autonomous_description** | Robot Definition | URDF/Xacro models, meshes, kinematics |
| **mecanum_drive_controller** | Drive Control | Omnidirectional wheel kinematics, odometry |
| **rn_autonomous_gazebo** | Simulation | Gazebo worlds, physics, sensors |
| **rn_autonomous_bringup** | Startup | Controller loading, initialization scripts |
| **rn_autonomous_localization** | Pose Estimation | EKF sensor fusion, AMCL particle filter |
| **rn_autonomous_navigation** | Path Planning | Nav2 navigation, path controllers |
| **rn_autonomous_system_tests** | Testing | Integration and system tests |
| **rn_autonomus_msgs** | Messages | Custom ROS message definitions |

---

## 🔄 Complete Data Flow

```mermaid
graph LR
    subgraph GazeboSim["Gazebo Simulation"]
        GZ_PHY["Physics Engine"]
        GZ_CAM["Virtual Camera"]
        GZ_LIDAR["Virtual LIDAR"]
        GZ_IMU["Virtual IMU"]
    end
    
    subgraph HardwareSensors["Real Hardware"]
        HW_ENC["Wheel Encoders"]
        HW_LIDAR["RealSense LIDAR"]
        HW_CAM["RealSense Camera"]
        HW_IMU["Real IMU"]
    end
    
    subgraph RosBridge["ROS2←→Gazebo Bridge"]
        BRIDGE["ros_gz_sim bridges"]
    end
    
    subgraph ROS2Nodes["ROS 2 Processing Nodes"]
        EKF_N["EKF Node<br/>Fuses odom + IMU"]
        AMCL_N["AMCL Node<br/>Refines pose"]
        NAV2_N["Nav2<br/>Plans path"]
        CTRL_N["Mecanum Controller<br/>Calculates wheel speeds"]
    end
    
    subgraph Topics["ROS 2 Topics"]
        T_SCAN["/scan<br/>sensor_msgs/LaserScan"]
        T_ODOM["/mecanum_drive_controller/odom<br/>nav_msgs/Odometry"]
        T_IMU["/imu/data<br/>sensor_msgs/Imu"]
        T_FILT_ODOM["/odometry/filtered<br/>nav_msgs/Odometry"]
        T_POSE["/amcl_pose<br/>geometry_msgs/PoseWithCovarianceStamped"]
        T_CMD["/cmd_vel<br/>geometry_msgs/Twist"]
        T_CLOUD["/cam_1/depth/color/points<br/>sensor_msgs/PointCloud2"]
    end
    
    subgraph SpinAndOutput["Spin and Command Output"]
        CMD_PUB["Publish cmd_vel"]
        WHL_CMD["Wheel Velocity<br/>Commands"]
    end
    
    GZ_LIDAR -->|lazy bridge| BRIDGE
    GZ_CAM -->|lazy bridge| BRIDGE
    GZ_IMU -->|eager bridge| BRIDGE
    
    HW_LIDAR -->|ROS driver| BRIDGE
    HW_CAM -->|ROS driver| BRIDGE
    HW_IMU -->|ROS driver| BRIDGE
    
    BRIDGE -->|T_SCAN| T_SCAN
    BRIDGE -->|T_CLOUD| T_CLOUD
    BRIDGE -->|T_IMU| T_IMU
    
    T_SCAN --> AMCL_N
    T_CLOUD --> NAV2_N
    
    HW_ENC -->|T_ODOM| T_ODOM
    GZ_PHY -->|T_ODOM| T_ODOM
    
    T_ODOM --> EKF_N
    T_IMU --> EKF_N
    EKF_N -->|T_FILT_ODOM| T_FILT_ODOM
    
    T_FILT_ODOM --> AMCL_N
    T_FILT_ODOM --> NAV2_N
    AMCL_N -->|T_POSE| T_POSE
    
    NAV2_N -->|T_CMD| T_CMD
    T_CMD --> CTRL_N
    CTRL_N --> CMD_PUB
    CMD_PUB --> WHL_CMD
    
    WHL_CMD -->|Bridge| BRIDGE
    WHL_CMD -->|Hardware| GZ_PHY
    
    style GazeboSim fill:#e1f5ff
    style HardwareSensors fill:#fff3e0
    style ROS2Nodes fill:#f3e5f5
    style Topics fill:#e8f5e9
```

---

## 🎮 Quick Start

### Prerequisites
```bash
# ROS 2 Jazzy installed
source /opt/ros/jazzy/setup.bash

# Clone workspace
git clone https://github.com/Rishabh-spec-ux/rn_autonomous.git
cd rn_autonomous
```

### Build
```bash
# Install dependencies
rosdep install --from-paths . --ignore-src -r -y

# Build packages
colcon build --symlink-install

# Source setup
source install/setup.bash
```

### Launch Simulation
```bash
# Option 1: Gazebo Simulation with Navigation
ros2 launch rn_autonomous_gazebo rn_autonomous.gazebo.launch.py world_file:=cafe.world

# Option 2: Load Controllers
ros2 launch rn_autonomous_bringup load_ros2_controllers.launch.py

# Option 3: Start Navigation
ros2 launch rn_autonomous_navigation rosmaster_x3_navigation.launch.py use_sim_time:=true
```

### Send Navigation Goals
```bash
# Use RViz to set goal through Nav2 panel
# Or use CLI:
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "pose: { header: { frame_id: \"map\" }, pose: { position: { x: 1.0, y: 1.0 } } }" \
  --feedback
```

---

## 🔌 Key Topics and Transforms

### Sensor Topics
| Topic | Type | Source | Frequency |
|-------|------|--------|-----------|
| `/scan` | LaserScan | LIDAR | 15 Hz |
| `/imu/data` | Imu | IMU | 15 Hz |
| `/cam_1/color/image_raw` | Image | RGBD Camera | 30 Hz |
| `/cam_1/depth/color/points` | PointCloud2 | RGBD Camera | 30 Hz |

### Command Topics
| Topic | Type | Frequency | Purpose |
|-------|------|-----------|---------|
| `/cmd_vel` | Twist | 10 Hz | Navigation velocity commands |
| `/mecanum_drive_controller/cmd_vel` | TwistStamped | 10 Hz | Raw wheel velocity commands |

### State Topics
| Topic | Type | Source | Purpose |
|-------|------|--------|---------|
| `/odometry/filtered` | Odometry | EKF | Fused pose estimate |
| `/amcl_pose` | PoseWithCovariance | AMCL | Globally localized pose |
| `/mecanum_drive_controller/odom` | Odometry | Controller | Raw encoder odometry |

### Transform Tree (TF)
```
map
  ├─> odom (from AMCL)
  │    └─> base_footprint (from EKF)
  │         └─> base_link
  │            ├─> camera_link
  │            ├─> lidar_link
  │            ├─> imu_link
  │            ├─> wheel_front_left_link
  │            ├─> wheel_front_right_link
  │            ├─> wheel_back_left_link
  │            └─> wheel_back_right_link
```

---

## 🛠️ Configuration Files

### Navigation Config
- `rn_autonomous_navigation/config/rosmaster_x3_nav2_default_params.yaml`
  - AMCL parameters for particle filter localization
  - MPPI controller parameters for path following
  - Costmap configuration

### Localization Config
- `rn_autonomous_localization/config/ekf.yaml`
  - EKF filter setup for sensor fusion
  - Odom and IMU input configuration
  - Frame definitions

### Bridge Config
- `rn_autonomous_gazebo/config/ros_gz_bridge.yaml`
  - Gazebo ↔ ROS 2 topic mappings
  - Sensor data bridging configuration

---

## 📊 System Parameters

### Robot Dimensions
| Parameter | Value | Unit |
|-----------|-------|------|
| Wheel Radius | 0.0325 | m |
| Wheel Base (L) | 0.120 | m |
| Wheel Separation (W) | 0.1665 | m |
| Base Mass | 4.6 | kg |
| Max Velocity (V_max) | 0.5 | m/s |
| Max Angular Velocity | 1.9 | rad/s |

### Controller Parameters
| Parameter | Value | Function |
|-----------|-------|----------|
| Controller Frequency | 5.0 | Hz |
| EKF Update Rate | 15.0 | Hz |
| LIDAR Frequency | 15 | Hz |
| Camera Frequency | 30 | Hz |

---

## 🚨 Troubleshooting

### Robot Spawning Inside Ground
**Problem**: Robot appears underground in Gazebo
```yaml
# Solution: Increase spawn height in launch file
declare_z_cmd = DeclareLaunchArgument(
    name='z',
    default_value='0.15',  # Increased from 0.05
    description='z component of initial position, meters')
```

### No Odometry Data
- Check encoder connections on real robot
- Verify Gazebo physics are running
- Inspect `/mecanum_drive_controller/odom` topic

### Navigation Not Working
- Ensure map is loaded/created
- Check AMCL is converging (monitor `/amcl_pose`)
- Verify `/odometry/filtered` is publishing

### TF Tree Broken
- Confirm `robot_state_publisher` is running
- Check `/tf` and `/tf_static` topics have data
- Use `tf2_tools.py` to visualize tree

---

## 📚 Detailed Package Documentation

See individual package README files:
- [mecanum_drive_controller README](src/rn_autonomous/mecanum_drive_controller/README.md)
- [rn_autonomous_description README](src/rn_autonomous/rn_autonomous_description/README.md)
- [rn_autonomous_gazebo README](src/rn_autonomous/rn_autonomous_gazebo/README.md)
- [rn_autonomous_bringup README](src/rn_autonomous/rn_autonomous_bringup/README.md)
- [rn_autonomous_localization README](src/rn_autonomous/rn_autonomous_localization/README.md)
- [rn_autonomous_navigation README](src/rn_autonomous/rn_autonomous_navigation/README.md)

---

## 📝 License

BSD-3-Clause License

## 🤝 Contributing

Contributions welcome! Please follow ROS 2 coding standards and submit pull requests.

---

**Last Updated**: March 2026  
**ROS 2 Version**: Jazzy  
**Robot Model**: Yahboom ROSMASTER X3
