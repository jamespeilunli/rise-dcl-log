# 260714

## tasks

## sources

## learned

## log

### attempting moveit

issues:

- the paths it solves for are very bizzare sometime, makes the arm move a lot
- tried to add the table surface as collision model, fixed this a little but it still like brushes against the table
- gravity makes arm start resting on table, so i had to make it paused as everything started up so i would start in a reasonable location
- very sensitive to colliding. like if theres a cup in the way it makes right_j5 out of tolerance and crashes
- still not very precise

### improving existing ik_request

#### try: loop ik_request while error > tolerance

#### try: interpolated steps

#### try: motion planning

### low level controller

#### outline

input:

- simple language task ("push the banana to the bowl")
- pose of relevant objects

output:

- sequence of poses

#### impl

in controller/

just copied a lot of vlm/ node stuff, just without the vision part

not planning on validation for now

#### 3d context

add a `sim_pose_context` function to `src/controller/scripts/instruction.py` that gives the names and 3d poses of every object published by `src/object_tf/scripts/gazebo_object_tf.py`

basically, this lets us assume that our perception is perfect in simulation, without having to figure out perception yet

#### testing 3d context prompt on multidirection task

looks pretty good

```
To determine the direction the apple should move to get closer to the bowl, let's analyze the positions provided:

1.  **Apple Position**: (0.631, -0.009, 0.776)
    *   X: 0.631
    *   Y: -0.009
    *   Z: 0.776

2.  **Bowl Position**: (1.050, 0.240, 0.775)
    *   X: 1.050
    *   Y: 0.240
    *   Z: 0.775

**Analysis**:
*   **Horizontal Movement (X-axis)**: The apple is at X = 0.631 and the bowl is at X = 1.050. To increase the X coordinate (move from left to right in this coordinate system), the apple needs to move **right**.
*   **Vertical Movement (Y-axis)**: The apple is at Y = -0.009 and the bowl is at Y = 0.240. To increase the Y coordinate (move from down to up in this coordinate system), the apple needs to move **up**.

Since the instruction requires listing horizontal movement before vertical movement, the directions are "right" and "up".

right, up
```

#### adding gripper pose context

we subscribe to `/robot/limb/right/endpoint_state`

#### crazy bug

see [this commit](https://github.com/BU-DEPEND-Lab/RISE-2026/commit/5c3518e35569a8522c53a1716d3b949e60dd1f37)

the sawyer gazebo cpp code had a dangling reference, making the published endpoint_state invalid. this was also what was causing our arm moving so weirdly

#### getting it to output coordinates

```
Generate 3-8 ordered end-effector positions that complete the task. Keep the tool orientation fixed.

Return only valid JSON in this format:

[
{"x": 0.0, "y": 0.0, "z": 0.0}
]
```

#### make pose in sawyer frame

```python
def pose_in_sawyer_frame(pose, source_frame, tf_listener):
    stamped_pose = PoseStamped()
    stamped_pose.header.frame_id = source_frame
    stamped_pose.header.stamp = rospy.Time(0)
    stamped_pose.pose = pose
    return tf_listener.transformPose(SAWYER_FRAME, stamped_pose).pose
```

i just call this on the output of the model and let the model do everything in the world frame
