# 260722

## tasks

## sources

## log

### misc

optimize for 40 series

```bash
sed -i \
  's/^TORCH_CUDA_ARCH_LIST_ARG=.*/TORCH_CUDA_ARCH_LIST_ARG=8.9/' \
  docker/.env
```

### fix image

```bash
python tools/run_foundation_pose_node.py \
  --mesh_file /home/peilunli/code/models/red_block/model.obj \
  --color_topic /io/internal_camera/head_camera/image_raw \
  --depth_topic /io/internal_camera/head_camera/depth/image_raw \
  --init_color /home/peilunli/code/recordings/ros/ros_image_color.jpg \
  --init_depth /home/peilunli/code/recordings/ros/ros_image_depth.png \
  --init_mask /home/peilunli/code/recordings/ros/ros_image_mask.png
```

### FoundationPoseROS fork

had to fork it to keep track of all my changes and fix some issues in the code

### memory

vllm takes up 90% of cuda 0 memory, which doesn't leave enough for foundationpose which causes out of memory error

```
(/home/peilunli/code/env) root@depend:~/code# nvidia-smi
Wed Jul 22 10:08:58 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.159.03             Driver Version: 580.159.03     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4090        Off |   00000000:41:00.0 Off |                  Off |
|  0%   39C    P8             40W /  480W |   22772MiB /  24564MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA GeForce RTX 4090        Off |   00000000:61:00.0 Off |                  Off |
| 30%   35C    P2            114W /  480W |    1237MiB /  24564MiB |     50%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

we need to run vllm with less memory.

how much memory is it using currently?

in older vlm version [https://docs.vllm.ai/en/v0.10.0/configuration/engine_args.html#gpu-memory-utilization]() `--gpu-memory-utilization` is 0.9, i.e. 90% of gpu

this matches up with `nvidia-smi`:

```
24564 MiB × 0.90 = 22108 MiB
22108 MiB ÷ 1024 = 21.59 GiB
```

the model weights show `5.33 GB + 3.99 GB = 9.32 GB = 8.68 GiB` so we need at least that much to load model in memory

actual gpu use is higher because:

- CUDA context and kernels
- Temporary activations
- Multimodal/vision processing
- Compilation or CUDA graphs
- KV cache

since we are only using a single message exchange with a few paragraphs (max like 1000 tokens) let's assume a liberal upper end of +15% memory usage.

`1.15 * 8.68 = 9.982 GiB`

we can do 50% allocation which gives us a clean `11.99 GiB` budget, which should be more than enough

we also choose max-model-len of 8192 which should be more than enough for our purposes (8192 tokens = ~6000 words)

we must make a new vllm container with the settings.

(it was called 9b before i never renamed it)

```bash
docker stop vllm-qwen35-9b
docker rm vllm-qwen35-9b-old

docker run -d \
  --name vllm-qwen35-4b \
  --gpus '"device=0"' \
  --user 2000:0 \
  -p 49173:8000 \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  Qwen/Qwen3.5-4B \
  --gpu-memory-utilization 0.50 \
  --max-model-len 8192

docker logs -f vllm-qwen35-4b

curl http://localhost:49173/v1/models
```

we get an error like this:

```
(EngineCore pid=196) ValueError: max_num_seqs (256) exceeds available Mamba cache blocks (73). Each decode sequence requires one Mamba cache block, so CUDA graph capture cannot proceed. Please lower max_num_seqs to at most 73 or increase gpu_memory_utilization.
```

> A “Mamba cache block” is a chunk of GPU memory that stores the running internal state for one active sequence in a recurrent/linear-attention layer.

> That state contains information such as the linear-attention/SSM state and short convolution state. Its size is mostly fixed per active request rather than growing for every token.

Basically it does some sort of mathematically compression within its State Space Model (SSM) to compress all preceding sequence history into a single, static vector.

> During CUDA-graph preparation, vLLM attempted to prepare execution for as many as 256 concurrent sequences. But it could only assign recurrent state to 73 sequences.

Since we don't need 256 concurrent sequences, we can lower the limit drastically.

```bash
docker run -d \
  --name vllm-qwen35-4b \
  --gpus '"device=0"' \
  --user 2000:0 \
  -p 49173:8000 \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  Qwen/Qwen3.5-4B \
  --gpu-memory-utilization 0.50 \
  --max-model-len 8192 \
  --max-num-seqs 8
```

this worked. i also tested our vlm + llm loop and it wasn't noticably slower

### fixing some warnings

```
(/home/peilunli/code/env) root@depend:~/code# python tools/run_foundation_pose_node.py   --mesh_file /home/peilunli/code/models/red_block/model.obj   --color_topic /
  io/internal_camera/head_camera/image_raw   --depth_topic /io/internal_camera/head_camera/depth/image_raw   --init_color /home/peilunli/code/recordings/ros/
  ros_image_color.jpg   --init_depth /home/peilunli/code/recordings/ros/ros_image_depth.png   --init_mask /home/peilunli/code/recordings/ros/ros_image_mask.png
  FutureWarning: `torch.cuda.amp.custom_fwd(args...)` is deprecated. Please use `torch.amp.custom_fwd(args..., device_type='cuda')` instead.
  Warp 1.3.3 initialized:
     CUDA Toolkit 12.5, Driver 13.0
     Devices:
       "cpu"      : "x86_64"
       "cuda:0"   : "NVIDIA GeForce RTX 4090" (24 GiB, sm_89, mempool enabled)
       "cuda:1"   : "NVIDIA GeForce RTX 4090" (24 GiB, sm_89, mempool enabled)
     CUDA peer access:
       Not supported
     Kernel cache:
       /home/peilunli/.cache/warp/1.3.3
  [INFO] [1784732863.127970, 0.000000]: ROS node 'foundation_pose' initialized successfully.
  ^R
  [INFO] [1784733116.698727, 244.957000]: Initializing FoundationPose...
  FutureWarning: You are using `torch.load` with `weights_only=False` (the current default value), which uses the default pickle module implicitly. It is possible to
  construct malicious pickle data which will execute arbitrary code during unpickling (See https://github.com/pytorch/pytorch/blob/main/SECURITY.md#untrusted-models for
  more details). In a future release, the default value for `weights_only` will be flipped to `True`. This limits the functions that could be executed during
  unpickling. Arbitrary objects will no longer be allowed to be loaded via this mode unless they are explicitly allowlisted by the user via
  `torch.serialization.add_safe_globals`. We recommend you start setting `weights_only=True` for any use case where you don't have full control of the loaded file.
  Please open an issue on GitHub for any issues related to this experimental feature.
  FutureWarning: You are using `torch.load` with `weights_only=False` (the current default value), which uses the default pickle module implicitly. It is possible to
  construct malicious pickle data which will execute arbitrary code during unpickling (See https://github.com/pytorch/pytorch/blob/main/SECURITY.md#untrusted-models for
  more details). In a future release, the default value for `weights_only` will be flipped to `True`. This limits the functions that could be executed during
  unpickling. Arbitrary objects will no longer be allowed to be loaded via this mode unless they are explicitly allowlisted by the user via
  `torch.serialization.add_safe_globals`. We recommend you start setting `weights_only=True` for any use case where you don't have full control of the loaded file.
  Please open an issue on GitHub for any issues related to this experimental feature.
  UserWarning: TORCH_CUDA_ARCH_LIST is not set, all archs for visible cards are included for compilation.
  If this is not desired, please set os.environ['TORCH_CUDA_ARCH_LIST'].
  num original candidates = 42
  num of pose after clustering: 42
  Module third_party.FoundationPose.Utils 64702ec load on device 'cuda:0' took 0.41 ms  (cached)
  UserWarning: torch.set_default_tensor_type() is deprecated as of PyTorch 2.1, please use torch.set_default_dtype() and torch.set_default_device() as alternatives.
  (Triggered internally at ../torch/csrc/tensor/python_tensor.cpp:432.)
  FutureWarning: `torch.cuda.amp.autocast(args...)` is deprecated. Please use `torch.amp.autocast('cuda', args...)` instead.
  FutureWarning: `torch.cuda.amp.autocast(args...)` is deprecated. Please use `torch.amp.autocast('cuda', args...)` instead.
  UserWarning: The given NumPy array is not writable, and PyTorch does not support non-writable tensors. This means writing to this tensor will result in undefined
  behavior. You may want to copy the array to protect its data or make it writable before converting it to a tensor. This type of warning will be suppressed for the
  rest of this program. (Triggered internally at ../torch/csrc/utils/tensor_numpy.cpp:206.)
  ^C(/home/peilunli/code/env) root@depend:~/code# python tools/run_foundation_pose_node.py   --mesh_file /home/peilunli/code/models/red_block/model.obj
  --color_topic /io/internal_camera/head_camera/image_raw   --depth_topic /io/internal_camera/head_camera/depth/image_raw   --init_color /home/peilunli/code/recordings/
  ros/ros_image_color.jpg   --init_depth /home/peilunli/code/recordings/ros/ros_image_depth.png   --init_mask /home/peilunli/code/recordings/ros/ros_image_mask.png
  FutureWarning: `torch.cuda.amp.custom_fwd(args...)` is deprecated. Please use `torch.amp.custom_fwd(args..., device_type='cuda')` instead.
  Warp 1.3.3 initialized:
     CUDA Toolkit 12.5, Driver 13.0
     Devices:
       "cpu"      : "x86_64"
       "cuda:0"   : "NVIDIA GeForce RTX 4090" (24 GiB, sm_89, mempool enabled)
       "cuda:1"   : "NVIDIA GeForce RTX 4090" (24 GiB, sm_89, mempool enabled)
     CUDA peer access:
       Not supported
     Kernel cache:
       /home/peilunli/.cache/warp/1.3.3
  [INFO] [1784733292.578645, 0.000000]: ROS node 'foundation_pose' initialized successfully.
  [INFO] [1784733292.779550, 75.920000]: Initializing FoundationPose...
  FutureWarning: You are using `torch.load` with `weights_only=False` (the current default value), which uses the default pickle module implicitly. It is possible to
  construct malicious pickle data which will execute arbitrary code during unpickling (See https://github.com/pytorch/pytorch/blob/main/SECURITY.md#untrusted-models for
  more details). In a future release, the default value for `weights_only` will be flipped to `True`. This limits the functions that could be executed during
  unpickling. Arbitrary objects will no longer be allowed to be loaded via this mode unless they are explicitly allowlisted by the user via
  `torch.serialization.add_safe_globals`. We recommend you start setting `weights_only=True` for any use case where you don't have full control of the loaded file.
  Please open an issue on GitHub for any issues related to this experimental feature.
  FutureWarning: You are using `torch.load` with `weights_only=False` (the current default value), which uses the default pickle module implicitly. It is possible to
  construct malicious pickle data which will execute arbitrary code during unpickling (See https://github.com/pytorch/pytorch/blob/main/SECURITY.md#untrusted-models for
  more details). In a future release, the default value for `weights_only` will be flipped to `True`. This limits the functions that could be executed during
  unpickling. Arbitrary objects will no longer be allowed to be loaded via this mode unless they are explicitly allowlisted by the user via
  `torch.serialization.add_safe_globals`. We recommend you start setting `weights_only=True` for any use case where you don't have full control of the loaded file.
  Please open an issue on GitHub for any issues related to this experimental feature.
  UserWarning: TORCH_CUDA_ARCH_LIST is not set, all archs for visible cards are included for compilation.
  If this is not desired, please set os.environ['TORCH_CUDA_ARCH_LIST'].
  num original candidates = 42
  num of pose after clustering: 42
  Module third_party.FoundationPose.Utils 64702ec load on device 'cuda:0' took 0.51 ms  (cached)
  UserWarning: torch.set_default_tensor_type() is deprecated as of PyTorch 2.1, please use torch.set_default_dtype() and torch.set_default_device() as alternatives.
  (Triggered internally at ../torch/csrc/tensor/python_tensor.cpp:432.)
  FutureWarning: `torch.cuda.amp.autocast(args...)` is deprecated. Please use `torch.amp.autocast('cuda', args...)` instead.
  FutureWarning: `torch.cuda.amp.autocast(args...)` is deprecated. Please use `torch.amp.autocast('cuda', args...)` instead.
  UserWarning: The given NumPy array is not writable, and PyTorch does not support non-writable tensors. This means writing to this tensor will result in undefined
  behavior. You may want to copy the array to protect its data or make it writable before converting it to a tensor. This type of warning will be suppressed for the
  rest of this program. (Triggered internally at ../torch/csrc/utils/tensor_numpy.cpp:206.)
```

add configs to `~/.bashrc`

```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID
CUDA_VISIBLE_DEVICES=0
TORCH_CUDA_ARCH_LIST=8.9
PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True
```

### massive weird detection

here is how the formats should be:

- `/io/internal_camera/head_camera/depth/image_raw/encoding`: this is a `sensor_msgs/Image` message with encoding `32FC1`, meaning 32 bits per channel, floating point, and one channel. these numbers are in meters by ROS depth camera convention.
- FoundationPose: expects 16-bit, single-channel PNG, with integer pixel values in millimeters

currently we directly convert the 32 bit float meters into 8 bit uint (which are then read as mm), which is not good

```python
  cv2.imwrite("ros_image_depth.png", depth)
```

basically all the depths are read as small millimeters, making the detection weird and very close to the camera

### fix coordinate frames

see [https://github.com/BU-DEPEND-Lab/RISE-2026/compare/fdc8dce...6ed634d]()

Basically we made the sawyer model properly at world z = 0, and the base frame +0.93 z from it. ik_request takes relative to base, not to sawyer.

Before, the entire Sawyer Gazebo model was spawned at world z = 0.93, but ROS incorrectly published base at world z = 0; this broke the TF tree because head_camera is defined relative to base, placing its TF pose 0.93 m below the simulated camera

ik_request still worked before because the Gazebo sawyer model frame accidentally still provided the base-relative coordinates expected by ik_request.

### initialization pose

turns out foundationpose is a tracking library so the initial image is supposed to be the initial pose.

i think giving it initial pose to start is probably easier than the image then

get the initial pose:

```
root@depend:~/ros_ws# rosrun tf tf_echo head_camera red_block_1
At time 3.638
- Translation: [0.025, -0.002, 1.041]
- Rotation: in Quaternion [0.671, 0.671, -0.223, 0.223]
            in RPY (radian) [3.142, 0.642, 1.571]
            in RPY (degree) [180.000, 36.761, 90.000]
```

and we can use it directly with initial pose passed to foundationpose

```bash
python tools/run_foundation_pose_node.py \
  --mesh_file /home/peilunli/code/models/red_block/model.obj \
  --color_topic /io/internal_camera/head_camera/image_raw \
  --depth_topic /io/internal_camera/head_camera/depth/image_raw \
  --camera_frame_id head_camera \
  --init_pose \
    0.671 0.671 -0.223 0.223 \
    0.025 -0.002 1.041
```

### multiple objects

we need a manager to call foundationpose on multiple objects

### stuff to look into from Dr Li

specification guided steering
[https://cacm.acm.org/research/specification-guided-reinforcement-learning/]()

control barrier functions
