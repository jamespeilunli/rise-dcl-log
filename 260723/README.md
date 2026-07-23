# 260723

## tasks

## sources

1. [specification guided RL](https://cacm.acm.org/research/specification-guided-reinforcement-learning/)

## log

### specification guided steering review

source 1

### foundationpose multiple objects

#### issues

- while gripping the object can rotate a lot and cause a tracking loss
- when the object is high up and its background is partially or fully off the table from the camera's perspective while the block in the gripper, foundation pose really struggles with tracking it, usually it starts tracking the place where it left the table background
- when dropping from the gripper the object often sticks to the gripper

#### tracking priority

here is the problem:

- the more frequently we run foundationpose on a given object or frame, the better the tracking is
- we have a limited amount of gpu cycles to run foundationpose at a given time

constraints

- it's very important to track a moving object accurately
- objects typically only move via the robot arm

solution:

- this is a resource allocation problem
- we design a scheduler to prioritize moving objects
- before, we had 5 refine iterations on the same frame for each object (5 iterations \* 4 objects = 20 runs)
- we can prioritize the object closest to the gripper
- let's try giving 5 refine iterations to the prioritized object every frame, then one background object tracks every fifth active frame with 1 iteration. choose background objects in round robin fashion

example:

```
[INFO] [1784821805.552598, 8.560000]: FoundationPose scheduler over 5.1s: priority=blue_block_1, active=10 (1.95 Hz, 344.6 ms avg), background=2 (0.39 Hz, 106.3 ms avg)
[INFO] [1784821810.568743, 13.535000]: FoundationPose scheduler over 5.0s: priority=blue_block_1, active=12 (2.39 Hz, 353.4 ms avg), background=2 (0.40 Hz, 72.4 ms avg)
[INFO] [1784821815.642318, 18.581000]: FoundationPose scheduler over 5.1s: priority=blue_block_1, active=11 (2.17 Hz, 389.1 ms avg), background=2 (0.39 Hz, 96.7 ms avg)
[INFO] [1784821820.671316, 23.564000]: FoundationPose scheduler over 5.0s: priority=blue_block_1, active=21 (4.18 Hz, 178.2 ms avg), background=4 (0.80 Hz, 45.7 ms avg)
[INFO] [1784821821.211856, 24.101000]: FoundationPose priority object set to 'red_block_1'
[INFO] [1784821825.757078, 28.608000]: FoundationPose scheduler over 5.1s: priority=red_block_1, active=29 (5.70 Hz, 116.8 ms avg), background=6 (1.18 Hz, 29.1 ms avg)
[INFO] [1784821830.786992, 33.579000]: FoundationPose scheduler over 5.0s: priority=red_block_1, active=29 (5.77 Hz, 114.7 ms avg), background=6 (1.19 Hz, 30.1 ms avg)
[INFO] [1784821835.791069, 38.532000]: FoundationPose scheduler over 5.0s: priority=red_block_1, active=24 (4.80 Hz, 149.8 ms avg), background=4 (0.80 Hz, 36.0 ms avg)
[INFO] [1784821840.826655, 43.535000]: FoundationPose scheduler over 5.0s: priority=red_block_1, active=11 (2.18 Hz, 374.8 ms avg), background=3 (0.60 Hz, 104.9 ms avg)
[INFO] [1784821846.129851, 48.795000]: FoundationPose scheduler over 5.3s: priority=red_block_1, active=12 (2.26 Hz, 371.0 ms avg), background=2 (0.38 Hz, 98.8 ms avg)
[INFO] [1784821851.482389, 54.092000]: FoundationPose scheduler over 5.4s: priority=red_block_1, active=12 (2.24 Hz, 374.5 ms avg), background=2 (0.37 Hz, 106.9 ms avg)
[INFO] [1784821856.897106, 59.472000]: FoundationPose scheduler over 5.4s: priority=red_block_1, active=12 (2.22 Hz, 373.3 ms avg), background=3 (0.55 Hz, 101.5 ms avg)
[INFO] [1784821862.112543, 64.648000]: FoundationPose scheduler over 5.2s: priority=red_block_1, active=12 (2.30 Hz, 365.9 ms avg), background=2 (0.38 Hz, 99.4 ms avg)
[INFO] [1784821866.000160, 68.500000]: FoundationPose priority object set to 'blue_block_1'
[INFO] [1784821867.231683, 69.720000]: FoundationPose scheduler over 5.1s: priority=blue_block_1, active=29 (5.67 Hz, 117.1 ms avg), background=6 (1.17 Hz, 33.3 ms avg)
[INFO] [1784821872.246425, 74.652000]: FoundationPose scheduler over 5.0s: priority=blue_block_1, active=28 (5.58 Hz, 119.0 ms avg), background=6 (1.20 Hz, 31.2 ms avg)
[INFO] [1784821877.369864, 79.719000]: FoundationPose scheduler over 5.1s: priority=blue_block_1, active=29 (5.66 Hz, 118.3 ms avg), background=5 (0.98 Hz, 32.1 ms avg)
[INFO] [1784821882.408889, 84.699000]: FoundationPose scheduler over 5.0s: priority=blue_block_1, active=22 (4.37 Hz, 167.4 ms avg), background=5 (0.99 Hz, 41.2 ms avg)
```

#### moving stuff around

- lowered safe height from +0.2m to +0.1m
- angled camera down more

this is all to reduce the chance the cube enters a non table area as its background, which i observed to mess up the tracking

### hook up foundationpose

```
The task is to move the red block to the center of the table. The task uses the word "move", not "push". Therefore, this is a pick and place task.

Step by step plan:
1. Open the gripper.
2. Move to a safe height (above the object's zmax) to avoid collisions. The red block's zmax is -0.110, so a safe height of -0.010 m is appropriate.
3. Move to an approach pose above the center of the red block (x=0.936, y=0.039).
4. Descend to the object's minimum z (zmin = -0.155) to grasp it.
5. Close the gripper to grasp the red block.
6. Lift the object straight up to the safe approach height (-0.010 m).
7. Move to the drop location above the center of the table (cafe_table center: x=0.750, y=0.000).
8. Descend to be right above the table center (z = -0.010 m).
9. Open the gripper to place the object.

Note: The current end-effector position is (0.750, 0.000, -0.132). The safe height is -0.010 m. We need to move up first.

[
{"action": "open_gripper"},
{"action": "move", "x": 0.936, "y": 0.039, "z": -0.010},
{"action": "move", "x": 0.936, "y": 0.039, "z": -0.155},
{"action": "close_gripper"},
{"action": "move", "x": 0.936, "y": 0.039, "z": -0.010},
{"action": "move", "x": 0.750, "y": 0.000, "z": -0.010},
{"action": "move", "x": 0.750, "y": 0.000, "z": -0.132},
{"action": "open_gripper"}
]
opening gripper
```

```

VLM response:
1.  **Analyze the Goal**: The task is to move all blocks (blue, red, yellow, green) to within 0.1 units of the table center.
2.  **Identify Target Location**: The table center is at (x=0.750, y=0.000).
3.  **Evaluate Current Positions**:
    *   **Red Block**: Center at (0.875, 0.025). Distance to center: $\sqrt{(0.875-0.750)^2 + (0.025-0.000)^2} \approx \sqrt{0.0156 + 0.0006} \approx 0.127$. This is slightly outside the 0.1 threshold.
    *   **Yellow Block**: Center at (0.575, -0.155). Distance to center: $\sqrt{(0.575-0.750)^2 + (-0.155-0.000)^2} \approx \sqrt{0.0306 + 0.0240} \approx 0.23$. Far away.
    *   **Green Block**: Center at (0.675, 0.225). Distance to center: $\sqrt{(0.675-0.750)^2 + (0.225-0.000)^2} \approx \sqrt{0.0056 + 0.0506} \approx 0.23$. Far away.
    *   **Blue Block**: Center at (0.675, 0.325). Distance to center: $\sqrt{(0.675-0.750)^2 + (0.325-0.000)^2} \approx \sqrt{0.0056 + 0.1056} \approx 0.33$. Far away.
4.  **Select Next Action**: The red block is the closest to the target but still slightly out of range (0.127 > 0.1). Moving it is the most logical immediate step to reduce the error. The other blocks are significantly further away.
5.  **Formulate Command**: I need to pick up the red block and place it on the table near the center.

Move the red block to the center of the table.
[INFO] [1784834869.169089, 11.611000]: Waiting for the current subtask to finish
LLM fetching response...
LLM response:
The task is to move the red block to the center of the table. The task uses the word "move", not "push". Therefore, this is a pick and place task.

Step by step plan:
1. Open the gripper.
2. Move to a safe height (above the object's zmax) to avoid collisions. The red block's zmax is -0.110, so a safe height of 0.020 (current height) or slightly higher is appropriate. Since the current z is 0.020, we are already at a safe height.
3. Move to an approach pose above the center of the red block.
4. Descend to the object's minimum z (-0.155) to grasp it.
5. Close the gripper.
6. Lift the object to the safe approach height (0.020).
7. Move to the drop location above the table center. The table center is at x=0.750, y=0.000.
8. Descend to be right above the table center.
9. Open the gripper to place the object.

Calculations:
- Object center (red_block_1): x=0.875, y=0.025, z=-0.133
- Object zmin: -0.155
- Object zmax: -0.110
- Safe height: 0.020 (current z is sufficient)
- Approach point above object: x=0.875, y=0.025, z=0.020
- Grasp point: x=0.875, y=0.025, z=-0.155
- Drop location above table center: x=0.750, y=0.000, z=0.020
- Place point: x=0.750, y=0.000, z=-0.133 (approximate drop height, usually just above the surface)

Action sequence:
1. Open gripper.
2. Move to approach above red block (0.875, 0.025, 0.020).
3. Move down to grasp (0.875, 0.025, -0.155).
4. Close gripper.
5. Move up to safe height (0.875, 0.025, 0.020).
6. Move to drop location above table center (0.750, 0.000, 0.020).
7. Move down to place (0.750, 0.000, -0.133).
8. Open gripper.

Note: The current end-effector position is (0.458, 0.148, 0.020). We need to move to the approach point first.

[
{"action": "open_gripper"},
{"action": "move", "x": 0.875, "y": 0.025, "z": 0.020},
{"action": "move", "x": 0.875, "y": 0.025, "z": -0.155},
{"action": "close_gripper"},
{"action": "move", "x": 0.875, "y": 0.025, "z": 0.020},
{"action": "move", "x": 0.750, "y": 0.000, "z": 0.020},
{"action": "move", "x": 0.750, "y": 0.000, "z": -0.133},
{"action": "open_gripper"}
]
```

it was off a little, somehow offsetting the pose goal by +0.01x worked

### overall

we still have some issues with losing tracking... may need to add object detection
