# 260720

## tasks

- need to figure out direction in regards to camera feed!!!
- potential future todo: publish an id for a given cycle
- potentially revisit 019f80d6-16b3-7122-ae8c-50ec6738ea09 for refactor

## sources

## learned

## log

### add back red and blue blocks

since it can't pick up the apple we will use these instead.

we will not try to train yolo on detecting it since we aren't planning on using it, but hopefully foundationpose can detect the blocks

### vlm loop

- `controller/llm.py` stays the same. it starts inference based on `/llm/instruction` whenever `/controller/start` is published, and publishes `/controller/stop` when finished
- `controller/instruction.py` publishes `/controller/start` when it has all the context and has received vlm instruction from `/vlm/output`
- `vlm/vlm_node.py` runs vlm inference everytime `/controller/stop` is published, and stops the loop when it gets "end" as output

### testing vlm loop

task: "move the blocks to the center of the table"

observations and solutions

- only outputting one line -> fixed part of instruction telling it to do that
- was repeating the same action over and over -> add an example where one object is already at the center
- was still repeating because it thought "near center" was close enough to center -> use a near object for that example and explicitly say near is good enough
- sometimes trying to move fruit instead of blocks -> explicitly say examples can have different tasks than the actual task
- sometimes pushing instead of pnp the blocks -> define push better in llm instruction
- too strict on being "near enough" -> in task say go within 0.1 units of center

### race condition

there's an issue where the llm uses a stale vlm output instead of the newest one when it is triggered, because `/llm/instruction` isn't guaranteed to be published before `/controller/start`

i fixed this by getting rid of `/controller/start` and making `/llm/instruction` trigger an llm inference. it's published whenever vlm output is published in the callback in `instruction.py`

### model issues (not trying to fix right now, just documenting)

llm sometimes skips steps like descending

```

Note: The example output format includes redundant moves sometimes, but I will provide the logical sequence. The prompt asks for "an ordered list of actions".

[
{"action": "open_gripper"},
{"action": "move", "x": 0.675, "y": 0.325, "z": 0.090},
{"action": "move", "x": 0.675, "y": 0.325, "z": -0.155},
{"action": "close_gripper"},
{"action": "move", "x": 0.675, "y": 0.325, "z": 0.090},
{"action": "move", "x": 0.750, "y": 0.000, "z": 0.090},
{"action": "open_gripper"}
]



LLM inference took 10.862 seconds
moving to: x=0.675 y=0.325 z=0.09
moving to: x=0.675 y=0.325 z=-0.155
closing gripper
moving to: x=0.675 y=0.325 z=0.09
moving to: x=0.75 y=0.0 z=0.09
opening gripper
```

vlm has too strict requirements for close

```
1.  **Analyze the scene and task**: The task is to move the blocks (yellow_block_1, red_block_1, blue_block_1) to the center of the table.
2.  **Identify object locations**:
    *   **Table Center**: (0.750, 0.000).
    *   **Yellow Block**: (0.785, -0.010). This is very close to the center (x diff ~0.035, y diff ~0.01).
    *   **Red Block**: (0.729, 0.020). This is also very close to the center (x diff ~0.021, y diff ~0.02).
    *   **Blue Block**: (0.728, -0.039). This is slightly further away (y diff ~0.04).
3.  **Determine next action**: The yellow block and red block are already nearly at the center. The blue block is the furthest from the center among the blocks. Therefore, the most logical next step is to move the blue block to the center.
4.  **Formulate command**: "Move the blue block to the center of the table."

Move the blue block to the center of the table.
```

(trying a stacking task) it needs to be more aware that its part of a sequence and some steps may already be done:

```

1.  **Analyze the Task:** The goal is to stack the blocks (red_block_1, blue_block_1, yellow_block_1) on top of each other in the center of the table.

2.  **Analyze Object Locations:**
    *   **Table Center:** Approximately (x=0.750, y=0.000).
    *   **Yellow Block:** Center (0.756, 0.003). This is very close to the table center.
    *   **Red Block:** Center (0.875, 0.025). This is relatively close to the center, slightly to the right.
    *   **Blue Block:** Center (0.675, 0.325). This is further away, to the left.

3.  **Analyze Robot State:** The image shows the robot's gripper is currently positioned over the yellow block. The gripper appears to be holding or interacting with the yellow block.

4.  **Determine Next Action:**
    *   The yellow block is the best candidate for the base of the stack because it is closest to the center.
    *   The robot is already positioned to interact with the yellow block.
    *   To start the stacking process, the robot needs to pick up the yellow block.
    *   Once picked up, the robot will need to move it to the center and place it down.
    *   Therefore, the immediate action is to grasp the yellow block.

5.  **Formulate Command:** "Grasp the yellow block."

Grasp the yellow block.
```

also, this hasn't caused issues yet but TODO need to figure out direction. i'm pretty sure because of where the camera is, the camera directions are inverted from the directions of the coordinate system

### full run log

#### task

```
Move the blocks to within 0.1 units from the center of the table.
```

```

VLM fetching response...
VLM response:
1.  **Analyze the Task**: The goal is to move all blocks (yellow_block_1, blue_block_1, red_block_1) to within 0.1 units of the table center.
2.  **Identify Target Location**: The table center is at (x=0.750, y=0.000).
3.  **Evaluate Object Positions**:
    *   **yellow_block_1**: Center at (0.575, -0.155). Distance to center: $\sqrt{(0.575-0.750)^2 + (-0.155-0.000)^2} \approx \sqrt{0.0306 + 0.0240} \approx 0.23$. This is > 0.1. Needs moving.
    *   **blue_block_1**: Center at (0.675, 0.325). Distance to center: $\sqrt{(0.675-0.750)^2 + (0.325-0.000)^2} \approx \sqrt{0.0056 + 0.1056} \approx 0.33$. This is > 0.1. Needs moving.
    *   **red_block_1**: Center at (0.875, 0.025). Distance to center: $\sqrt{(0.875-0.750)^2 + (0.025-0.000)^2} \approx \sqrt{0.0156 + 0.0006} \approx 0.127$. This is > 0.1. Needs moving.
4.  **Select Action**: All three blocks need to be moved. I will choose the red_block_1 as it is relatively close to the center but still outside the threshold, and it is positioned slightly to the right (+x). Moving it towards the center (left) will bring it closer.
5.  **Formulate Command**: "Move the red block to the center of the table."

Move the red block to the center of the table.
[INFO] [1784575189.760530, 10.860000]: Waiting for the current subtask to finish
LLM fetching response...
LLM response:
opening gripper
The task is to move the red block to the center of the table. The task uses the word "move", not "push". Therefore, this is a pick and place task.

Step by step plan:
1. Open the gripper.
2. Move to a safe height (above the object's zmax + 0.2m or current z if higher). The red block's zmax is -0.110. Safe height = -0.110 + 0.2 = 0.090. Current z is 0.020, which is below safe height. Move to 0.090.
3. Move to an approach pose above the center of the red block. Red block center: x=0.875, y=0.025.
4. Descend to the object's minimum z (zmin = -0.155) to grasp it.
5. Close the gripper.
6. Lift the object to the safe approach height (0.090).
7. Move to the drop location above the table center. Table center: x=0.750, y=0.000.
8. Descend to be right above the table center (slightly above zmax of table to avoid collision, or just above the target surface). Table zmax is -0.155. Let's aim for z = -0.150.
9. Open the gripper to place the object.

Calculations:
- Object: red_block_1 (x=0.875, y=0.025, z=-0.133, zmin=-0.155, zmax=-0.110)
- Table Center: (0.750, 0.000, -0.175)
- Safe Height: -0.110 + 0.2 = 0.090
- Approach Height: 0.090
- Grasp Height: -0.155
- Drop X, Y: 0.750, 0.000
- Drop Z: -0.150 (just above table surface)

Sequence:
1. Open gripper.
2. Move to (0.875, 0.025, 0.090).
3. Move to (0.875, 0.025, -0.155).
4. Close gripper.
5. Move to (0.750, 0.000, 0.090).
6. Move to (0.750, 0.000, -0.150).
7. Open gripper.

[
{"action": "open_gripper"},
{"action": "move", "x": 0.875, "y": 0.025, "z": 0.090},
{"action": "move", "x": 0.875, "y": 0.025, "z": -0.155},
{"action": "close_gripper"},
{"action": "move", "x": 0.750, "y": 0.000, "z": 0.090},
{"action": "move", "x": 0.750, "y": 0.000, "z": -0.150},
{"action": "open_gripper"}
]



LLM inference took 8.900 seconds
moving to: x=0.875 y=0.025 z=0.09
moving to: x=0.875 y=0.025 z=-0.155
closing gripper
moving to: x=0.75 y=0.0 z=0.09
moving to: x=0.75 y=0.0 z=-0.15
opening gripper
[INFO] [1784575213.857789, 34.904000]: Current subtask finished; requesting the next one
VLM inference took 29.749 seconds
VLM fetching response...
VLM response:
1.  **Analyze the Task**: The goal is to move all blocks (yellow_block_1, blue_block_1, red_block_1) to within 0.1 units of the table center (x=0.750, y=0.000).

2.  **Evaluate Current Positions**:
    *   **Table Center**: (0.750, 0.000)
    *   **Yellow Block**: (0.575, -0.155). Distance to center: $\sqrt{(0.575-0.750)^2 + (-0.155-0.000)^2} \approx \sqrt{0.0306 + 0.0240} \approx 0.23$. This is > 0.1. Needs moving.
    *   **Blue Block**: (0.675, 0.325). Distance to center: $\sqrt{(0.675-0.750)^2 + (0.325-0.000)^2} \approx \sqrt{0.0056 + 0.1056} \approx 0.33$. This is > 0.1. Needs moving.
    *   **Red Block**: (0.758, 0.006). Distance to center: $\sqrt{(0.758-0.750)^2 + (0.006-0.000)^2} \approx \sqrt{0.000064 + 0.000036} \approx 0.012$. This is < 0.1. The red block is already in the correct zone.

3.  **Select Next Action**: The robot needs to move the yellow block and the blue block. The yellow block is at (0.575, -0.155), which is to the left and slightly forward relative to the center. The blue block is at (0.675, 0.325), which is to the left and significantly forward.
    Looking at the image, the robot arm is positioned near the red block. The yellow block is visible to the left of the red block. Moving the yellow block is a logical first step as it is relatively close to the center compared to the blue block.

4.  **Formulate Command**: "Move the yellow block to the center of the table."

Move the yellow block to the center of the table.
[INFO] [1784575220.114004, 41.134000]: Waiting for the current subtask to finish
LLM fetching response...
LLM response:
The task is to move the yellow block to the center of the table. This is a pick and place task because the task does not explicitly use the word "push".

Step by step plan:
1. Open the gripper.
2. Move to a safe height (above the object's zmax) to avoid collisions. The yellow block's zmax is -0.110, so a safe height of -0.110 + 0.2 = 0.090 m is appropriate.
3. Move to an approach pose above the center of the yellow block (x=0.575, y=-0.155).
4. Descend to the object's minimum z (zmin = -0.155) to grasp it.
5. Close the gripper to secure the block.
6. Lift the block straight up to the safe approach height (0.090 m).
7. Move to the drop location above the center of the table. The table center is at x=0.750, y=-0.000.
8. Descend to be right above the table center (z=0.090 m).
9. Open the gripper to release the block.

[
{"action": "open_gripper"},
{"action": "move", "x": 0.575, "y": -0.155, "z": 0.090},
{"action": "move", "x": 0.575, "y": -0.155, "z": -0.155},
{"action": "close_gripper"},
{"action": "move", "x": 0.575, "y": -0.155, "z": 0.090},
{"action": "move", "x": 0.750, "y": -0.000, "z": 0.090},
{"action": "move", "x": 0.750, "y": -0.000, "z": -0.090},
{"action": "open_gripper"}
]



LLM inference took 5.325 seconds
opening gripper
moving to: x=0.575 y=-0.155 z=0.09
moving to: x=0.575 y=-0.155 z=-0.155
closing gripper
moving to: x=0.575 y=-0.155 z=0.09
moving to: x=0.75 y=-0.0 z=0.09
moving to: x=0.75 y=-0.0 z=-0.09
opening gripper
[INFO] [1784575241.381686, 62.352000]: Current subtask finished; requesting the next one
VLM inference took 27.448 seconds
VLM fetching response...
VLM response:
1.  **Analyze the Task**: The goal is to move all blocks (yellow_block_1, blue_block_1, red_block_1) to within 0.1 units of the table center (x=0.750, y=0.000).

2.  **Evaluate Object Positions**:
    *   **Table Center**: (0.750, 0.000)
    *   **Yellow Block**: (0.771, -0.002). Distance to center: $\sqrt{(0.771-0.750)^2 + (-0.002-0.000)^2} \approx 0.021$. This is well within the 0.1 unit threshold.
    *   **Blue Block**: (0.675, 0.325). Distance to center: $\sqrt{(0.675-0.750)^2 + (0.325-0.000)^2} \approx \sqrt{0.0056 + 0.1056} \approx 0.33$. This is far from the center.
    *   **Red Block**: (0.751, 0.003). Distance to center: $\sqrt{(0.751-0.750)^2 + (0.003-0.000)^2} \approx 0.003$. This is well within the 0.1 unit threshold.

3.  **Determine Next Action**: The yellow and red blocks are already in the correct position. The blue block is the only one that needs to be moved. It is located at x=0.675, y=0.325. The robot needs to grasp and move it to the center.

4.  **Formulate Command**: Address the robot directly and instruct it to move the blue block to the center of the table.

Move the blue block to the center of the table.
[INFO] [1784575246.455483, 67.412000]: Waiting for the current subtask to finish
LLM fetching response...
LLM response:
The task is to move the blue block to the center of the table. The task uses the word "move", not "push". Therefore, this is a pick and place task.

Step by step plan:
1. Open the gripper.
2. Move to a safe height (above the object's zmax + 0.2m or current safe height). The blue block's zmax is -0.110. Safe height = -0.110 + 0.2 = 0.090. Current z is -0.090, so we need to move up.
3. Move to an approach pose above the center of the blue block.
4. Descend to the object's minimum z (-0.155) to grasp it.
5. Close the gripper.
6. Lift the object to the safe approach height (0.090).
7. Move to the drop location above the table center (0.750, -0.000, 0.090).
8. Descend to be right above the table center (z = 0.090).
9. Open the gripper to place the object.

Calculations:
- Object: blue_block_1, center (0.675, 0.325, -0.133), zmin=-0.155, zmax=-0.110.
- Table center: (0.750, -0.000, -0.175).
- Safe height: -0.110 + 0.2 = 0.090.
- Approach point above object: (0.675, 0.325, 0.090).
- Grasp point: (0.675, 0.325, -0.155).
- Drop point above table center: (0.750, -0.000, 0.090).

Action sequence:
1. Open gripper.
2. Move to safe height (0.090).
3. Move to approach pose (0.675, 0.325, 0.090).
4. Move down to grasp (0.675, 0.325, -0.155).
5. Close gripper.
6. Move up to safe height (0.675, 0.325, 0.090).
7. Move to drop location (0.750, -0.000, 0.090).
8. Move down to place (0.750, -0.000, 0.090). Note: The rule says "Descend to be right above the target location" then "Open". Since the target is on the table, "above" implies the safe height z.
9. Open gripper.

[
{"action": "open_gripper"},
{"action": "move", "x": 0.675, "y": 0.325, "z": 0.090},
{"action": "move", "x": 0.675, "y": 0.325, "z": -0.155},
{"action": "close_gripper"},
{"action": "move", "x": 0.675, "y": 0.325, "z": 0.090},
{"action": "move", "x": 0.750, "y": -0.000, "z": 0.090},
{"action": "move", "x": 0.750, "y": -0.000, "z": 0.090},
{"action": "open_gripper"}
]



opening gripper
LLM inference took 9.496 seconds
moving to: x=0.675 y=0.325 z=0.09
moving to: x=0.675 y=0.325 z=-0.155
closing gripper
moving to: x=0.675 y=0.325 z=0.09
moving to: x=0.75 y=-0.0 z=0.09
moving to: x=0.75 y=-0.0 z=0.09
opening gripper
[INFO] [1784575271.663302, 92.549000]: Current subtask finished; requesting the next one
VLM inference took 30.197 seconds
VLM fetching response...
VLM response:
1.  **Analyze the Task:** The goal is to move all blocks (yellow_block_1, blue_block_1, red_block_1) to within 0.1 units of the table center.
2.  **Identify Target Location:** The table center is at (x=0.750, y=0.000).
3.  **Evaluate Object Positions:**
    *   **yellow_block_1:** Center at (0.799, -0.001). Distance to center: $\sqrt{(0.799-0.750)^2 + (-0.001-0.000)^2} \approx 0.049$. This is within 0.1 units.
    *   **blue_block_1:** Center at (0.696, -0.007). Distance to center: $\sqrt{(0.696-0.750)^2 + (-0.007-0.000)^2} \approx 0.054$. This is within 0.1 units.
    *   **red_block_1:** Center at (0.750, 0.003). Distance to center: $\sqrt{(0.750-0.750)^2 + (0.003-0.000)^2} \approx 0.003$. This is within 0.1 units.
4.  **Determine Action:** All three blocks are already within the required tolerance of the table center. No movement is needed.
5.  **Formulate Command:** Since the task is complete, output "end".

end
[INFO] [1784575275.967478, 96.844000]: VLM reported that the task is complete
[INFO] [1784575275.967498, 96.844000]: VLM reported that the task is complete
VLM inference took 4.295 seconds
[INFO] [1784575275.968663, 96.845000]: VLM inference loop stopped after end
```

### memory

wontfix
