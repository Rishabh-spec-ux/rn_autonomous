# rn_autonomous
![OS](https://img.shields.io/ubuntu/v/ubuntu-wallpapers/noble)
![ROS_2](https://img.shields.io/ros/v/jazzy/rclcpp)



Quick start
- Clone into a ROS 2 workspace (colcon build compatible)
- Source your ROS 2 distro and workspace:
  - source /opt/ros/<distro>/setup.bash
  - colcon build --symlink-install
  - source install/setup.bash
- Launch available demos and controllers with the package launch files (see package README files for specifics).