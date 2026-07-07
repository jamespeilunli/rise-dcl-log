# 260706

## tasks

- correct camera location in sim
- add more objects into simulation
- set up high level vlm as a ros node (takes in high level instruction (str), outputs low level instruction (str))
- test the vlm

later

- figure out good pose control (without ai) first, look into libraries

- figure out how to stop the program properly on ctrl c :sob:

## sources

1. [moveit](https://moveit.github.io/moveit_tutorials/)
2. [list of gazebo materials](https://wiki.ros.org/simulator_gazebo/Tutorials/ListOfMaterials)
3. [qwen 3.5 hf model](https://huggingface.co/Qwen/Qwen3.5-9B)
4. [vllm docker non root tutorial](https://docs.vllm.ai/en/stable/deployment/docker/#run-as-a-non-root-user)
5. [swri console TODO look into this later](https://github.com/swri-robotics/swri_console)
6. [openai vision docs](https://developers.openai.com/api/docs/guides/images-vision)

## learned

### `output="screen"`

this inside a `<node>` in the launch file will put print and statements to the std output

### create ros package

```bash
catkin_create_pkg <package_name> rospy <more dependencies...>
```

create a launch file in launch/ and python script in scripts/

add this to `CMakeLists.txt`

```
catkin_install_python(PROGRAMS
  scripts/<script_name>.py
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
```

```bash
catkin_make
```

and finally, include it in the launch file

```
<include file="$(find <package_name>)/launch/<name>.launch" />
```

reminder: your script must include:

```python
rospy.init_node("<node_name>")
```

## log

### task meeting discussion

- VLM: qwen 3.5 9b probably good (source 3). use something small
- look into "linear temporal logic", similar to the CodeGraphVLP approach, encoding behavior in a logic formula
- moveit for closed loop arm control potentially (see source 1)

### small changes to bashrc

```bash
source /root/ros_ws/devel/setup.bash
export DISPLAY=":12"
cd /root/ros_ws
```

### a combo for launching

got annoyed at waiting for the sigterm every time i ctrl c, so now i do this

```bash
roslaunch sawyer_sim_examples sawyer_pick_and_place_demo.launch; kill $(ps aux | grep gazebo | awk '{print $2}'); fg
```

now just need to ctrl z then ctrl c, and it will end the process instantly

probably bad practice but i didn't figure out how to make ctrl c close cleanly

### correct camera location in sim

- need to change /io/internal_camera/head_camera
- Gazebo/URDF tooling treats world specially in a lot of places
- we change the parent link to base instead
- the camera coordinate system is also a little weird, it looks like +z is forward, +x is right, +y is down in regard to the camera feed

### add more objects into simulation

- figured out how to add multiple of the same red block, just by reusing the same model and adding nodes for them in the launch file
- figured out how to add a custom banana model i got from https://github.com/yding25/URDF_models/tree/main
  - copied into `src/sawyer_simulator/sawyer_sim_examples/models`
  - edited `filename` fields like `collision.obj` into `package://sawyer_sim_examples/models/plastic_banana/collision.obj`
  - had to make it not transparent in `textured.mtl`: with `d 1.000000` for opacity and `Tr 0.000000` for transparency
  - added node to launch file
- and all of these i added to `gazebo_object_tf.py` to broadcast their tf
  figured out how to add blue block by copying red block model and changing material (see source 2)

### vlm on ros node

#### vllm setup

run vllm in separate docker container (see source 4)

```bash
docker run -d \
  --name vllm-qwen35-9b \
  --gpus all \
  --user 2000:0 \
  -p 49173:8000 \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  Qwen/Qwen3.5-4B
```

the 9b model took too much memory hence this error so i switched to 4b

```
(EngineCore pid=194) torch.OutOfMemoryError: CUDA out of memory. Tried to allocate 1.03 GiB. GPU 0 has a total capacity of 23.52 GiB of which 904.25 MiB is free. Process 503076 has 9.91 MiB memory in use. Process 3112211 has 22.61 GiB memory in use. Of the allocated memory 21.95 GiB is allocated by PyTorch, and 183.56 MiB is reserved by PyTorch but unallocated. If reserved but unallocated memory is large try setting PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True to avoid fragmentation.  See documentation for Memory Management  (https://docs.pytorch.org/docs/stable/notes/cuda.html#optimizing-memory-usage-with-pytorch-cuda-alloc-conf)
```

test that we can use the api from the ros docker container (see source 3)

```bash
curl "http://localhost:49173/v1/models"
```

```bash
curl -X POST "http://localhost:49173/v1/chat/completions" \
	-H "Content-Type: application/json" \
	--data '{
		"model": "Qwen/Qwen3.5-4B",
		"messages": [
			{
				"role": "user",
				"content": [
					{
						"type": "text",
						"text": "Describe this image in one sentence."
					},
					{
						"type": "image_url",
						"image_url": {
							"url": "https://cdn.britannica.com/61/93061-050-99147DCE/Statue-of-Liberty-Island-New-York-Bay.jpg"
						}
					}
				]
			}
		]
	}'
```

#### creating ros node

using instructions for creating package from learned

we use chatgpt responses api
