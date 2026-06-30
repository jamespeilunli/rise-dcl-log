# rise-dcl-log

# 260629

- [slides](https://docs.google.com/presentation/d/1C7Mwcdt3m7QfknjxOcZXIfugGhLVEKumrAQWOlkqRtM/edit)
- [workstation setup](https://web.archive.org/web/20230217220014/https://sdk.rethinkrobotics.com/intera/Workstation_Setup#Install_Intera_SDK_Dependencies)
- [gazebo tutorial](https://web.archive.org/web/20230217230807/https://sdk.rethinkrobotics.com/intera/Gazebo_Tutorial)

## install ROS on ubuntu 20 docker

```
docker run -it --rm \
  --name=ros \
  --net=host \
  --gpus all \
  -e DISPLAY=$DISPLAY \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  osrf/ros:noetic-desktop-full
```

```
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

confirm gpu and display

```
nvidia-smi
echo $DISPLAY
```

need to

```
export DISPLAY=:1
```

## sawyer setup

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

```
cv_ptr->image = cv::imread(img_path, cv::IMREAD_UNCHANGED);
```

```
. devel/setup.bash
roslaunch sawyer_sim_examples sawyer_pick_and_place_demo.launch
```
