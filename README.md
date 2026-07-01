# rise-dcl-log

# 260629

## sources

1. [slides](https://docs.google.com/presentation/d/1C7Mwcdt3m7QfknjxOcZXIfugGhLVEKumrAQWOlkqRtM/edit)
2. [workstation setup](https://web.archive.org/web/20230217220014/https://sdk.rethinkrobotics.com/intera/Workstation_Setup#Install_Intera_SDK_Dependencies)
3. [gazebo tutorial](https://web.archive.org/web/20230217230807/https://sdk.rethinkrobotics.com/intera/Gazebo_Tutorial)

## learned

- introduced to basic idea of the project with VLA
- pretty sure this sort of organization VPN + user on build server with ssh/remote desktop is very common in irl setup
- docker stuff:

docker useful commands

```
# create a container from an image but do not run it
docker create ...

# start an EXISTING container (e.g. one you created)
docker start <container_name>

# run a fresh no container from an image
docker run ...

# get into bash of an existing container
docker exec -it <container_name> /bin/bash
```

flags:

- `-d`: detached
- `-it`: interactive and tty
- `--rm`: delete filesystem after done

## log

We visited the lab and the office.

We got introduced to the robot arm and localization

We set up accounts on the build server and set up remote connection and remote desktop using BU's vpn and using
NoMachine for remote desktop.

We noticed that NoMachine only worked for one session at a time, so still figuring that out.

### install ROS on ubuntu 20 docker

install and run container with noetic setup already. use host network, host gpu, and host display.

```
docker run -it \
  --name=ros \
  --net=host \
  --gpus all \
  -e DISPLAY=$DISPLAY \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  osrf/ros:noetic-desktop-full
```

> oops actually I used `--rm` irl but I don't think we want to run with that because that will remove the filesystem when it exits

> update: yup that's exactly what happened, we just ran the commands again and we good now

make sure ros setup script always runs:

```
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

confirm gpu and display

```
nvidia-smi
echo $DISPLAY
```

DISPLAY is empty, need to set it

```
export DISPLAY=:1
```

colon syntax: `hostname:display.screen`, so :0 is usually main monitor, :1 may be remote GUI

### sawyer setup

dependencies

```
apt update
apt install -y git vim
apt install -y git-core python3-wstool python3-vcstools python3-rosdep ros-noetic-control-msgs ros-noetic-joystick-drivers ros-noetic-xacro ros-noetic-tf2-ros ros-noetic-rviz ros-noetic-cv-bridge ros-noetic-actionlib ros-noetic-actionlib-msgs ros-noetic-dynamic-reconfigure ros-noetic-trajectory-msgs ros-noetic-rospy-message-converter
apt install -y gazebo11 ros-noetic-gazebo-ros  ros-noetic-gazebo-ros-control ros-noetic-gazebo-ros-pkgs ros-noetic-ros-control ros-noetic-control-toolbox ros-noetic-realtime-tools ros-noetic-ros-controllers ros-noetic-xacro python3-wstool ros-noetic-tf-conversions ros-noetic-kdl-parser

apt install -y python3 python3-pip python3-venv
pip install argparse

rosdep init
rosdep update
```

get code and compile

```
mkdir -p ~/ros_ws/src
cd ~/ros_ws/src
git clone https://github.com/RethinkRobotics/sawyer_simulator.git -b noetic_devel
git clone https://github.com/RethinkRobotics-opensource/sns_ik.git -b melodic-devel
cd ~/ros_ws/src
wstool init .
wstool merge sawyer_simulator/sawyer_simulator.rosinstall
wstool update

source /opt/ros/noetic/setup.bash
cd ~/ros_ws
catkin_make
```

Change /ros_ws/src/sawyer_simulator/sawyer_gazebo/src/head_interface.cpp line 71, due to deprecated constant:

old:

```
cv_ptr->image = cv::imread(img_path, CV_LOAD_IMAGE_UNCHANGED);
```

new:

```
cv_ptr->image = cv::imread(img_path, cv::IMREAD_UNCHANGED);
```

run

```
catkin_make
. devel/setup.bash
roslaunch sawyer_sim_examples sawyer_pick_and_place_demo.launch
```

haven't tested with desktop yet

# 260630

## sources

https://wiki.ros.org/tf/Tutorials/Introduction%20to%20tf

1. [ros gemoetry_msgs api docs](https://docs.ros.org/en/noetic/api/geometry_msgs/html/index-msg.html)
2. [tf intro and useful commands](https://wiki.ros.org/tf/Tutorials/Introduction%20to%20tf)
3. [tf broadcaster](<https://wiki.ros.org/tf/Tutorials/Writing%20a%20tf%20broadcaster%20(Python)>)
4. [MessageStates docs](https://docs.ros.org/en/jade/api/gazebo_msgs/html/msg/ModelStates.html)

## learned

- looking at documentation is #1
  - e.g. finding types through api documentation, reading tutorials for common tools like tf
- basic tf
- basic viewing of logs (TODO: honestly still figuring it out)

### ros commands

rostopic:

- `rostopic list`
- `rostopic info <topic_name>` tells you type, publishing nodes, and subscribing nodes
- `rostopic echo <topic_name>`

other:

- `roscore` must be run if you are doing `rosrun`
- always do `. devel/setup.bash` before running anything
- `roslaunch <package_name> <launch_file_name.launch>` launches using launchfile

### ros packages

package usually have

```
package.xml
CMakeLists.txt
scripts/
launch/
```

can be located anywhere, it's defined by `package.xml`

## log

### getting the GUI to show up

host:

allows all local connections to open windows on my display.

local means connecting via `/tmp/.X11-unix/X1`, the unix socket file

```
xhost +local:
```

```
echo $DISPLAY # got :5
xhost # should have something permissive like LOCAL:
ls -l /tmp/.X11-unix/ # should contain the $DISPLAY, e.g. /tmp/.X11-unix/X1
```

on the container

```
echo $DISPLAY # should match up with host
ls -l /tmp/.X11-unix/ # should match up with host

gzclient --verbose
```

### remote desktop using Microsoft Remote Desktop

### misc fixes/changes

- got VSCode SSH + Docker working. unfortunately python LSP not set up yet

- `s/Exception, e/Exception as e/`

- moved the block and table objects to be defined in the launch file instead of in code
  syntaax is like this

```
  <node name="spawn_block"
        pkg="gazebo_ros"
        type="spawn_model"
        output="screen"
        args="-file $(find sawyer_sim_examples)/models/block/model.urdf
              -urdf
              -model block
              -x 0.4225
              -y 0.1265
              -z 0.7725
              -Y 0.0" />
```

- abdullah removed the intermediate steps in the solver because there were some issues with the unit length (TODO: look into this)

> we had an issue where the interpolation for the robotic arm to come down onto the block (\_servo_to_pose) was linear, which didn't work for quaternions due to unit vector math that meant that linear interpolation would make the length of the vector !=1
> solution: we reduced the step size to 1 so that we did not have to worry about intermediate steps. since we are only working with cartesian movements, this wasn't a huge worry

### getting the robot to pick and place accurately

we want the robot to pick up the cube regardless of where it is (so, avoiding hardcoding the location). essentially, moving from _open loop_ to _closed loop_. to do this we want to solve for the coordinates of the block in the robot frame so that we can use that as the goal pose of the arm.

here is the idea:

- we broadcast sawyer robot and the block transforms
- `ik_pick_and_place_demo.py` listens and uses the block coordinates in the robot frame as the goal pose of the arm

#### tf stuff

see source 2

tf = transform. it represents relationships between coordinate frames

##### view_frames

there is a tree of all frames broadcast via tf over ROS

```
rosrun tf view_frames
```

then open `frame.pdf`

##### tf_echo

outputs the transform between any two frames broadcast over ROS

```
rosrun tf tf_echo <reference_frame> <target_frame>
```

##### debugging commands

```
rostopic echo /gazebo/model_states/name
rostopic info /tf
rostopic echo /tf | grep frame_id
rosrun tf tf_echo world object
rosrun tf tf_echo base_link object
rosrun tf view_frames
```

#### broadcasting transforms

see source 3

we need to do this because sawyer objects do not have their frames broadcast transforms by default

```python
#!/usr/bin/env python

import rospy
import tf2_ros
from gazebo_msgs.msg import ModelStates
from geometry_msgs.msg import TransformStamped

OBJECT_NAMES = ["block", "sawyer"]

class GazeboObjectTFBroadcaster:
    def __init__(self):
        self.br = tf2_ros.TransformBroadcaster() # initialize broadcaster
        self.last_stamp = rospy.Time(0)
        rospy.Subscriber("/gazebo/model_states", ModelStates, self.callback)

    def callback(self, msg):
        stamp = rospy.Time.now()

        # check last timestamp to avoid broadcasting identical frames too fast
        if stamp == self.last_stamp:
            return
        self.last_stamp = stamp

        for object_name in OBJECT_NAMES:
            if object_name not in msg.name:
                rospy.logwarn_throttle(2.0, "Object '%s' not found in /gazebo/model_states", object_name)
                continue

            """
            formatted like this:

            string[] name                 # model names
            geometry_msgs/Pose[] pose     # desired pose in world frame
            geometry_msgs/Twist[] twist   # desired twist in world frame

            so we need to search the index then use that to find pose
            """
            idx = msg.name.index(object_name)
            pose = msg.pose[idx]

            t = TransformStamped()
            t.header.stamp = stamp
            # header frame_id is the frame where the expressed coordinates are in.
            # Header is pretty common, it's metadata
            # for Header without coord data you leave it blank
            t.header.frame_id = "world"
            t.child_frame_id = object_name # this is the name of the tf

            # transform: Vector3 translation, Quaternion rotation
            # pose: Point position, Quaternion orientation
            t.transform.translation.x = pose.position.x
            t.transform.translation.y = pose.position.y
            t.transform.translation.z = pose.position.z
            t.transform.rotation = pose.orientation

            self.br.sendTransform(t)

if __name__ == "__main__":
    rospy.init_node("gazebo_object_tf_broadcaster")
    GazeboObjectTFBroadcaster()
    rospy.spin() # spin() is a "blocking call" that halts main thread indefinitely. used so the node doesn't end
```

#### listening to transforms to use them

listener

```
listener = tf.TransformListener()
```

lookup transform

```
(translation, rotation) = listener.lookupTransform('sawyer', 'block', rospy.Time(0))
```

btw this tf stuff is all tf 1, tf 2 is the updated version that i don't think we're using

### future thinking

- collaboration was kind of annoying so we probably going to use git repo in future
- the robot only gets the block pose once before going for it, so it could move around while we're getting to it
- had to offset the position, maybe due to semantics of where the origin in the block model is
