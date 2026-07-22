# 260721

## tasks

## sources

1. [foundationpose](https://github.com/NVlabs/FoundationPose)
2. [foundationpose ros](https://github.com/IRVLUTD/FoundationPoseROS)
3. [ros env vars](https://wiki.ros.org/ROS/EnvironmentVariables)
4. [orbbec sdk](https://github.com/orbbec/OrbbecSDK_ROS1)
5. [astra specs](https://www.orbbec.com/products/structured-light-camera/astra-series/)
6. [gazebo depth tutorial](https://classic.gazebosim.org/tutorials?cat=connect_ros&tut=ros_depth_camera)

## learned

## log

### foundation pose on separate docker container

- it requires python 3.11, we only have python 3.8 on the ros noetic container
- therefore, we use instructions from source 2 to run it on separate docker container
- this works because ros can actually communicate over the host network, as if each container was its own device
- we just need to configure `ROS_MASTER_URI` and `ROS_IP` to be on localhost (see source 3)
- then, roscore on our main ros container will make the master on the host network and allow nodes from other "machines" (container) to communicate over the network

confirm main ros container has host network access:

```bash
docker inspect -f '{{.HostConfig.NetworkMode}}' ros
```

set env vars in `~/.bashrc` and restart

```bash
export ROS_MASTER_URI=http://127.0.0.1:11311
export ROS_IP=127.0.0.1
```

```bash
cd ~/workspace

git clone https://github.com/IRVLUTD/FoundationPoseROS.git
cd FoundationPoseROS

cat > docker/.env <<EOF
PROJ_ROOT=$(pwd)
DOCKER_USER_ARG=$(id -un)
DOCKER_USER_UID_ARG=$(id -u)
DOCKER_USER_GID_ARG=$(id -g)
TORCH_CUDA_ARCH_LIST_ARG="7.5 8.0 8.6 8.9+PTX"
ROS1_APT_PACKAGE_ARG=ros-noetic-desktop
DISPLAY=${DISPLAY:-:0}
TERM=${TERM:-xterm-256color}
EOF

bash docker/container_handler.sh build ros1
```

got an error like

```
 => ERROR [posekit-ros1 ros1-base  8/18] RUN --mount=type=cache,target=/var/cache/apt     apt-key adv --keyserver keyserver.ubuntu.com --recv-key F6E65AC044F831AC  3.5s
------
 > [posekit-ros1 ros1-base  8/18] RUN --mount=type=cache,target=/var/cache/apt     apt-key adv --keyserver keyserver.ubuntu.com --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE || apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE &&     add-apt-repository "deb https://librealsense.intel.com/Debian/apt-repo $(lsb_release -cs) main" -u &&     apt-get update && apt-get install -qq -y --no-install-recommends     librealsense2-udev-rules librealsense2 librealsense2-dkms librealsense2-utils librealsense2-dev     ros-noetic-realsense2-camera ros-noetic-realsense2-description:
0.589 Warning: apt-key output should not be parsed (stdout is not a terminal)
0.618 Executing: /tmp/apt-key-gpghome.LeYN6G6Tfe/gpg.1.sh --keyserver keyserver.ubuntu.com --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE
1.280 gpg: key C8B3A55A6F3EFCDE: public key ""CN = Intel(R) Intel(R) Realsense", O=Intel Corporation" imported
1.285 gpg: Total number processed: 1
1.285 gpg:               imported: 1
1.844 Hit:1 http://security.ubuntu.com/ubuntu focal-security InRelease
1.868 Hit:2 https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64  InRelease
1.968 Hit:3 https://packages.lunarg.com/vulkan/1.3.283 focal InRelease
1.970 Hit:4 http://packages.ros.org/ros/ubuntu focal InRelease
2.002 Hit:5 http://archive.ubuntu.com/ubuntu focal InRelease
2.089 Hit:6 http://archive.ubuntu.com/ubuntu focal-updates InRelease
2.172 Get:7 https://librealsense.intel.com/Debian/apt-repo focal InRelease [3589 B]
2.175 Hit:8 http://archive.ubuntu.com/ubuntu focal-backports InRelease
2.463 Err:7 https://librealsense.intel.com/Debian/apt-repo focal InRelease
2.463   The following signatures couldn't be verified because the public key is not available: NO_PUBKEY FB0B24895113F120
2.564 Reading package lists...
3.493 W: GPG error: https://librealsense.intel.com/Debian/apt-repo focal InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY FB0B24895113F120
[+] Building 0/1ository 'https://librealsense.intel.com/Debian/apt-repo focal InRelease' is not signed.
 ⠼ Service posekit-ros1  Building                                                                                                                                 161.4s
failed to solve: process "/bin/bash -c apt-key adv --keyserver keyserver.ubuntu.com --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE || apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE &&     add-apt-repository \"deb https://librealsense.intel.com/Debian/apt-repo $(lsb_release -cs) main\" -u &&     apt-get update && apt-get install -qq -y --no-install-recommends     librealsense2-udev-rules librealsense2 librealsense2-dkms librealsense2-utils librealsense2-dev     ros-noetic-realsense2-camera ros-noetic-realsense2-description" did not complete successfully: exit code: 100
```

fix: get dedicated realsense keyring

```
# ---------------------- Install librealsense ---------------------
RUN --mount=type=cache,target=/var/cache/apt \
    mkdir -p /etc/apt/keyrings && \
    curl -sSf https://librealsense.realsenseai.com/Debian/librealsenseai.asc | \
        gpg --dearmor --batch --yes \
        --output /etc/apt/keyrings/librealsenseai.gpg && \
    echo "deb [signed-by=/etc/apt/keyrings/librealsenseai.gpg] https://librealsense.realsenseai.com/Debian/apt-repo $(lsb_release -cs) main" \
        > /etc/apt/sources.list.d/librealsense.list && \
    apt-get update && \
    apt-get install -qq -y --no-install-recommends \
        librealsense2-udev-rules \
        librealsense2 \
        librealsense2-dkms \
        librealsense2-utils \
        librealsense2-dev \
        ros-noetic-realsense2-camera \
        ros-noetic-realsense2-description
```

error:

```
 => ERROR [posekit-ros1 ros1-base 12/18] RUN wget -q https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh &&     bash miniconda.  19.7s
------
 > [posekit-ros1 ros1-base 12/18] RUN wget -q https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh &&     bash miniconda.sh -b -p /opt/conda && rm miniconda.sh &&     ln -s /opt/conda/etc/profile.d/conda.sh /etc/profile.d/conda.sh &&     find /opt/conda/ -follow -type f -name '*.a' -delete &&     find /opt/conda/ -follow -type f -name '*.js.map' -delete &&     /opt/conda/bin/conda install mamba -c conda-forge &&     /opt/conda/bin/conda clean -afqy &&     chown -R peilunli:1005 /opt/conda &&     chmod -R g=u /opt/conda:
2.458 PREFIX=/opt/conda
2.763 Unpacking bootstrapper...
2.852 Unpacking payload...
9.190
9.190 Installing base environment...
9.190
10.46 Preparing transaction: ...working... done
10.72 Executing transaction: ...working... done
16.49 installation finished.
18.66
18.66 CondaToSNonInteractiveError: Terms of Service have not been accepted for the following channels. Please accept or remove them before proceeding:
18.66     - https://repo.anaconda.com/pkgs/main
18.66     - https://repo.anaconda.com/pkgs/r
18.66
18.66 To accept these channels' Terms of Service, run the following commands:
18.66     conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
18.66     conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
18.66
18.66 For information on safely removing channels from your conda configuration,
18.66 please see the official documentation:
18.66
18.66     https://www.anaconda.com/docs/tools/working-with-conda/channels
[+] Building 0/1
 ⠹ Service posekit-ros1  Building                                                                                                                                 332.2s
failed to solve: process "/bin/bash -c wget -q https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh &&     bash miniconda.sh -b -p /opt/conda && rm miniconda.sh &&     ln -s /opt/conda/etc/profile.d/conda.sh /etc/profile.d/conda.sh &&     find /opt/conda/ -follow -type f -name '*.a' -delete &&     find /opt/conda/ -follow -type f -name '*.js.map' -delete &&     /opt/conda/bin/conda install mamba -c conda-forge &&     /opt/conda/bin/conda clean -afqy &&     chown -R ${USER_NAME_ARG}:${USER_GID_ARG} /opt/conda &&     chmod -R g=u /opt/conda" did not complete successfully: exit code: 1
```

fix: its an issue with their interactive tos prompt so we accept

```
# --------------------- Install Miniconda3 ---------------------
RUN wget -q https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh && \
    bash miniconda.sh -b -p /opt/conda && rm miniconda.sh && \
    ln -s /opt/conda/etc/profile.d/conda.sh /etc/profile.d/conda.sh && \
    find /opt/conda/ -follow -type f -name '*.a' -delete && \
    find /opt/conda/ -follow -type f -name '*.js.map' -delete && \
    /opt/conda/bin/conda tos accept \
        --override-channels \
        --channel https://repo.anaconda.com/pkgs/main && \
    /opt/conda/bin/conda tos accept \
        --override-channels \
        --channel https://repo.anaconda.com/pkgs/r && \
    /opt/conda/bin/conda install -y mamba -c conda-forge && \
    /opt/conda/bin/conda clean -afqy && \
    chown -R ${USER_NAME_ARG}:${USER_GID_ARG} /opt/conda && \
    chmod -R g=u /opt/conda
```

one more fix: fix the permissions, in `docker/docker-compose.yaml`:

```
    user: "0:0"

    environment:
      <<: *default-hocap-environment
      HOME: /home/${DOCKER_USER_ARG}
```

continue:

```bash
bash docker/container_handler.sh run ros1
bash docker/container_handler.sh enter ros1
```

inside container:

```
conda create --prefix "$PWD/env" python=3.10 -y
conda activate "$PWD/env"

python -m pip install \
  torch==2.5.1 \
  torchvision==0.20.1 \
  --index-url https://download.pytorch.org/whl/cu118 \
  --no-cache-dir

python -m pip install -r requirements.txt --no-cache-dir
python -m pip install -e source/RosPoseToolkit --no-cache-dir

bash scripts/install_foundationpose.sh



```

```

source /opt/conda/etc/profile.d/conda.sh
```

in `bash scripts/install_foundationpose.sh`

```
[INFO] 2026-07-21 12:20:19 - NVDiffRast installed successfully.
[INFO] 2026-07-21 12:20:19 - Installing PyTorch3D...
Collecting git+https://github.com/facebookresearch/pytorch3d.git@V0.7.8
  Cloning https://github.com/facebookresearch/pytorch3d.git (to revision V0.7.8) to /tmp/pip-req-build-gvxim8qz
  Running command git clone --filter=blob:none --quiet https://github.com/facebookresearch/pytorch3d.git /tmp/pip-req-build-gvxim8qz
  Running command git checkout -q 75ebeeaea0908c5527e7b1e305fbc7681382db47
  Resolved https://github.com/facebookresearch/pytorch3d.git to commit 75ebeeaea0908c5527e7b1e305fbc7681382db47
  Installing build dependencies ... done
  Getting requirements to build wheel ... error
  error: subprocess-exited-with-error

  × Getting requirements to build wheel did not run successfully.
  │ exit code: 1
  ╰─> [17 lines of output]
      Traceback (most recent call last):
        File "/home/peilunli/code/env/lib/python3.10/site-packages/pip/_vendor/pyproject_hooks/_in_process/_in_process.py", line 389, in <module>
          main()
        File "/home/peilunli/code/env/lib/python3.10/site-packages/pip/_vendor/pyproject_hooks/_in_process/_in_process.py", line 373, in main
          json_out["return_val"] = hook(**hook_input["kwargs"])
        File "/home/peilunli/code/env/lib/python3.10/site-packages/pip/_vendor/pyproject_hooks/_in_process/_in_process.py", line 143, in get_requires_for_build_wheel
          return hook(config_settings)
        File "/tmp/pip-build-env-rufxmsrv/overlay/lib/python3.10/site-packages/setuptools/build_meta.py", line 333, in get_requires_for_build_wheel
          return self._get_build_requires(config_settings, requirements=[])
        File "/tmp/pip-build-env-rufxmsrv/overlay/lib/python3.10/site-packages/setuptools/build_meta.py", line 301, in _get_build_requires
          self.run_setup()
        File "/tmp/pip-build-env-rufxmsrv/overlay/lib/python3.10/site-packages/setuptools/build_meta.py", line 520, in run_setup
          super().run_setup(setup_script=setup_script)
        File "/tmp/pip-build-env-rufxmsrv/overlay/lib/python3.10/site-packages/setuptools/build_meta.py", line 317, in run_setup
          exec(code, locals())
        File "<string>", line 15, in <module>
      ModuleNotFoundError: No module named 'torch'
      [end of output]

  note: This error originates from a subprocess, and is likely not a problem with pip.
ERROR: Failed to build 'git+https://github.com/facebookresearch/pytorch3d.git@V0.7.8' when getting requirements to build wheel
[ERROR] 2026-07-21 12:20:24 - Failed to install PyTorch3D.
```

```
# Install PyTorch3D
log_message "Installing PyTorch3D..."
if MAX_JOBS="${MAX_JOBS:-4}" FORCE_CUDA=1 \
    "${PYTHON_PATH}" -m pip install \
    --no-build-isolation \
    --no-cache-dir \
    "git+https://github.com/facebookresearch/pytorch3d.git@V0.7.8"; then
    log_message "PyTorch3D installed successfully."
else
    handle_error "Failed to install PyTorch3D."
fi
```

`--no-build-isolation` allows to use existing torch

```
rm -rf /home/peilunli/code/third_party/FoundationPose
bash scripts/install_foundationpose.sh
```

```
bash scripts/download_models.sh --foundationpose

wget -O checkpoints/sam2.1_t.pt \
  https://github.com/ultralytics/assets/releases/download/v8.3.0/sam2.1_t.pt
```

verify deps

```
python - <<'PY'
import torch
import pytorch3d
import nvdiffrast.torch

print("Torch:", torch.__version__)
print("CUDA version:", torch.version.cuda)
print("CUDA available:", torch.cuda.is_available())
print("PyTorch3D:", pytorch3d.__version__)
print("NVDiffRast: OK")
PY

source /opt/ros/noetic/setup.bash
source /opt/catkin_ws/devel/setup.bash

python - <<'PY'
import rospy
import cv2
import torch

print("ROS Python: OK")
print("OpenCV:", cv2.__version__)
print("GPU:", torch.cuda.get_device_name(0) if torch.cuda.is_available() else "unavailable")
PY
```

### rgbd

turns out our simulation doesn't have depth yet.

the actual camera we are using is the Astra Pro 3D Depth Camera. i will try to get this working in sim. it has ros support. working in sim should = working in real life

#### source 4

```bash
# Assuming you have sourced the ROS environment, same below
sudo apt install libgflags-dev ros-$ROS_DISTRO-image-geometry ros-$ROS_DISTRO-camera-info-manager \
ros-${ROS_DISTRO}-image-transport-plugins ros-${ROS_DISTRO}-compressed-image-transport \
ros-$ROS_DISTRO-image-transport ros-$ROS_DISTRO-image-publisher libgoogle-glog-dev libusb-1.0-0-dev libeigen3-dev \
ros-$ROS_DISTRO-diagnostic-updater ros-$ROS_DISTRO-diagnostic-msgs \
libdw-dev
```

```bash
cd ~/ros_ws/src
git clone https://github.com/orbbec/OrbbecSDK_ROS1.git
cd OrbbecSDK_ROS1
```

we want main

actually, we skip this because it is about real hardware and don't want to mess with that yet

```
root@depend:~/ros_ws/src/OrbbecSDK_ROS1# bash ./scripts/install_udev_rules.sh
cp: cannot create regular file '/etc/udev/rules.d/99-obsensor-libusb.rules': No such file or directory
usb rules file install at /etc/udev/rules.d/99-obsensor-libusb.rules
reload udev rules
./scripts/install_udev_rules.sh: line 19: udevadm: command not found
udev rules reload done
exit
```

#### simulate in gazebo

we use source 5 and source 6

edit `src/sawyer_robot/sawyer_description/urdf/sawyer_base.gazebo.xacro` to have depth configuration and other correct values like fov

test that it works in rviz:

1. set Fixed Frame to world or base.
2. add Image display for RGB: `/io/internal_camera/head_camera/image_raw`
3. add Image display for depth: `/io/internal_camera/head_camera/depth/image_raw`
4. add PointCloud2 display: `/io/internal_camera/head_camera/depth/points`

### configuration for one object

per source 2, we need to be able to run

```bash
python tools/run_foundation_pose_node.py \
  --mesh_file ./assets/Objects/DexCube/model.obj \
  --color_topic /000684312712/rgb/image_raw \
  --depth_topic /000684312712/depth_to_rgb/image_raw \
  --init_color ./recordings/ros/ros_image_color.jpg \
  --init_depth ./recordings/ros/ros_image_depth.png \
  --init_mask ./recordings/ros/ros_image_mask.png
```

we have color and depth topics now.

#### model

we convert the red block urdf to obj

#### init_color and init_depth

chatgpt wrote this script to capture the info we want

```python
#!/usr/bin/env python3

from pathlib import Path

import cv2
import message_filters
import rospy
from cv_bridge import CvBridge
from sensor_msgs.msg import Image


COLOR_TOPIC = "/io/internal_camera/head_camera/image_raw"
DEPTH_TOPIC = "/io/internal_camera/head_camera/depth/image_raw"
OUTPUT_DIR = Path("/home/peilunli/code/recordings/ros")


class CaptureInitialFrame:
    def __init__(self):
        self.bridge = CvBridge()
        self.saved = False

        OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

        color_sub = message_filters.Subscriber(COLOR_TOPIC, Image)
        depth_sub = message_filters.Subscriber(DEPTH_TOPIC, Image)

        sync = message_filters.ApproximateTimeSynchronizer(
            [color_sub, depth_sub],
            queue_size=20,
            slop=0.05,
        )
        sync.registerCallback(self.callback)

        rospy.loginfo("Waiting for synchronized RGB and depth frames...")

    def callback(self, color_msg, depth_msg):
        if self.saved:
            return

        color = self.bridge.imgmsg_to_cv2(color_msg, desired_encoding="bgr8")
        depth = self.bridge.imgmsg_to_cv2(
            depth_msg,
            desired_encoding="passthrough",
        )

        color_path = OUTPUT_DIR / "ros_image_color.jpg"
        depth_path = OUTPUT_DIR / "ros_image_depth.png"

        if not cv2.imwrite(str(color_path), color):
            raise RuntimeError(f"Failed to save {color_path}")

        if not cv2.imwrite(str(depth_path), depth):
            raise RuntimeError(f"Failed to save {depth_path}")

        rospy.loginfo(
            "Saved color %s %s and depth %s %s",
            color.shape,
            color.dtype,
            depth.shape,
            depth.dtype,
        )

        self.saved = True
        rospy.signal_shutdown("Initial RGB-D frame saved")


if __name__ == "__main__":
    rospy.init_node("capture_foundationpose_initial_frame")
    CaptureInitialFrame()
    rospy.spin()
```

attempting:

```bash
python tools/run_foundation_pose_node.py \
  --mesh_file /home/peilunli/models/model.obj \
  --color_topic /io/internal_camera/head_camera/image_raw \
  --depth_topic /io/internal_camera/head_camera/depth/image_raw \
  --init_color /home/peilunli/code/recordings/ros/ros_image_color.jpg \
  --init_depth /home/peilunli/code/recordings/ros/ros_image_depth.png \
  --init_mask /home/peilunli/code/recordings/ros/ros_image_mask.png
```

### resolving issues
