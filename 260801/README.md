# 260801

## log

### reset sawyer joint angles

reset sawyer joint angles using the `/gazebo/set_model_configuration` `ServiceProxy`

did this because once sawyer got stuck in an unrecoverable pose in the middle of an experiment, it messed up all future trials

i also made sure to set the goals to the joint angles and pause and clear physics to make this consistent

#### crazy sawyer wrapping bug

problem:

- Sawyer’s Gazebo reset occasionally sent the arm out of control between trials, caused by angular wrapping
- e.g. right_j3 was teleported from -1.166 to +2.175 rad, a change greater than pi.
- Gazebo’s ROS control interface updates revolute joints using shortest_angular_distance [](https://docs.ros.org/api/gazebo_ros_control/html/default__robot__hw__sim_8cpp_source.html), so it interpreted the target as the equivalent -4.108 rad branch
- the position controller then fought a roughly 2π error, saturated its efforts, and destabilized the arm

fix:

- the Python-only fix divides each reset into intermediate joint configurations whose changes are strictly less than pi
- physics is briefly unpaused between stages so gazebo_ros_control observes each intermediate position
- during reset, the position controller is stopped, model velocity is cleared, then the controller is restarted and commanded to hold the final pose
