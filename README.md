# UR3e ROS 2


## 1. Setup

The PC and robot communicate via the [UR Client Library](https://github.com/UniversalRobots/Universal_Robots_Client_Library).


### 1.1 Robot

Install and set up the URCaps:
1. [`External Control URCap`](https://docs.universal-robots.com/Universal_Robots_ROS_Documentation/rolling/doc/ur_client_library/doc/setup/robot_setup.html#urcap-installation)
2. [`Robotiq Gripper URCap`](https://blog.robotiq.com/knowledge/how-to-install-or-uninstall-a-urcap)


### 1.2 PC

#### Network

Follow the [Network setup](https://docs.universal-robots.com/Universal_Robots_ROS_Documentation/rolling/doc/ur_client_library/doc/setup/network_setup.html) instructions to set up the network communication between the robot and the PC:
1. Connect the robot to one Ethernet interface.
2. Set a static IP for the robot, e.g. `192.168.56.101` with netmask 24.
3. Set a static IP on your host PC interface, e.g. `192.168.56.105`.
4. On the UR tablet: `Installation` (header) -> `URCaps` -> `External Control` -> `Host IP`: set the IP of the PC that controls the robot, e.g. `192.168.56.105`


#### Kernel

Install the low-latency kernel and reboot:
```sh
sudo apt install linux-lowlatency
```

You can also install the real-time `PREEMPT_RT` kernel via `sudo apt install ubuntu-realtime` and set the [real-time permissions](https://docs.universal-robots.com/Universal_Robots_ROS_Documentation/rolling/doc/ur_client_library/doc/real_time.html#setup-user-privileges-to-use-real-time-scheduling). Note that this will conflict with the Nvidia driver and is not strictly necessary to run the UR, but it is recommended to improve the performance.


#### ROS

Install the [Universal_Robots ROS 2 driver](https://github.com/UniversalRobots/Universal_Robots_ROS2_Driver):
```sh
sudo apt install ros-lyrical-ur
```

Install the [`robotiq_2f_urcap_adapter` package](https://github.com/mul-cps/robotiq_2f_urcap_adapter/tree/use_std_msg):
```sh
echo "deb [trusted=yes] https://raw.githubusercontent.com/mul-cps/robotiq_2f_urcap_adapter/resolute-lyrical-amd64/ ./" | sudo tee /etc/apt/sources.list.d/mul-cps_robotiq_2f_urcap_adapter-resolute-lyrical-amd64.list
sudo apt update
sudo apt install ros-lyrical-robotiq-2f-urcap-adapter
```


## 2. Usage


### 2.1 UR PolyScope Tablet
1. boot the UR tablet
2. switch from `Local Control` to `Remote Control` (header right)
3. boot the robot:
    1. click `Power off` (red indicator in footer left)
    2. press `ON`, the robot will boot
    3. the indicator in the bottom left should be yellow (Idle)
    4. press `START` to unlock the breaks
4. at this stage, you can read the robot state via the ROS 2 node, but not control the robot yet

Alternatively, you can use ROS 2 services to start and stop the robot.


### 2.2 joint_trajectory_controller

Start the robot controller:
```sh
. /opt/ros/lyrical/setup.bash
ros2 launch ur_robot_driver ur_control.launch.py \
    ur_type:=ur3e \
    headless_mode:=True \
    launch_dashboard_client:=True \
    launch_rviz:=false \
    robot_ip:=192.168.56.101
```

Since we set `headless_mode:=True`, the `External Control URCap` will start automatically.

With `launch_dashboard_client:=True` we are able to start and stop the robot via service calls:
```sh
# boot robot and release breaks
ros2 service call /dashboard_client/brake_release std_srvs/srv/Trigger {}
# engage breaks and stop robot
ros2 service call /dashboard_client/power_off std_srvs/srv/Trigger {}
```

To send joint trajectories to the robot, the `joint_trajectory_controller` needs to be active.

Check the controllers:
```sh
ros2 control list_controllers
```

This should list the `joint_trajectory_controller` as `active`:
```sh
joint_trajectory_controller         joint_trajectory_controller/JointTrajectoryController            active
```
If the controller is not active or loaded, check that the the robot is running and the tablet is set to `Remote Control`.

Once the connection is established, you can send joint trajectories to the `joint_trajectory_controller`.


### 2.3 MoveIt

The `joint_trajectory_controller` needs to be active in order to control the robot via MoveIt. Alternatively, the robot can be controlled without MoveIt using alternative controllers.

Start MoveIt with RViz:
```sh
ros2 launch ur_moveit_config ur_moveit.launch.py ur_type:=ur3e launch_rviz:=true
```

You are now able to command the robot using the `ur_manipulator` planning group:
1. move the end-effector in RViz to a desired pose
2. click `Plan` to see the planned trajectory visualised in RViz
3. click `Execute` to send the trajectory to the real robot


### 2.4 Gripper

To control the Robotiq 2F-140 gripper, install the `robotiq_2f_urcap_adapter` package: from:
https://github.com/mul-cps/robotiq_2f_urcap_adapter/tree/use_std_msg

and run the instructions there to start the node:
```sh
ros2 launch robotiq_2f_urcap_adapter robotiq_2f140_urcap_adapter_launch.py robot_ip:=192.168.56.101
```
and open/close the gripper:
```sh
# close (width 0 cm)
ros2 action send_goal -f /robotiq_2f140_urcap_adapter/robotiq_2f85_urcap_adapter/gripper_command \
    control_msgs/action/GripperCommand \
    "{ command: { position: 0, max_effort: 125 }}"
# open (with 0.14 cm)
ros2 action send_goal -f /robotiq_2f140_urcap_adapter/robotiq_2f85_urcap_adapter/gripper_command \
    control_msgs/action/GripperCommand \
    "{ command: { position: 0.14, max_effort: 125 }}"
```

The `control_msgs/action/GripperCommand` has two parameters:
1. `position`: the width of the gripper in metre [0, 0.14]
2. `max_effort`: the maximum grasp force in Netwon [10, 125]

Those values must be in the valid range for the gripper, i.e. the Robotiq 2F-140 in this case.


## 3. Calibration

The robot is factory calibration but the ROS 2 node does not use this calibration by default. See https://docs.universal-robots.com/Universal_Robots_ROS2_Documentation/doc/ur_robot_driver/ur_calibration/doc/usage.html on how to extract and use this calibration.

```sh
ros2 launch ur_calibration calibration_correction.launch.py robot_ip:=192.168.56.101 target_filename:="${HOME}/my_robot_calibration.yaml"
```

Store the configuration per serial number and load it via:

```sh
ros2 launch ur_robot_driver ur_control.launch.py ur_type:=ur3e robot_ip:=192.168.56.101 kinematics_params_file:="${HOME}/my_robot_calibration.yaml"
```
