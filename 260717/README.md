# 260717

## tasks

- get the vlm to output sequence of low level language instructions
- try pick and place task
- hook it up to the llm
- add vlm loop with memory
- add success measurement
- scene summaries

## sources

## learned

## log

### moving 3d bbox and center shared context to node in yolo/

publishing `ObjectContext` of bbox and center

`ObjectContext.msg`:

```
std_msgs/Header header
string object_centers
string object_bboxes
```

`src/vlm/scripts/instruction.py` also uses this same string context now

### vlm setup

initially copying from the hi-vla paper (source 1 appendix E)

```
Image Input: [Current Camera Observations]
Text Input: You are a decision-making agent tasked with generating natural language command that is
sent to a Robotics Vision-Language-Action (VLA) model for a two-armed robot towards completing
the given task [Task Instruction]. The command must be in the active, second-person voice (addressing
the robot), based on the current observation of the robot shown in the image above. [Observation
Representation].
Key Directives:
1. Output a single command that should be executed immediately. The command should facilitate
completion of the given task.
2. The command should be doable within 10 seconds. Consider the affordance of the VLA based on the
history steps as well as the current state of the robot.
3. Think step by step internally to arrive at the command. Do not output your thought process.
Current Memory: [Memory].
VLM Policy Output: [Language Command]
```

this example works pretty well

```
Output:
1.  Identify Objects: The scene contains a cafe_table, an apple, a banana, a white_mug, and a bowl. The task specifically targets the "fruits", which are the apple and the banana. The table is the target location.
2.  Analyze Locations:
    *   Apple: Located at x=0.651, y=-0.353. This is on the left side of the table (negative y).
    *   Banana: Located at x=0.328, y=0.263. This is on the right side of the table (positive y).
    *   Table Center: The table center is approximately x=0.750, y=0.000.
3.  Determine Action Plan: Both fruits are currently off-center. To complete the task, both need to be moved. The prompt asks for a list of commands describing actions. A logical sequence is to address each fruit individually to ensure clarity and completion.
4.  Formulate Commands:
    *   Command 1: Address the apple. "Move the apple to the center of the table."
    *   Command 2: Address the banana. "Move the banana to the center of the table."
```

### making the llm more generic

just added both push and pnp to a constant "rules" prompt, and both push and pnp examples

### hook it up to the llm

- we have a callback listening to `/vlm/output` in `src/controller/scripts/instruction.py`
- in the callback, we have a loop that goes through the low level language instructions
- it publishes `/controller/start` and blocks until `/controller/stop` (what used to be `/controller/status`) is published
- once the loop finishes, it's done. i don't publish anything that triggers a restart, i just have `vlm_node` only run once right now

### vlm in a loop

made some progress, check git stash
