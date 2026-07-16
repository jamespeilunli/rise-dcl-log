# 260716

## tasks

path to hierarchical vla setup

low level controller

- try pushing task

high level vlm planner

- let it plan out text subgoals for low level controller
- exit condition for subgoal

  - vlm does it
  - proximty to gripper? force sensor?

- camera see into scene
- foundationpose detects objects, calibrates 3d pose relative to sawyer frame
- vlm input: 3d poses, image, natural language instruction
- vlm output: sequence of low level instructions (generated all at the start - keep it open loop for now)
- low level controller input: 3d poses, summary of the scene from vlm, low level instruction from vlm
- low level controller output: sawyer commands
- sawyer commands turn into action
- vlm evaluates whether the low level instruction was successful

## sources

## learned

`rospy.wait_for_message`: blocks the thread until one message is published from the chosen topic, and returns the message

## log

### evaluation pipeline

prompt

```
Move the banana to the center of the table.

Rules:
- Approach the banana from a safe height above it (+0.2m z)
- When descending to the banana to pick it up, descend all the way to it's minimum z value to ensure you grasp it
```

added `ground_truth.py` that controls the trials. in a loop:

- set the banana to a uniformly distributed random location on the table
- start the trial by publishing `/controller/start` which `llm.py` will listen to to begin inference, which subsequently triggers the actions in `controller_node.py`
- `controller_node.py` will publish `/controller/status` when the actions have finished
- calculate distance from banana to the center of the table
- based on distance threshold, publish `/controller/ground_truth` as true or false

`evaluate_bag.py`:

- similar to the vlm evaluate bag
- creates results/trials.csv and results/summary.csv with accuracy and logs of prompts, responses, latency

### testing

eventually got to 0.897 accuracy after some more precise wording in the task and the examples

### 2026-07-16 14:56:20 - TABLE SQUEAKING FIXED BY LIFTING UP AND PUTTING DOWN

### mug push task

#### example setup

calculation example:

[](https://www.desmos.com/calculator/i7c91mogwv)

we place the end effector to be colinear with between the origin and the apple

we tell it to generate key waypoints before determining the plan.

```
Push the white mug to the center of the table.

First determine the key waypoints:
1. Push direction:
   - Compute the direction from the mug center to the table center.
   - Normalize this direction.

2. Far-side point:
   - Place the end-effector on the opposite side of the mug from the table center.
   - Compute:
     far_side = mug_center - push_direction * offset
   - Choose an offset large enough to place the end-effector outside the mug's bounding box.

3. Heights:
   - Safe height = mug zmax + 0.2 m.
   - Push height = mug center z.
   - If the current end-effector z is already above the safe height, keep its current z.

Then, generate a step by step plan according to these rules:
- Use only move actions. Never use open_gripper or close_gripper.
- Move coordinates are absolute target positions.
- Safe height is mug center z + 0.2 m. A higher current z is already safe. Do not add a redundant move when the end-effector is already at a safe height.
- The point behind the mug must lie on the line extending away from the table center.
- Stop the push slightly before the table center so the end-effector does not push the mug past it.
- Do not discuss alternative interpretations, contradictions, calculations, or hidden reasoning.
```

the example follows this as well

TODO: i started with apple initially but switched to cup, the dimensions in example still use apple dimensions i think, so should probably fix later
