# 260724

## sources

1. [moveit install tutorial](http://web.archive.org/web/20230217230652/http://sdk.rethinkrobotics.com/intera/MoveIt_Tutorial)
2. [docker network limitation](https://docs.docker.com/engine/security/rootless/troubleshoot/#until-docker-engine-v295)

## log

### goal: ping sawyer

- the other pc uses wifi to connect to the network switch that sawyer is connected to
- did a functional of moveit with that pc
- this proves that the switch and sawyer work
- swayer ip: `192.168.0.103`
- lambda ip: `192.168.0.20`

#### checking `ip a`

```
Every 2.0s: ip a                                                    depend: Fri Jul 24 10:26:09 2026

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp37s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 08:bf:b8:86:8f:46 brd ff:ff:ff:ff:ff:ff
    inet 10.210.22.197/24 brd 10.210.22.255 scope global dynamic noprefixroute enp37s0f0
       valid_lft 14155sec preferred_lft 14155sec
    inet6 fe80::3426:5a9e:731e:a120/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: enp37s0f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 08:bf:b8:86:8f:47 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::adfe:59df:1750:902a/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
4: wlp39s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen
 1000
    link/ether 70:32:17:53:ea:8f brd ff:ff:ff:ff:ff:ff
5: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none
    inet 192.168.2.1/24 scope global wg0
       valid_lft forever preferred_lft forever
6: br-52131b65dbfe: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group defa
ult
    link/ether 26:5b:79:0b:56:9a brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.1/16 brd 172.19.255.255 scope global br-52131b65dbfe
       valid_lft forever preferred_lft forever
7: br-def0021e0140: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group defa
ult
    link/ether ea:77:a6:03:79:5f brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-def0021e0140
       valid_lft forever preferred_lft forever
8: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 4a:03:8e:14:e2:e2 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

- we can `watch ip a` and plug and unplug the ethernet connection from lambda to switch
- `enp37s0f1` goes from DOWN to UP when we do that, so we know that is our ethernet connection
- notice it only has a link local ipv6 address: `fe80::adfe:59df:1750:902a`, no ipv4 address
- this is kind of similar to robotics: we need to set the static ipv4 address of the connection/interface

#### nmcli

```
peilunli@depend:~$ nmcli connection show
NAME                UUID                                  TYPE       DEVICE
Wired connection 1  0860a249-4581-3c94-80ee-2d73f7ccd03b  ethernet   enp37s0f0
docker0             b2f93591-ca12-47de-8322-ee8360f1e4cd  bridge     docker0
br-52131b65dbfe     ad88f48f-c0b7-4c6f-a94f-bedb4a17b921  bridge     br-52131b65dbfe
br-def0021e0140     6571342c-3996-4e80-89c0-b48f02a5543d  bridge     br-def0021e0140
wg0                 8ef71199-c647-46ad-80b7-92f0a567fb73  wireguard  wg0
Wired connection 2  c383fe71-a02f-3945-a28e-3ecfaca9b806  ethernet   --
peilunli@depend:~$ nmcli device status
DEVICE           TYPE       STATE                   CONNECTION
enp37s0f0        ethernet   connected               Wired connection 1
docker0          bridge     connected (externally)  docker0
br-52131b65dbfe  bridge     connected (externally)  br-52131b65dbfe
br-def0021e0140  bridge     connected (externally)  br-def0021e0140
wg0              wireguard  connected (externally)  wg0
wlp39s0          wifi       disconnected            --
p2p-dev-wlp39s0  wifi-p2p   disconnected            --
enp37s0f1        ethernet   unavailable             --
lo               loopback   unmanaged               --
```

- our target interface is likely Wired connection 2.
- we can use nmcli to set the static ip of this interface
- can't use user settings, it appears network stuff is pretty locked down

```
sudo nmcli connection modify "Wired connection 2" \
  ipv4.method manual \
  ipv4.addresses 192.168.0.20/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes \
  ipv6.method disabled

sudo nmcli connection down "Wired connection 2"
sudo nmcli connection up "Wired connection 2"
```

#### ethernet port

- i played around with the port the lambda was plugged into. turns out it can be plugged into all of the "ethernet" ports, but the "internet" port does not work, it likely serves a different purpose
- this was confusing because the other pc could only be plugged into the internet port
- anyway, now we could `ping 192.168.0.103` from both outside and inside docker

### `/etc/hosts`

```
root@depend:~/ros_ws# rosnode info /realtime_loop
--------------------------------------------------------------------------------
Node [/realtime_loop]
Publications:
 * /diagnostics [diagnostic_msgs/DiagnosticArray]
 * /errors [rethink_errors/Error]
 * /io/robot/config [intera_core_msgs/IONodeConfiguration]
 * /io/robot/cuff/config [intera_core_msgs/IODeviceConfiguration]
 * /io/robot/cuff/state [intera_core_msgs/IODeviceStatus]
 * /io/robot/navigator/config [intera_core_msgs/IODeviceConfiguration]
 * /io/robot/navigator/state [intera_core_msgs/IODeviceStatus]
 * /io/robot/pneumatic/config [intera_core_msgs/IODeviceConfiguration]
 * /io/robot/pneumatic/state [intera_core_msgs/IODeviceStatus]
 * /io/robot/right_end_of_arm/config [intera_core_msgs/IODeviceConfiguration]
 * /io/robot/right_end_of_arm/state [intera_core_msgs/IODeviceStatus]
 * /io/robot/robot/config [intera_core_msgs/IODeviceConfiguration]
 * /io/robot/robot/state [intera_core_msgs/IODeviceStatus]
 * /io/robot/safety/config [intera_core_msgs/IODeviceConfiguration]
 * /io/robot/safety/state [intera_core_msgs/IODeviceStatus]
 * /io/robot/state [intera_core_msgs/IONodeStatus]
 * /robot/accelerometer/right_accelerometer/state [sensor_msgs/Imu]
 * /robot/accelerometer_names [motor_control_msgs/StringArray]
 * /robot/accelerometer_states [motor_control_msgs/AccelerometerStates]
 * /robot/analog_io/head_wheel/state [intera_core_msgs/AnalogIOState]
 * /robot/analog_io/head_wheel/value_uint32 [std_msgs/UInt32]
 * /robot/analog_io/right_analog_input/state [intera_core_msgs/AnalogIOState]
 * /robot/analog_io/right_analog_input/value_uint32 [std_msgs/UInt32]
 * /robot/analog_io/right_vacuum_sensor_analog/state [intera_core_msgs/AnalogIOState]
 * /robot/analog_io/right_vacuum_sensor_analog/value_uint32 [std_msgs/UInt32]
 * /robot/analog_io/right_wheel/state [intera_core_msgs/AnalogIOState]
 * /robot/analog_io/right_wheel/value_uint32 [std_msgs/UInt32]
 * /robot/analog_io/torso_fan/state [intera_core_msgs/AnalogIOState]
 * /robot/analog_io/torso_fan/value_uint32 [std_msgs/UInt32]
 * /robot/analog_io/torso_lighting/state [intera_core_msgs/AnalogIOState]
 * /robot/analog_io/torso_lighting/value_uint32 [std_msgs/UInt32]
 * /robot/analog_io_names [motor_control_msgs/StringArray]
 * /robot/analog_io_states [intera_core_msgs/AnalogIOStates]
 * /robot/camera_strobe [intera_core_msgs/CameraStrobe]
 * /robot/digital_io/head_blue_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_back/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_back_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_circle/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_circle_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_ok/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_sawyer/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_sawyer_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_square/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_square_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_triangle/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_button_triangle_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_green_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/head_red_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_back/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_back_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_circle/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_circle_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_ok/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_sawyer/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_sawyer_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_square/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_square_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_triangle/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_button_triangle_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_hand_blue_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_hand_camera_power/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_hand_green_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_hand_red_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_inner_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_lower_button/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_lower_cuff/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_outer_light/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_shoulder_button/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_upper_button/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_valve_1a/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_valve_1b/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_valve_2a/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/right_valve_2b/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/torso_camera_power/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/torso_digital_input0/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/torso_foot_pedal/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/torso_process_sense0/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/torso_process_sense1/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io/torso_safety_stop/state [intera_core_msgs/DigitalIOState]
 * /robot/digital_io_names [motor_control_msgs/StringArray]
 * /robot/digital_io_states [intera_core_msgs/DigitalIOStates]
 * /robot/end_effector_names [motor_control_msgs/StringArray]
 * /robot/head/head_state [intera_core_msgs/HeadState]
 * /robot/homing_names [motor_control_msgs/StringArray]
 * /robot/homing_states [intera_core_msgs/HomingState]
 * /robot/in_sim_mode [std_msgs/Bool]
 * /robot/joint_limits [intera_core_msgs/JointLimits]
 * /robot/joint_names [motor_control_msgs/StringArray]
 * /robot/joint_states [sensor_msgs/JointState]
 * /robot/limb/right/arm_state_estimate [motor_control_msgs/ArmStateEstimate]
 * /robot/limb/right/attached_mass [std_msgs/Float32]
 * /robot/limb/right/collision_avoidance_state [intera_core_msgs/CollisionAvoidanceState]
 * /robot/limb/right/collision_detection_state [intera_core_msgs/CollisionDetectionState]
 * /robot/limb/right/commanded_endpoint_state [intera_core_msgs/EndpointState]
 * /robot/limb/right/endpoint_state [intera_core_msgs/EndpointState]
 * /robot/limb/right/gravity_compensation_torques [intera_core_msgs/SEAJointState]
 * /robot/limb/right/interaction_controller/state [intera_core_msgs/InteractionControlState]
 * /robot/limb/right/joint_tracking_error [intera_motion_msgs/JointTrackingError]
 * /robot/limb/right/limb_state [motor_control_msgs/LimbState]
 * /robot/limb/right/pose_metric_info [motor_control_msgs/PoseMetrics]
 * /robot/limb/right/stiffness [motor_control_msgs/Stiffness]
 * /robot/limb/right/stiffness_constraint [motor_control_msgs/StiffnessConstraint]
 * /robot/limb/right/suppress_squish_safety [std_msgs/Empty]
 * /robot/limb/right/tip_states [intera_core_msgs/EndpointStates]
 * /robot/limb/right/torque_data [motor_control_msgs/TorqueData]
 * /robot/limb/right/twist_speed_constraint [motor_control_msgs/SpeedConstraint]
 * /robot/limb/right/velocity_controller_state [motor_control_msgs/VelocityControllerState]
 * /robot/limb_names [motor_control_msgs/StringArray]
 * /robot/navigators/head_navigator/state [intera_core_msgs/NavigatorState]
 * /robot/navigators/right_navigator/state [intera_core_msgs/NavigatorState]
 * /robot/navigators_names [motor_control_msgs/StringArray]
 * /robot/navigators_states [intera_core_msgs/NavigatorStates]
 * /robot/ref_joint_names [motor_control_msgs/StringArray]
 * /robot/ref_joint_states [sensor_msgs/JointState]
 * /robot/state [intera_core_msgs/RobotAssemblyState]
 * /rosout [rosgraph_msgs/Log]

Subscriptions:
 * /collision/right/collision_detection [motor_control_msgs/CollisionDetection]
 * /engine/task_state [motor_control_msgs/TaskState]
 * /intera/endpoint_ids [motor_control_msgs/StringArray]
 * /io/internal_camera/right_hand_camera/set_strobe/state [std_msgs/Bool]
 * /io/robot/command [intera_core_msgs/IOComponentCommand]
 * /io/robot/cuff/command [intera_core_msgs/IOComponentCommand]
 * /io/robot/navigator/command [intera_core_msgs/IOComponentCommand]
 * /io/robot/pneumatic/command [intera_core_msgs/IOComponentCommand]
 * /io/robot/right_end_of_arm/command [intera_core_msgs/IOComponentCommand]
 * /io/robot/robot/command [intera_core_msgs/IOComponentCommand]
 * /io/robot/safety/command [intera_core_msgs/IOComponentCommand]
 * /robot/analog_io/command [intera_core_msgs/AnalogOutputCommand]
 * /robot/digital_io/command [intera_core_msgs/DigitalOutputCommand]
 * /robot/head/command_head_pan [intera_core_msgs/HeadPanCommand]
 * /robot/joint_state_publish_rate [std_msgs/UInt16]
 * /robot/limb/right/command_joint_position [motor_control_msgs/JointMotionCommand]
 * /robot/limb/right/command_nullspace_setpoint_and_twist_stamped [motor_control_msgs/NullspaceTwistStamped]
 * /robot/limb/right/command_stiffness [std_msgs/UInt32]
 * /robot/limb/right/command_stiffness_limit [motor_control_msgs/Stiffness]
 * /robot/limb/right/command_twist_speed_limit [motor_control_msgs/SpeedLimit]
 * /robot/limb/right/command_twist_speed_limit_scale [motor_control_msgs/SpeedLimitScale]
 * /robot/limb/right/command_twist_stamped [geometry_msgs/TwistStamped]
 * /robot/limb/right/command_velocity_tozero [std_msgs/Bool]
 * /robot/limb/right/interaction_control_command [intera_core_msgs/InteractionControlCommand]
 * /robot/limb/right/joint_command [intera_core_msgs/JointCommand]
 * /robot/limb/right/joint_command_timeout [std_msgs/Float64]
 * /robot/limb/right/set_damping_correction_weights [motor_control_msgs/JointPosition]
 * /robot/limb/right/set_dominance [std_msgs/Bool]
 * /robot/limb/right/set_feed_forward_weights [motor_control_msgs/JointPosition]
 * /robot/limb/right/set_speed_ratio [std_msgs/Float64]
 * /robot/limb/right/suppress_collision_avoidance [std_msgs/Empty]
 * /robot/limb/right/suppress_contact_safety [std_msgs/Empty]
 * /robot/limb/right/suppress_cuff_interaction [std_msgs/Empty]
 * /robot/limb/right/suppress_gravity_compensation [std_msgs/Empty]
 * /robot/limb/right/suppress_hand_overwrench_safety [std_msgs/Empty]
 * /robot/limb/right/suppress_squish_safety [std_msgs/Empty]
 * /robot/limb/right/use_default_spring_model [std_msgs/Empty]
 * /robot/limb/right/weight_integral_terms [motor_control_msgs/WeightIntegralTerms]
 * /robot/set_homing_mode [intera_core_msgs/HomingCommand]
 * /robot/set_motor_voltage_low [std_msgs/Bool]
 * /robot/set_sim_mode [std_msgs/Bool]
 * /robot/set_super_enable [std_msgs/Bool]
 * /robot/set_super_reset [std_msgs/Empty]
 * /robot/set_super_stop [std_msgs/Empty]
 * /robot/urdf [intera_core_msgs/URDFConfiguration]

Services:
 * /io/robot/command
 * /io/robot/cuff/command
 * /io/robot/navigator/command
 * /io/robot/pneumatic/command
 * /io/robot/right_end_of_arm/command
 * /io/robot/robot/command
 * /io/robot/safety/command
 * /list_controller_types
 * /list_controllers
 * /load_controller
 * /realtime_loop/get_loggers
 * /realtime_loop/set_logger_level
 * /reload_controller_libraries
 * /robot/limb/right/set_gc_enable
 * /robot/limb/right/set_gc_tare
 * /switch_controllers
 * /unload_controller


contacting node http://021802CP00071.local:43166/ ...
ERROR: Communication with node[http://021802CP00071.local:43166/] failed!
(reverse-i-search)`etc': echo '192.168.0.103 021802CP00071.local 021802CP00071' >> /^Cc/hosts
root@depend:~/ros_ws# cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       new

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
(reverse-i-search)`echo ': rostopic ^Cho /robot/accelerometer_names
(reverse-i-search)`etc': git f^Cch && git pull
```

- ros master at `192.168.0.103:11311` is reachable.
- `/robot/state` is registered.
- its publisher advertises `http://021802CP00071.local:43166/`.
- the container cannot resolve or reach `021802CP00071.local`, so rostopic echo hangs.

add sawyer hostname to hosts:

```bash
echo '192.168.0.103 021802CP00071.local 021802CP00071' >> /etc/hosts
```

### moveit functional

source 1

```bash
apt update
apt install ros-noetic-moveit

cd ~/ros_ws/
./intera.sh
cd ~/ros_ws/src
wstool merge https://raw.githubusercontent.com/RethinkRobotics/sawyer_moveit/melodic_devel/sawyer_moveit.rosinstall
wstool update
cd ~/ros_ws/
catkin_make
```

running

in every terminal for real robot, run

```bash
source /root/ros_ws/src/intera_sdk/intera.sh && export ROS_IP=192.168.0.20 && unset ROS_HOSTNAME
```

```bash
rosrun intera_interface enable_robot.py -e
rosrun intera_interface joint_trajectory_action_server.py
roslaunch sawyer_moveit_config sawyer_moveit.launch
```

### docker network issues

moveit was hanging here

```
[intera - http://192.168.0.103:11311] root@depend:~/ros_ws# roslaunch sawyer_moveit_config sawyer_moveit.launch
... logging to /root/.ros/log/1a5bd2b6-80e2-11f1-8757-d89ef31d645e/roslaunch-depend-2177969.log
Checking log directory for disk usage. This may take a while.
Press Ctrl-C to interrupt
Done checking log file disk usage. Usage is <1GB.

xacro: in-order processing became default in ROS Melodic. You can drop the option.
Child elements of a <xacro:include> tag are ignored
when processing file: /root/ros_ws/src/sawyer_robot/sawyer_description/urdf/sawyer.urdf.xacro
xacro: in-order processing became default in ROS Melodic. You can drop the option.
```

report from chat:

#### How we proved Sawyer could not connect back to the container

##### 1. Confirmed the container could reach Sawyer

```bash
ping -c 3 192.168.0.103
```

Output showed successful replies:

```text
64 bytes from 192.168.0.103
0% packet loss
```

We also reached Sawyer’s ROS master:

```bash
echo "$ROS_MASTER_URI"
rostopic list
```

Output:

```text
http://192.168.0.103:11311
```

`rostopic list` returned Sawyer topics such as:

```text
/robot/state
/robot/joint_states
/robot/limb/right/follow_joint_trajectory/goal
/robot/limb/right/follow_joint_trajectory/status
```

This proved:

```text
Container -> Sawyer: working
```

##### 2. Tested the trajectory action connection

```bash
python3 - <<'PY'
import rospy
import actionlib
from control_msgs.msg import FollowJointTrajectoryAction

rospy.init_node("test_action_client")

client = actionlib.SimpleActionClient(
    "/robot/limb/right/follow_joint_trajectory",
    FollowJointTrajectoryAction
)

print("connected:", client.wait_for_server(rospy.Duration(10)))
PY
```

Output:

```text
connected: False
```

The action server was registered with the ROS master, but the complete action connection could not be established.

##### 3. Inspected the action topics

```bash
rostopic info /robot/limb/right/follow_joint_trajectory/goal
rostopic info /robot/limb/right/follow_joint_trajectory/cancel
rostopic info /robot/limb/right/follow_joint_trajectory/status
rostopic info /robot/limb/right/follow_joint_trajectory/feedback
rostopic info /robot/limb/right/follow_joint_trajectory/result
```

The key result was:

```text
status / feedback / result: connections present
goal / cancel: no TCPROS connection established
```

This indicated one-way ROS communication. The container could receive information from Sawyer, but Sawyer could not connect back to the address advertised by the container.

##### 4. Checked the advertised ROS address

```bash
env | grep '^ROS_'
```

Output included:

```text
ROS_MASTER_URI=http://192.168.0.103:11311
ROS_IP=10.0.2.100
```

Sawyer was therefore being told to connect to:

```text
10.0.2.100
```

That address belonged to the rootless Docker network and was not reachable from Sawyer.

##### 5. Checked the container network

```bash
ip -4 addr
ip route
ip route get 192.168.0.103
```

The container showed an interface similar to:

```text
tap0
inet 10.0.2.100
```

The route used the rootless Docker network rather than the physical Ethernet interface.

On the host:

```bash
ip -4 addr show enp37s0f1
```

Output:

```text
inet 192.168.0.20/24
```

The address Sawyer could actually reach was:

```text
192.168.0.20
```

##### 6. Confirmed Docker was rootless

```bash
docker context show
docker context ls
docker info
```

Output showed the rootless context and a Docker root directory similar to:

```text
rootless
/home/peilunli/.local/share/docker
```

Therefore, `--network host` was using the rootless daemon’s network namespace, not the physical host network.

#### Conclusion

```text
Container -> Sawyer ROS master: working
Sawyer -> container callbacks: failing
```

The container advertised `10.0.2.100`, but Sawyer could only reach the host at `192.168.0.20`. We fixed this by moving the ROS container to system Docker with real host networking and using:

```bash
export ROS_MASTER_URI=http://192.168.0.103:11311
export ROS_IP=192.168.0.20
unset ROS_HOSTNAME
```

### fix docker

source 2 confirms that rootless docker containers don't share the same network namespace even in `--net=host` so ports inside the container were not reachable from outside

updating docker was an option we considered but didn't want to risk breaking other containers

instead i just got access to docker use group and used the default not rootless context

to do this we need to make a new container on the root context

#### new container

backing up docker

```bash
docker context use rootless
docker stop ros
docker commit ros sawyer-ros-backup:noetic
docker save -o ~/sawyer-ros-backup.tar sawyer-ros-backup:noetic

ls -lh ~/sawyer-ros-backup.tar
```

```bash
docker context use default
unset DOCKER_HOST
docker load -i ~/sawyer-ros-backup.tar

xhost +local:docker

docker run -it \
  --name ros \
  --network host \
  --gpus all \
  -e DISPLAY="$DISPLAY" \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  sawyer-ros-backup:noetic
```

```bash
docker --context rootless rm ros
docker --context rootless rmi sawyer-ros-backup:noetic
rm ~/sawyer-ros-backup.tar
```

for astra camera we need to give usb permissions as well so this is hte final command

```bash
docker run -it \
  --name ros \
  --network host \
  --gpus all \
  --device-cgroup-rule='c 189:* rmw' \
  -e DISPLAY="$DISPLAY" \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v /dev/bus/usb:/dev/bus/usb \
  sawyer-ros-full-backup:noetic
```

`--device-cgroup-rule='c 189:* rmw'` broadly permits access to USB character devices (still need `--volume /dev/bus/usb:/dev/bus/usb` to expose the devices themselves)
