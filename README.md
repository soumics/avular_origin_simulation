# Avular Origin simulation

This repository contains the simulation environment for the Avular Origin One, using Gazebo Fortress and ROS2 Humble.

> **`ros2-jazzy` branch:** this fork's `ros2-jazzy` branch ports the simulation to ROS 2 Jazzy + Gazebo Harmonic
> (Ubuntu 24.04). See [Running on ROS 2 Jazzy](#running-on-ros-2-jazzy-gazebo-harmonic) below and
> [docs/jazzy-compat-notes.md](docs/jazzy-compat-notes.md). The Humble instructions that follow are upstream's.

## Getting started

We currently only support ROS2 Humble on Ubuntu 22.04. For installing ROS2 Humble on your system, refer to the [ROS2 Humble installation guide](https://docs.ros.org/en/humble/Installation.html).
After making sure that ROS2 Humble is installed, install Gazebo Fortress on your system:

```
sudo apt update
sudo apt install ros-humble-ros-gz git-lfs
```

To get started with the Avular Origin Simulation, first make a ROS workspace on your computer (for instructions, see [the ROS tutorial](https://docs.ros.org/en/humble/Tutorials/Beginner-Client-Libraries/Creating-A-Workspace/Creating-A-Workspace.html)).
Next, go to the 'src' folder of your workspace and clone this repository:

```
cd <your-ros-workspace>/src
git clone -b origin_1.0/3 https://github.com/avular-robotics/avular_origin_simulation.git
```

In the same `src` folder, we also need to clone the `avular_origin_description` repository, which contains the URDF and meshes for the Origin One robot:
```
git clone https://github.com/avular-robotics/avular_origin_description.git
cd avular_origin_description
git lfs pull
```

Finally, we can install debian packages for the cmd_vel_controller, which the real Origin One robot uses to prioritize various control command inputs.
This step is optional, but recommended if you want the simulation to behave similar to the real robot, or if you want to use Avular example code. 
To do so, go to the `cmd_vel_controller` folder in this repository and use `apt` to install the binary packages:

```
cd <your-ros-workspace>/src/avular_origin_simulation/cmd_vel_controller/
sudo apt install ./loggercxx_3.3.0_amd64.deb ./rclcpp-avular_3.3.0_amd64.deb ./origin-msgs_2.3.0_amd64.deb ./cmd-vel-controller_1.7.2_amd64.deb ./rclcpp-mocks_3.3.1_amd64.deb
```

Next, go to the top level workspace folder and build the packages in the workspace:
```
cd <your-ros-workspace>
colcon build --symlink-install
```

This will generate a warning that easy_install is deprecated, which can be ignored.

You can now source the workspace (so ROS knows where to find the packages) and launch the simulation environment:
```
source install/setup.bash
ros2 launch origin_one_gazebo ty_test_area.launch.py use_cmd_vel_controller:=True
```
Note that you can set `use_cmd_vel_controller` to `False` if you don't want to use the cmd_vel_controller package.
Alternatively, the simulation can also be launched with the mecanum wheel configuration on the robot:
```
ros2 launch origin_one_gazebo ty_test_area.launch.py use_cmd_vel_controller:=True drive_configuration:=mecanum_drive
```
This will show the avular simulation environment in Gazebo Fortress, with the Origin One spawned.
You can now drag to mouse to change the view to the robot model.
If you want to launch the simulation without sourcing each time, you can source the setup file in your bashrc:
```
echo "source <path_to_workspace>/install/setup.bash" >> ~/.bashrc
```

![image](./doc/img/simulation.png)

## Driving and visualizing the robot in the simulation

The simulation contains gazebo plugins for the drive controller, the Ouster LiDAR and the realsense camera. To visualize the robot and its sensors, you can use the RViz visualization tool. To launch RViz with the correct configuration, run the following command:
```
ros2 launch origin_one_description origin_one_rviz.launch.py
```

This will open RViz with the robot model and the sensor data. You can now drive the robot around in the simulation by publishing Twist messages to the `/robot/cmd_vel` topic. For example by using the `teleop_twist_keyboard` package:
```
sudo apt install ros-humble-teleop-twist-keyboard
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -r /cmd_vel:=/robot/cmd_vel
```


## Running on ROS 2 Jazzy (Gazebo Harmonic)

Tested in `osrf/ros:jazzy-desktop-full` (Gazebo Harmonic 8.11, `ros_gz` 1.0.22) with
`avular_origin_description` `main` (2.0.0) in the same workspace.

```
cd <your-ros-workspace>
colcon build --symlink-install
source install/setup.bash
ros2 launch origin_one_gazebo ty_test_area.launch.py rviz:=true
```

Launch arguments: `headless:=true` (no Gazebo GUI), `rviz:=true` (RViz with robot model, Ouster point
cloud and camera image), `drive_configuration:=mecanum_drive|skid_steer_drive|big_skid_steer_drive|elevated_skid_steer_drive`.
All nodes run with `use_sim_time:=true`.

Drive the robot on `/robot/cmd_vel`. Main topics: `/robot/lidar/points` (Ouster, 1024x32 @ 10 Hz, frame
`os_lidar`), `/robot/camera/color/image_raw`, `/robot/camera/depth/points`, `/robot/odom`, `/robot/gnss/fix_off`.

The Avular `cmd_vel_controller` `.deb`s are Humble-only and are not used on Jazzy (`use_cmd_vel_controller`
defaults to `False`). When installing dependencies, skip the Avular-internal keys:

```
rosdep install --from-paths src --ignore-src -y --skip-keys "cmd_vel_controller ament_copyright_avular"
```

On NVIDIA hybrid-graphics laptops in Docker, run the container with `--gpus=all -e NVIDIA_DRIVER_CAPABILITIES=all`
and set `__EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/10_nvidia.json` (Gazebo server sensors) and
`__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia` (Gazebo GUI, RViz). Otherwise Gazebo falls back to
software rendering and the lidar drops to about 6 Hz.
