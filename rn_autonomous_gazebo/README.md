# RN Autonomous Gazebo Package

Gazebo simulation environment for the Yahboom ROSMASTER X3 robot. Provides physics simulation, sensor simulation, world definitions, and ROS 2 integration through the Gazebo bridge system.

## 🎮 Purpose

This package enables:
- **Physics Simulation**: Realistic robot dynamics with ODE physics engine
- **Sensor Simulation**: Virtual LIDAR, RGBD camera, and IMU sensors
- **Development Testing**: Full navigation stack testing without hardware
- **ROS 2 Integration**: Seamless topic bridging between Gazebo and ROS 2
- **Multiple Environments**: Different world files (empty, cafe, house, etc.)

---

## 🌍 World Files

```
worlds/
├── empty.world           # Minimal environment for testing
├── cafe.world            # Realistic cafe with tables and obstacles
├── house.world           # Indoor house environment
└── pick_and_place.world  # Demo environment
```

### World Structure Example (cafe.world)

```xml
<!-- Physics Engine -->
<plugin name="gz::sim::systems::Physics">
  <!-- ODE solver configuration -->
</plugin>

<!-- User Command System -->
<plugin name="gz::sim::systems::UserCommands"/>

<!-- Scene Updates -->
<plugin name="gz::sim::systems::SceneBroadcaster"/>

<!-- Sensor System -->
<plugin name="gz::sim::systems::Sensors">
  <render_engine>ogre2</render_engine>
</plugin>

<!-- Lighting -->
<light type="directional" name="sun">
  <pose>0 0 10 0 0 0</pose>
  <intensity>1</intensity>
</light>

<!-- Ground Plane -->
<model name="ground_plane">
  <static>true</static>
  <link name="link">
    <collision>
      <geometry>
        <plane><normal>0 0 1</normal></plane>
      </geometry>
    </collision>
    <visual>...</visual>
  </link>
</model>

<!-- Scene Objects -->
<include><uri>model://cafe</uri></include>
<include><uri>model://cafe_table</uri></include>
```

---

## 🔄 Gazebo ↔ ROS 2 Bridge Architecture

```mermaid
graph TB
    subgraph Gazebo["Gazebo Simulator"]
        GZ_PHY["Physics Engine<br/>World State"]
        GZ_PATH["Sensor Plugins<br/>Camera, LIDAR, IMU"]
        GZ_WORLD["World Models<br/>Robot, Objects"]
    end
    
    subgraph GZTopics["Gazebo Topics<br/>/model/*, /sensor/*"]
        GZ_LIDAR_T["scan"]
        GZ_CAM_T["camera_info, image_raw"]
        GZ_IMU_T["imu/data"]
        GZ_CLOCK["clock"]
    end
    
    subgraph Bridge["ros_gz_bridge<br/>Topic Mapping"]
        LAZY_MAP["Lazy Bridge<br/>On-demand connection"]
        EAGER_MAP["Eager Bridge<br/>Always connected"]
        CLOCK_MAP["Clock Sync"]
    end
    
    subgraph ROS2Topics["ROS 2 Topics"]
        ROS_SCAN["/scan<br/>LaserScan"]
        ROS_CAM["/cam_1/color/*<br/>Image"]
        ROS_IMU["/imu/data<br/>Imu"]
        ROS_CLOCK["/clock"]
    end
    
    subgraph ROS2Stack["ROS 2 Stack"]
        AMCL["AMCL<br/>Localization"]
        NAV2["Nav2<br/>Navigation"]
        EKF["EKF Filter"]
    end
    
    subgraph ROS2Control["ROS 2 Control"]
        CMD_PUB["cmd_vel Publisher"]
        CTRL["Mecanum<br/>Controller"]
    end
    
    GZ_WORLD --> GZ_PATH
    GZ_PATH --> GZ_LIDAR_T
    GZ_PATH --> GZ_CAM_T
    GZ_PATH --> GZ_IMU_T
    GZ_PHY --> GZ_IMU_T
    
    GZ_LIDAR_T -->|Lazy| LAZY_MAP
    GZ_CAM_T -->|Lazy| LAZY_MAP
    GZ_IMU_T -->|Eager| EAGER_MAP
    GZ_CLOCK -->|Eager| CLOCK_MAP
    
    LAZY_MAP --> ROS_SCAN
    LAZY_MAP --> ROS_CAM
    EAGER_MAP --> ROS_IMU
    CLOCK_MAP --> ROS_CLOCK
    
    ROS_SCAN --> AMCL
    ROS_CAM --> NAV2
    ROS_IMU --> EKF
    
    NAV2 --> CMD_PUB
    CMD_PUB --> CTRL
    
    CTRL --> Bridge
    Bridge -->|Velocity Command| GZ_PHY
    GZ_PHY -->|Updates pose| GZ_WORLD
    
    style Gazebo fill:#b3e5fc
    style GZTopics fill:#e1f5ff
    style Bridge fill:#f3e5f5
    style ROS2Topics fill:#e8f5e9
    style ROS2Stack fill:#fff9c4
```

---

## 🌉 ROS 2 ↔ Gazebo Bridge Configuration

**File**: `config/ros_gz_bridge.yaml`

### Bridge Entries

```yaml
# RGBD Camera - Lazy Bridge (starts on subscriber demand)
- ros_topic_name: "cam_1/color/camera_info"
  gz_topic_name: "cam_1/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msgs.CameraInfo"
  direction: GZ_TO_ROS
  lazy: true

# Point Cloud - Lazy Bridge
- ros_topic_name: "cam_1/depth/color/points"
  gz_topic_name: "cam_1/points"
  ros_type_name: "sensor_msgs/msg/PointCloud2"
  gz_type_name: "gz.msgs.PointCloudPacked"
  direction: GZ_TO_ROS
  lazy: true

# LIDAR Scan - Eager Bridge (always connected)
- ros_topic_name: "scan"
  gz_topic_name: "scan"
  ros_type_name: "sensor_msgs/msg/LaserScan"
  gz_type_name: "gz.msgs.LaserScan"
  direction: GZ_TO_ROS
  lazy: false

# IMU Data - Eager Bridge
- ros_topic_name: "imu/data"
  gz_topic_name: "imu/data"
  ros_type_name: "sensor_msgs/msg/Imu"
  gz_type_name: "gz.msgs.IMU"
  direction: GZ_TO_ROS
  lazy: false

# Velocity Commands - ROS to Gazebo
- ros_topic_name: "mecanum_drive_controller/cmd_vel"
  gz_topic_name: "mecanum_drive_controller/cmd_vel"
  ros_type_name: "geometry_msgs/msg/TwistStamped"
  gz_type_name: "gz.msgs.Twist"
  direction: ROS_TO_GZ
  lazy: false

# Clock Sync - Essential for sim_time
- ros_topic_name: "clock"
  gz_topic_name: "clock"
  ros_type_name: "rosgraph_msgs/msg/Clock"
  gz_type_name: "gz.msgs.Clock"
  direction: GZ_TO_ROS
  lazy: false
```

### Bridge Connection Types

| Type | Purpose | Use Case |
|------|---------|----------|
| **Lazy** | On-demand connection | Sensors used intermittently (camera) |
| **Eager** | Always connected | Critical sensors (LIDAR, IMU, clock) |
| **GZ_TO_ROS** | Gazebo publishes to ROS | Sensor data flow |
| **ROS_TO_GZ** | ROS publishes to Gazebo | Command flow |

---

## 🚀 Launch System

**File**: `launch/rn_autonomous.gazebo.launch.py`

### Launch Flow

```mermaid
graph TD
    PARAMS["Parse Launch<br/>Arguments"]
    PKG_FIND["Locate Package<br/>Paths & Files"]
    ENV["Set Environment<br/>GZ_SIM_RESOURCE_PATH"]
    RSP["Launch Robot State<br/>Publisher"]
    CTRL["Load Controllers<br/>ROS 2 Control"]
    GZ_SRV["Start Gazebo<br/>Server -r -s"]
    GZ_GUI["Start Gazebo<br/>GUI Client"]
    BRIDGE["Launch ROS-GZ<br/>Bridge"]
    IMG_BRIDGE["Launch Image<br/>Bridge"]
    SPAWN["Spawn Robot in<br/>Gazebo"]
    
    PARAMS --> PKG_FIND
    PKG_FIND --> ENV
    ENV --> RSP
    RSP --> CTRL
    CTRL --> GZ_SRV
    GZ_SRV --> GZ_GUI
    GZ_GUI --> BRIDGE
    BRIDGE --> IMG_BRIDGE
    IMG_BRIDGE --> SPAWN
    
    style SPAWN fill:#fff9c4
    style GZ_SRV fill:#b3e5fc
```

### Launch Arguments

| Argument | Default | Type | Purpose |
|----------|---------|------|---------|
| `world_file` | `empty.world` | string | Which world file to load |
| `headless` | `False` | bool | Disable Gazebo GUI |
| `use_sim_time` | `true` | bool | Use Gazebo clock |
| `use_rviz` | `true` | bool | Launch RViz visualization |
| `use_robot_state_pub` | `true` | bool | Publish robot TF |
| `load_controllers` | `true` | bool | Load ROS2 controllers |
| `robot_name` | `rosmaster_x3` | string | Robot model name |

### Spawn Position Arguments

| Argument | Default | Unit | Purpose |
|----------|---------|------|---------|
| `x` | `0.0` | m | X position in world |
| `y` | `0.0` | m | Y position in world |
| `z` | `0.05` | m | Z position (height above ground) |
| `roll` | `0.0` | rad | Roll orientation |
| `pitch` | `0.0` | rad | Pitch orientation |
| `yaw` | `0.0` | rad | Yaw orientation |

---

## 🎟️ Usage Examples

### Basic Simulation - Empty World

```bash
# Terminal 1: Launch Gazebo with robot
ros2 launch rn_autonomous_gazebo rn_autonomous.gazebo.launch.py \
  world_file:=empty.world

# Terminal 2: Start navigation
ros2 launch rn_autonomous_navigation rosmaster_x3_navigation.launch.py \
  use_sim_time:=true

# Terminal 3: Send goal
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "pose: {header: {frame_id: 'map'}, pose: {position: {x: 1.0, y: 1.0}}}"
```

### Cafe World with Obstacles

```bash
ros2 launch rn_autonomous_gazebo rn_autonomous.gazebo.launch.py \
  world_file:=cafe.world \
  use_rviz:=true
```

### Headless Simulation (No GUI)

```bash
ros2 launch rn_autonomous_gazebo rn_autonomous.gazebo.launch.py \
  world_file:=cafe.world \
  headless:=true \
  use_rviz:=true
```

### Custom Spawn Position

```bash
ros2 launch rn_autonomous_gazebo rn_autonomous.gazebo.launch.py \
  x:=2.0 \
  y:=3.0 \
  z:=0.15 \
  yaw:=1.57  # 90 degrees
```

---

## 📊 Sensor Simulation Details

### LIDAR Specification

```
Type: GPU-based ray-casting
Horizontal FOV: 360° (full circle)
Samples per scan: 360
Range: 0.27 - 120 m
Update rate: 15 Hz
```

### RGBD Camera Specification

```
Type: RealSense simulation
Resolution: 640 × 480 pixels
Horizontal FOV: 60°
Depth range: 0.1 - 10 m
RGB frame rate: 30 Hz
Depth frame rate: 30 Hz
```

### IMU Specification

```
Type: 6-DOF IMU
Angular velocity noise: σ = 0.009 rad/s
Linear acceleration noise: σ = 0.021 m/s²
Update rate: 15 Hz
```

---

## 🔌 Topic Flow During Runtime

```mermaid
graph LR
    ROS["ROS 2<br/>Nodes"]
    CMD["Publish<br/>cmd_vel"]
    BRIDGE["Bridge"]
    GZ_CMD["Gazebo<br/>cmd_vel"]
    PHY["Physics<br/>Update"]
    SEN["Sensor<br/>Update"]
    GZ_SCAN["Gazebo<br/>scan"]
    BRIDGE2["Bridge"]
    ROS_SCAN["ROS<br/>scan"]
    
    ROS -->|10 Hz| CMD
    CMD -->|Bridge| BRIDGE
    BRIDGE -->|TwistStamped| GZ_CMD
    GZ_CMD --> PHY
    PHY --> SEN
    SEN -->|15 Hz| GZ_SCAN
    GZ_SCAN -->|Bridge| BRIDGE2
    BRIDGE2 -->|LaserScan| ROS_SCAN
    ROS_SCAN --> ROS
    
    style ROS fill:#e8f5e9
    style PHY fill:#b3e5fc
    style SEN fill:#f8bbd0
```

---

## 🛠️ Configuration Files

| File | Purpose |
|------|---------|
| `config/ros_gz_bridge.yaml` | Topic bridging configuration |
| `launch/rn_autonomous.gazebo.launch.py` | Main simulation launcher |
| `models/` | Custom Gazebo models |
| `worlds/` | Simulation environment files |

---

## 🔧 Gazebo Environment Variables

```bash
# Gazebo model path (includes our models)
export GZ_SIM_RESOURCE_PATH=$GZ_SIM_RESOURCE_PATH:$(ros2 pkg prefix rn_autonomous_gazebo)/models

# Gazebo plugin path
export GZ_SIM_SYSTEM_PLUGIN_PATH=$GZ_SIM_SYSTEM_PLUGIN_PATH:/usr/lib/x86_64-linux-gnu/gz-sim-8/plugins

# Verbose output
export GZ_SIM_LOG_LEVEL=3
```

---

## ⚠️ Troubleshooting

### Robot Spawning Inside Ground

**Problem**: Robot appears underground or intersecting with ground plane

**Solution**: Increase spawn Z position
```bash
ros2 launch rn_autonomous_gazebo rn_autonomous.gazebo.launch.py z:=0.15
```

### No Sensor Data

**Problem**: Sensor topics not receiving data

**Solution**: Check bridge configuration and lazy settings
```bash
# Verify topics exist
ros2 topic list
ros2 topic echo /scan
```

### Physics Unstable

**Problem**: Robot bouncing, oscillating motion

**Solution**: Adjust physics parameters
```yaml
# In launch file:
max_step_size: 0.0005  # Reduce from 0.001
```

### Gazebo Crashes

**Problem**: Gazebo exits unexpectedly

**Solution**: Check GPU resources
```bash
# Run headless (CPU only)
ros2 launch rn_autonomous_gazebo rn_autonomous.gazebo.launch.py \
  headless:=true
```

---

## 🎛️ Performance Optimization

### For CPU-Limited Systems

```bash
ros2 launch rn_autonomous_gazebo rn_autonomous.gazebo.launch.py \
  headless:=true \
  world_file:=empty.world
```

### For Better Physics Accuracy

```yaml
# Decrease step size and increase update rate
max_step_size: 0.0005
real_time_update_rate: 2000.0
```

---

## 📁 Directory Structure

```
rn_autonomous_gazebo/
├── launch/
│   └── rn_autonomous.gazebo.launch.py
├── worlds/
│   ├── empty.world
│   ├── cafe.world
│   ├── house.world
│   └── pick_and_place_demo.world
├── models/
│   ├── cafe/
│   ├── cafe_table/
│   └── house/
├── config/
│   └── ros_gz_bridge.yaml
├── rviz/
│   └── rn_autonomous_gazebo_sim.rviz
└── README.md (this file)
```

---

## 🔗 Integration Points

**Used with**:
- **rn_autonomous_description**: Provides robot model
- **mecanum_drive_controller**: Sends wheel commands
- **rn_autonomous_bringup**: Loads controller configs
- **rn_autonomous_navigation**: Navigation testing environment
- **rn_autonomous_localization**: Sensor fusion testing

---

## 📝 License

BSD-3-Clause License

---

**Last Updated**: March 2026  
**ROS 2 Version**: Jazzy  
**Gazebo Version**: Garden/Harmonic  
**Robot Model**: Yahboom ROSMASTER X3
