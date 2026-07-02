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

# 260704

## sources

1. [rqt_plot](https://wiki.ros.org/rqt_plot)
2. [rviz](https://wiki.ros.org/rviz)
3. [intera_interface](https://rethinkrobotics.github.io/intera_sdk_docs/5.0.4/intera_interface/html/intera_interface.limb.Limb-class.html)

## learned

### rviz

- see source 2
- rviz seems to have many use cases, but today I used it to visualize the tf frames
- you can check the frames you want to see and see their relationships to each other
- make sure to keep the time in sync with the sim

### rqt_plot

see source 1

### rqt_image_view

### rosrun image_view

`rosrun image_view image_view image:=<path_to_image_topic>`

### coordinate conventions/vocab

- "rpy": roll pitch yaw
- x y z matches up with r g b

## log

### pt 1: somewhat successful pnp!

#### using block center as block coordinates

the origin of the block model is in the corner. we must offset by + [0.025, 0.025, 0.025] to make the coordinates in the center.

```python
if object_name == "block":
    p = np.array([
        pose.position.x,
        pose.position.y,
        pose.position.z
    ])

    q = [
        pose.orientation.x,
        pose.orientation.y,
        pose.orientation.z,
        pose.orientation.w
    ]

    # quaternion_matrix from tf.transformations
    # turn quaternion into a 3d rotation matrix.
    # by default the matrix returns a 4x4 *homogeneous transform* which has a translation component
    R = quaternion_matrix(q)[:3, :3]

    offset_local = np.array([0.025, 0.025, 0.025])
    # rotation matrix times offset is our delta to find the center
    center = p + R.dot(offset_local)

    t.transform.translation.x = center[0]
    t.transform.translation.y = center[1]
    t.transform.translation.z = center[2]
```

#### also did some logic changes to make picking up and placing down faster

#### exiting a ros process

i tried using try catch on stuff like `rospy.exceptions.ROSInterruptException` but looks like that doesn't actually work

TODO: look into this further. but i think you just have to do `if rospy.is_shutdown() return`

#### why does ik_request take in a pose in the sawyer frame?

`hdr = Header(stamp=rospy.Time.now(), frame_id='base')` in
`src/intera_sdk/intera_interface/src/intera_interface/limb.py`:
you may think base means the base frame, but it's actually the base of the model.

in `src/sawyer_simulator/sawyer_gazebo/launch/sawyer_world.launch`:

```
  args="-param robot_description -urdf -z 0.93 -model sawyer ..."
```

additionally, base is the same as world in this case.

so sawyer is 0.93 z above world/base.

this matches up with what we see from tf_echo:

```
root@depend:~/ros_ws# rosrun tf tf_echo base block
At time 1362.251
- Translation: [1.027, 0.325, 0.797]
- Rotation: in Quaternion [-0.000, 0.000, -0.409, 0.913]
            in RPY (radian) [-0.000, -0.000, -0.842]
            in RPY (degree) [-0.000, -0.000, -48.268]
root@depend:~/ros_ws# rosrun tf tf_echo base sawyer
At time 1366.104
- Translation: [-0.000, -0.000, 0.930]
- Rotation: in Quaternion [0.000, -0.000, -0.000, 1.000]
            in RPY (radian) [0.000, -0.000, -0.000]
            in RPY (degree) [0.000, -0.000, -0.000]
root@depend:~/ros_ws# rosrun tf tf_echo sawyer block
At time 1371.908
- Translation: [1.027, 0.325, -0.133]
- Rotation: in Quaternion [-0.000, 0.000, -0.409, 0.913]
            in RPY (radian) [-0.000, -0.000, -0.842]
            in RPY (degree) [-0.000, -0.000, -48.268]
root@depend:~/ros_ws# rosrun tf tf_echo world base
At time 0.000
- Translation: [0.000, 0.000, 0.000]
- Rotation: in Quaternion [0.000, 0.000, 0.000, 1.000]
            in RPY (radian) [0.000, -0.000, 0.000]
            in RPY (degree) [0.000, -0.000, 0.000]
```

#### troubles with max speed

TODO: figure out

`set_joint_position_speed` didn't seem to change speed at all even when setting it to zero

#### normalized quaternion interpolation

#### final system

- i edited the logic to make it do the pick and place loop a bit faster.
- it reuses \_servo_to_pose for the approach location which is just the goal position but +hover_distance z
- in a loop, it goes to approach location for picking, then picks, then goes back up to approach, then does the same except for placing instead of picking
- placing is done at +0.1 x the original picking location

#### future

- maybe could detect if block is inside by getting force on grabbers (see intera docs)

### pt 2: camera simulation

- the sawyer sim example already had camera simulation. i checked by doing `rostopic list | grep camera` and seeing `/io/internal_camera/head_camera/image_raw` and `/io/internal_camera/right_hand_camera/image_raw`
- can stream using `rosrun image_view image_view image:=<path_to_image_topic>` or `rqt_image_view`
- the head camera worked pretty well but the right_hand_camera was not in the right location, orientation. it also was monochrome

### fixing right_hand_camera pose

- checked arm components using `rosrun tf view_frames`
- in gazebo we highlighted the components of the arm (right_l5 and right_l6)
- noticed right_hand_camera was on right_l5 in `~/ros_ws/src/sawyer_robot/sawyer_description/urdf/sawyer_base.urdf.xacro`
- because we want it to be pointing sort of parallel to the grabbers, we want it to be on right_l6
- we configured its parent to be right_l6 and updated its position and orientation accordingly

### fixing right_hand_camera to have color

- had to change configs in `~/ros_ws/src/sawyer_simulator/sawyer_gazebo/launch/sawyer_sim_cameras.launch`
- had to change format from `L8` to `R8G8B8` in `~/ros_ws/src/sawyer_robot/sawyer_description/urdf/sawyer_base.gazebo.xacro`

# 260702

## sources

1. [hierarchical VLA paper](https://arxiv.org/abs/2606.10267)
2. [sparse reward](https://arxiv.org/html/2407.00324v2)
3. [GROD](https://deepmind.google/models/gemini-robotics/gemini-robotics-on-device/)
4. [CodeGraphVLP](https://arxiv.org/abs/2604.22238)
5. [mark based visual prompting](https://arxiv.org/abs/2403.03174)

## learned

AI concepts:

- sparse reward
- non-markovian problems and importance of memory
- policy
- frontier VLM/VLA robotics basic architecture

## log

### some clarification on project goals - difference between setups

> We will start with a VLM+MoCap/Camera+LLM setup first and move to a VLM+VLA setup if we have the time.

(Will be referencing source 1 a lot.)

Both use a VLM to operate in a high-level loop, outputting small instructions to the low-level loop. They differ in what the low-level loop looks like.

VLM+MoCap/Camera+LLM setup

- Must have code that processes between parts of the system
- Importantly, the MoCap/Camera information must be turned into text-based data (e.g. a pose of a block) to be fed as context into the LLM
- Easier in terms of getting it to work/training
- RL fine-tuning can still be done, but you can't really train the whole system as an end-to-end model

VLM+VLA setup

- End-to-end
- The only inputs to the VLA are the direct language instruction from the VLM, and images of the scene
- Harder because of training - because existing VLA are often tuned for a specific setup. Fine-tuning is required to get it to work.
- Training can be done end-to-end

#### RL

(See source 2)

Very important RL concept: _sparse reward_

> Sparse reward reinforcement learning occurs when an agent only receives feedback for completing a complex task

For complex tasks, it may be nearly impossible to train like this because there could be millions of states receiving 0 reward

_reward shaping_: adds intermediate heuristic metrics to guide agent toward final goal

### source 1 hierarchical VLA (Hi-VLA) literature review

#### 1. intro

- monolithic VLA bad at long or abstract reasoning tasks. struggle to generalize
- fine tuning often compromises reasoning
- hierarchical VLA better because VLM plans and orchestrates and VLA executes specific small task
- hierarchical VLA systems can be improved through:
  - better VLMs (better reasoning, spatial awareness)
  - steerable VLA controllers
  - termination rules
  - memory mechanisms for VLM
  - representations of observations for VLM

#### 3. hierarchical VLA overview

Unified Hi-VLA algorithm pseudocode:

```
Input: task instruction

while task isn't complete {
  // High level VLM loop
  observe and process images/observations from environment
  update memory with past observations, commands, and outcomes
  subtask instruction = VLM(memory, task instruction)

  while subtask hasn't reached termination {
    action = VLA(current time observation, subtask instruction)
    perform the action
  }
}
```

#### 4.2. VLM policy

- they tried Gemini 2.5 Lite, Flash, and Pro

conclusion:

- reasoning = good
- model size doesn't matter too much

#### 4.3. VLA policy

- tried Gemini Robotics On-Device (GROD) model (see source 3)

conclusion:

- model size bigger = more "steerable" = good
- fine tuning can lose steerability, ability to generalize, and struggle in long horizon

#### 4.4. termination conditions

ways to terminate:

- fixed frequency: force a duration for VLA (an _execution horizon_) to run each subtask
- success detection: success detector detects success
- VLM termination: VLM generates expected execution duration

conclusion:

- looks like they tried success detection and fixed frequency
- success detection = good
- fixed frequency = they recommend 4-8 seconds for execution horizon. smaller is better but computation cost is a consideration.

#### 4.5. observation representations

ways to represent observations to feed into the VLM:

- raw image
- naive summarization: image + basic summarization from VLM
- summarize with privileged info: sim information passed when generating text summary (e.g. contact between objects)
- summarize with bounding box: image with VLM generated bounding boxes + text description

conclusion:

- bounding box = good
- privileged = good. obviously you can't get privileged info irl.
  - they frame this as a limitation of VLM models as they SHOULD be able to have all that information in the image but currently lack complex spatial understanding capabilities
  - additional sensors (e.g. force sensor) can achieve similar effect in this regard

#### 4.6. memory

> However, summarizing experiences across episodes can positively impact performance. This suggests that for hierarchical systems, extracting affordances from cross- episode information (especially previous successful episodes) is more beneficial than relying on in-episode failure signals for on-the-fly VLM corrections.

This is also interesting to me:

> Future work could explore more powerful techniques, such as reinforcement learning or supervised finetuning of the VLM, to better leverage cross-episodic interactions with the VLA.

conclusion:

- in-episode memory = no effect
- cross-episode memory = good

#### 5. future

- investigate latency sensitive scenarios (dynamic environment)
- RL or supervised finetuning of VLM to better integrate cross-episodic knowledge -> give better prompts to VLA
- hierarchical architecture can "guide low-level continual policy improvement", TODO see https://arxiv.org/abs/2603.11653

#### setup

- MuJoCo ALOHA suite - tabletop manipulation benchmark
- experiments mainly in simulation
- tasks categorized by short-horizon, long-horizon, and reasoning
- to me it looks like they relied on existing solutions from Google for setup (ALOHA, GROD)

interesting tasks

- "Put the item that monkey can eat into the bowl"
- "Put all the cubes into the green cup"
- irrelevant items on table as distractors

### source 4 CodeGraphVLP

#### vocab

- _Markovian_: assumes current visual observation contains all the information needed to predict the next action
- _Non-Markovian_: provides memory of what happened in the past. useful for complex, long-horizon tasks
- _policy_: a function/model that outputs an action based on inputs like vision/language

#### summary

problem

- lack of "grounded task state" in the memory of VLA systems
- basically they struggle when the correct next action depends on previous events (non-Markovian)
- difficulty in cluttered scenes
- latency in traditional language based VLM setups

solution

- create a graph representation where objects are the nodes and edges are relationships between them
  - instance segmentation model finds objects
  - VLM determines which objects matter, naming them
  - objects in multiple views are associated
  - relationships between objects are determined via code
- LLM is given task instruction, graph schema/API, and initial graph. they write a python function `policy(graph)` that outputs a language subtask for the VLA, given the current graph state
- after that, the code executes the plan, and VLA follows each subtask

#### setup

tasks

- Pick-and-Place Twice: "Move the cube to the other plate, then return it to the original plate"
- Place-and-Stack: "Put the cube into the nearest cup, then stack the other cup on top."
- Swap Cups: "Swap two cups, start with blue cup."

some VLA that they compared against

- pi 0 (used this as the one CodeGraphVLP used)
- pi 0 fast
- pi 0.5
- gr00t n1

software impl

- "fine-tune all VLA models from their public checkpoints for 50K iterations using low-rank adaptation (learning rate 1 × 10^−5, batch size 128) on four NVIDIA A6000 GPUs."
- each VLA queried predicts a short sequence of actions (_action chunk_) of size 10. i.e. execute 10 actions before reobserving scene and going back to policy()
- GPT-5 creates `policy` function

hardware

- UR10e robotic arm with a Robotiq 2F-85 parallel-jaw gripper

camera

- rgb
- images 10 hz
- fixed camera over shoulder
- wrist-mounted camera near gripper

### source 5 mark-based visual prompting

#### summary

- uses _mark-based visual prompting_ to convert language instructions + RGB-D into actions
- VLM doesn't directly output 3d actions
- overlays "affordance marks" over image and asks VLM to select keypoints, waypoint regions, and motion attributes in a structured response

#### setup

some tasks ("open-world" tabletop scenes with multiple relevant and distractor objects)

- table wiping: move glasses into a case; sweep trash with a broom
- watch cleaning: put watch into ultrasonic cleaner; press power button
- gift preparation: put filler in gift box; put perfume in gift box
- laptop packing: unplug cable; close laptop lid

hardware

- 7-DoF Franka Emika robot arm
- 2F-85 Robotiq gripper
- 5 hz

cameras

- RGB-D
- two fixed ZED 2.0 cameras
- one ZED mini wrist
- top-down camera as primary camera enables spatial grounding

> This design highlights an emerging strategy in VLM-based manipulation: simplifying robotic spatial reasoning by converting continuous action generation into discrete visual question answering.

### summary of literature review

- source 1 gives us a strong overview of the basic Hi VLA architecture, an emerging technique that allows complex task planning to be done separately from VLA controllers turning basic language instructions into robot actions
- source 4 changes the means of planning using a semantic graph of object relationships and generating code to execute on a plan instead of relying on a VLM every loop
- source 5 uses "affordance marks" to address spatial awareness problems via labelling key points
- all of them emphasize the importance of maintaining spatial and temporal reasoning over a complex task. VLM/VLA struggle with this, (ie VLMs lack complex spatial awareness of objects from raw images alone). additionally, there is the memory aspect, with non-Markovian tasks requiring awareness of past events and success status. therefore, there are many techniques to essentially engineer context beyond the raw image and current observations for these models
- on the setup perspective:
  - experimental tasks emphasize spatial, temporal, or reasoning/interpretation complexity
  - scenes are usually a flat table with gripper(s), various household objects or shapes. importantly, distractor items not relevant to the task simulate an "open-world" environment
  - cameras are RGB, sometimes depth. usually there is the "main camera" that is fixed and see the entire environment from top-down. sometimes there is camera mounted at gripper
  - prompts are relatively simple from what i see. they are clear, concise, direct, and prioritize getting the context across

### stuff to look into further after literature review

- exact prompt engineering strategies (or lack thereof)
- what the VLA controller outputs - just a pose?
- what makes the VLA work itself. it needs GOOD 3D spatial reasoning
  - extra context?
  - for our Vision + LLM, may be easier, but any systems to make poses make sense, or code/libraries we can use, etc.
- couldn't find any sawyer examples. investigate difficulty or difference of running sawyer
