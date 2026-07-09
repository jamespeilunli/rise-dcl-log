# 260709

## tasks

use evaluation pipeline from yesterday to experiment with different instructions that were decided yesterday

## sources

## learned

Visual Question Answering (VQA): answers a natural language question about an image

- this is what our banana quadrant prompt is

see summary for what i learned about prompt engineering small VLMs

## log

### analyzing four trials with the banana quadrant prompt from yesterday

1. results/2026-07-08_19-55-22_test: 0.82258 accuracy
2. results/2026-07-08_20-27-03_test1: 0.71698 accuracy
3. results/2026-07-08_20-31-27_test2: 0.82222 accuracy - this was when the banana fell off the table and all expected outputs were none
4. results/2026-07-08_20-51-38_test3: 0.46296 accuracy - this was when the banana was near the center and the answer was very ambiguous

#### case studies

##### observations

test2 confusing the banana with the plate:

```
21,49945000000,49.945,none,bottom left,"Looking at the image, we can divide the table into four quadrants: top left, top right, bottom left, and bottom right.

- The robotic arm and the plate (which appears to be holding or interacting with a banana-like object, though it's actually a plate — but the question says “banana” — perhaps referring to the object on the plate as being banana-shaped? Or maybe a misinterpretation — but let’s assume the question intends to refer to the plate as containing the target object) are located in the **bottom left** part of the table.
- The two colored cubes (blue and red) are near the center-right, closer to the top right quadrant.
- However, since the question specifically asks about the “banana,” and if we interpret that as the object being manipulated by the robot (which is the plate, possibly with a banana on it — though not clearly visible), then that object is in the **bottom left**.

Thus, based on spatial positioning:

The banana (if assuming it’s on the plate being held by the robot) is in the **bottom left** quadrant.

bottom left",False,True,50.114,52.553,2.439,False,False,False,False
```

```
44,95948000000,95.948,none,bottom left,"Looking at the image:

- The table is a 2D surface in front of us.
- On the left side of the table (from the viewer’s perspective), there is a light blue circular object — **a plate** — which likely represents a ""banana"" in this context (perhaps a symbolic or mislabeled object for the task).
- To the right of the plate, there are two blocks: one blue on top and one red below.

Since the plate (interpreted as “the banana”) is located on the **left side** of the table, and it is positioned in the **lower half** relative to the blocks (which are more toward the center-top), we can divide the table into four quadrants:

- Top left: upper-left corner
- Top right: upper-right corner
- Bottom left: lower-left corner
- Bottom right: lower-right corner

The plate is clearly in the **bottom left** area of the table.

Therefore, the quadrant that contains the “banana” (plate) is:

bottom left",False,True,96.112,98.432,2.3200000000000074,False,False,False,False
```

test2 putting bold formatting on the final answer:

```
12,31919000000,31.919,none,**none**,"Looking at the image:

- The robotic arm and objects are arranged on a wooden table surface.
- The blue plate is in the lower-left portion of the table relative to the view.
- The red and blue cubes are toward the center-right.
- The banana is not visible in the image — it’s completely absent.

Therefore, since there is no banana in the entire scene, we cannot assign it to any quadrant.

**none**",False,True,32.112,33.165,1.0529999999999973,False,False,False,False
```

##### areas to improve

- explicitly define banana
- formatting of final answer

### trying new prompts for banana quadrant

#### prompt 1

```
        Find the quadrant of the table that the banana is in, relative to the center of the table from the viewer's perspective.

        Your final answer is the last line of your response. It must be one of these unformatted, exact phrases:
        - top left
        - top right
        - bottom left
        - bottom right
        - none
```

##### observations

0.787 accuracy

looks like it fixed the formatting issue. but "relative to the center of the table from the viewer's perspective" is still imprecise

also, another issue: sometimes it treats center as "none" which we do not want

```
[...]
In the image, the banana is to the right of the vertical midline of the table? Actually, looking closely, the robot arm is slightly left of center, and the banana is held under its tip — so the banana is also slightly left of center? Or is it?

Actually, no — the robot arm is angled, and the gripper is positioned near the center, and the banana is hanging down — so it’s very close to the center.

Given that, and since “center” is not a quadrant, and the question says “find the quadrant... it must be one of the unformatted phrases”, and since it’s not in any of the four, I think the correct answer is:

none

But let me double-check with common interpretation: In such visual reasoning tasks, if an object is near the center, it’s often classified as “none” for quadrant-based classification.

Yes.

Final Answer:
none
```

##### areas to improve

- define center better

#### prompt 3

```
Locate the yellow banana.

Determine the true physical center of the table (the geometric midpoint of the table's surface).

Find the quadrant of the table that the banana is positioned over, relative to this true center. "Top" refers to the far side of the table, and "bottom" refers to the near side.

"none" means the banana does not appear in view or is outside the bounds of the table.

Your final answer is the last line of your response. It must be one of these unformatted, exact phrases:
- top left
- top right
- bottom left
- bottom right
- none
```

##### observations

test 1 - with the banana off the table, we have 1.0 accuracy across 41 trials!

test 2 - 0.813 accuracy
test 2 - here's an issue i found, it says the wrong answer even though it gets it right

```
Looking at the image:

- The table is a rectangular wooden surface, viewed from an elevated angle.
- The geometric center of the table would be approximately in the middle of its visible surface — roughly where the red cube and blue cube are positioned, but slightly more toward the center of the entire table area.
- The yellow banana is being held by a robotic gripper. It is located near the top-left corner of the table’s surface (relative to the viewer), just above the light blue plate and to the left of the red and blue cubes.
- “Top” refers to the far side of the table (away from the viewer), and “bottom” refers to the near side (closer to the viewer).

Since the banana is on the far side (top half) of the table and on the left side of the table’s width, it lies in the **top left** quadrant relative to the table’s true center.

none
```

##### areas to improve

- somehow prevent this issue of just saying the wrong final answer...

### order prompt

#### prompt 1

```
Identify the tabletop objects and determine their relative order from left to right, based on their center.

These are all the possible objects that may appear. Use these exact names:
- red block
- blue block
- banana
- white mug
- plate

If any of the possible objects don't appear on the table, do not include them in the list.
If none of the objects appear on the table, output "none".

Your final answer is the last line of your response. It must be only a comma-separated list of the visible objects from left to right.

Example format:
banana,plate,blue block,red block,white mug
```

##### observations

test 1: works well when there are no objects - 1.0 accuracy across 12 trials

test 2: model was formatting with no spaces after commas when the script was expecting spaces, leading to 0 accuracy

test 2: the mug doesn't appear white so the model often ignores it

```
The grey cup is not ""white"", so it is excluded based on the strict instruction ""If any of the possible objects don't appear... do not include them"".
```

test 2: sometimes the model takes a bit longer than last time

##### areas to improve

- change the example format to have spaces after commas
- call the white mug just "mug"
- lower frequency of inference

#### prompt 2

##### what changed

implemented all areas to improve above. frequency is now 0.2hz

```
Identify the tabletop objects and determine their relative order from left to right, based on their center.

These are all the possible objects that may appear. Use these exact names:
- red block
- blue block
- banana
- mug
- plate

If any of the possible objects don't appear on the table, do not include them in the list.
If none of the objects appear on the table, output "none".

Your final answer is the last line of your response. It must be only a comma-separated list of the visible objects from left to right.

Example response:
banana, plate, blue block, red block, mug
```

##### observations

test 1: works pretty well (0.7 accuracy across 10 trials). main failure points are very close objects (like banana and plate start at identical centers i remember)

test 1: the last one is interesting, it figures it out but final answer is just wrong:

```
Looking at the image:

- On the left side of the table, there is a **plate**.
- In the center-right area, there is a small **blue block**.
- To the right of the blue block, there is a **mug** (gray cup-like object).

There is no red block or banana visible.

Ordering from left to right based on their centers:
1. Plate
2. Blue block
3. Mug

banana, plate, blue block, red block, mug
```

##### areas to improve

- make the environment not have exactly aligned objects
- word the example prompts better (maybe multiple example prompts) to prevent final answer hallucinations

#### prompt 3

##### what changed

the two areas of improvement mentioned above

```
Identify the tabletop objects and determine their relative order from left to right, based on their center.

These are all the possible objects that may appear. Use these exact names:
- red block
- blue block
- banana
- mug
- plate

If any of the possible objects don't appear on the table, do not include them in the list.
If none of the objects appear on the table, output "none".

Your final answer is the last line of your response. It must be only a comma-separated list of the visible objects from left to right.

Example responses:

Input: an image containing the following objects in left to right order:
1. banana
2. plate
3. blue block
4. red block
5. mug

Output:
banana, plate, blue block, red block, mug


Input: an image containing the following objects in left to right order:
1. plate
2. mug
3. red block
(banana and blue block are not visible)

Output:
plate, mug, red block


Input: an image containing just the plate
(other possible objects are not visible)

Output:
plate


Input: an image containing no objects, just the table and the robot arm

Output:
none
```

##### observations

test 1: got 0.333 accuracy because it was considering items on the ground as valid items

##### areas to improve

make examples such that it doesn't consider ground items as valid

#### prompt 4

##### what changed

updated these examples
changed the environment a little to make blue block not overlapping

```
Input: an image containing the following objects in left to right order:
1. plate
2. mug
3. red block
(The banana is not visible, and the blue block is on the ground, outside of the bounds of the table)

Output:
plate, mug, red block


Input: an image containing just the plate
(other possible objects are not visible or on the ground)

Output:
plate
```

##### observations

test 1: we got an accuracy of 0.905 across 45 trials. i tried to move the red block and mug on the ground in a place where it would be visible to the camera. generally, it was pretty good at noticing it was on the ground.

however, when the red block was very close to the edge of the table it thought it was on the table.

```
9,54969000000,54.969,"banana, blue block, plate","red block, banana, blue block, plate","Based on the image provided:

1.  There is a small **red block** positioned near the bottom edge of the table (left side) at the lowest vertical position in the scene.
2.  The robot arm is grasping a **banana**, which is located to the right and slightly further back than the block.
3.  Further to the right, on the same horizontal level as the banana, is a **blue block**.
4.  To the right of the blue block is a **plate**.
5.  A **mug** is not visible on the table.

Reading from the leftmost object to the rightmost object based on their center points:
- Red block
- Banana
- Blue block
- Plate
```

even when i moved it away, sometimes it would consider it on the table

```
Based on the image provided, here are the objects on the table from left to right:

1.  Banana
2.  Red block
3.  Blue block
4.  Plate

The red block, although small, is positioned on the table surface roughly below the robotic arm's base, which is to the left of the blue block, banana, and plate in the overall scene composition (or specifically, looking at the TableSurface tags, the red block is the leftmost item on the table surface).

banana, red block, blue block, plate
```

also, rarely it still takes a very long time and outputs a lot of text. seems like a limitation in the model to me

##### areas to improve

maybe define the table better (brown wooden table) to let it understand the bounds more

#### prompt 5

##### what changed

specified on the table more

```
Identify the tabletop objects and determine their relative order from left to right, based on their center.

These are all the possible objects that may appear. Use these exact names:
- red block
- blue block
- banana
- mug
- plate

If any of the possible objects don't appear on the brown wooden table, do not include them in the list.
If none of the objects appear on the table, output "none".

Your final answer is the last line of your response. It must be only a comma-separated list of the visible objects on the table from left to right.
```

##### observations

##### areas to improve

the model doesn't reason about 3d objects very much. we have the far mug to the right of the plate, but if you look at the image from a 2d perspective the plate is clearly to the left

#### prompt 6

##### what changed

got qwen to generate a prompt that focuses on 3d reasoning

```
You are analyzing a 3D scene of a tabletop. The camera is positioned at an angle, so objects that appear "higher" in the image are actually further away on the table.

**Task:** Identify the objects on the brown wooden table and determine their order from left to right **as if you were looking down at the table from directly above**.

**How to reason about left-to-right:**
1. First, identify the table's orientation. The table extends from the foreground (bottom of image) to the background (top of image).
2. "Left" and "right" refer to the left and right sides of the table surface itself, NOT the left and right of the image.
3. To determine the correct order, mentally rotate your viewpoint to look straight down at the table. An object's horizontal position on the table determines its left-to-right order, regardless of how far forward or back it is on the table.
4. Two objects at different depths (one closer, one farther) can still have a clear left-to-right relationship based on their position across the width of the table.

**Possible objects:**
- red block
- blue block
- banana
- mug
- plate

**Rules:**
- Only include objects that are ON the brown wooden table.
- Objects on the ground/floor (outside the table bounds) should NOT be included.
- If no objects are on the table, output "none".

**Your response format:**
First, briefly explain your reasoning about the 3D layout and the left-to-right order.
Then, on the last line, output ONLY a comma-separated list of the visible objects on the table from left to right.

Example:
Reasoning: The banana is on the left side of the table. The plate is on the right side, closer to the camera. The mug is on the right side, further from the camera. Looking from above, the order left-to-right is banana, then plate, then mug.
banana, plate, mug
```

##### observations

in test 1 it was good (0.871 accuracy), but in the more varied test 2 the model hallucinated a lot

i'm guessing the prompt made it good at 3d reasoning but also made it overthink, and this is reflected in the average latency of 2.8 compared to typical 1-1.5 accuracy was 0.1515 because of these very long hallucination runs

##### areas to improve

need less emphasis on that 3d reasoning. we are overprompting

#### prompt 7

##### what changed

```
Identify the tabletop objects and determine their relative order from left to right across the **width of the table**.

Important: Ignore depth (distance from the camera). "Left" and "right" refer strictly to the table's left and right edges, not the 2D image perspective.

These are all the possible objects that may appear. Use these exact names:
- red block
- blue block
- banana
- mug
- plate

If any of the possible objects don't appear on the brown wooden table, do not include them in the list.
If none of the objects appear on the table, output "none".

Your final answer is the last line of your response. It must be only a comma-separated list of the visible objects on the table from left to right.

Example responses:

Input: an image containing the following objects in left to right order:
1. banana
2. plate
3. blue block
4. red block
5. mug

Output:
banana, plate, blue block, red block, mug


Input: an image containing the following objects in left to right order:
1. plate
2. mug
3. red block
(The banana is not visible, and the blue block is on the ground, outside of the bounds of the table)

Output:
plate, mug, red block


Input: an image containing just the plate
(other possible objects are not visible or on the ground)

Output:
plate


Input: an image containing no objects, just the table and the robot arm

Output:
none
```

##### observations

we are back to more normal responses, but the 3d reasoning issue continues to be a major issue. i will stop trying to continue fixing it through prompts as it seems like a limitation of VLM in general

### counting

#### prompt 1

```
Look at the image and answer this question:
How many bananas are on the table?

Count only bananas that are on the tabletop.
Ignore the robot arm, the table, and all non-banana objects.

Answer on the last line with only one number.
```

##### observations

it thinks that in the robot arm doesn't count as on the table

##### areas to improve

make it count in the arm as on the table

#### prompt 2

```
Look at the image and answer this question:
How many bananas are on the table or held by the robot arm?

Count only bananas that are:
- directly on the tabletop
- held in the robot arm or gripper above the table.

Do NOT count bananas on the floor or anywhere else off the table.
Ignore all non-banana objects.

Answer on the last line with only one number.
```

##### observations

the criteria makes it overthink and over/undercount

```
Let's count the bananas on the table:

- One banana is being held by the robot arm — but the instruction says to **ignore the robot arm**. So, this banana is not counted.
- One banana is lying on the table near the blue cube.
- Another banana is lying on the table in the foreground, near the bottom right.

So, only the two bananas that are clearly resting on the tabletop (not interacted with by the robot) are to be counted.

Total = 2

Answer:
2
```

##### areas to improve

make prompt simpler

#### prompt 3

##### observations

0.6286 accuracy

issues:

```
Answer: 2
```

instead of just "2"

still sometimes counts the banana in the gripper as not part of the count

#### prompt 6

##### what changed

- explicit prompt saying don't include "Answer" in the line
- say "on top or above" in regards to what counts as in bounds
- brown wooden table
- added example to also prevent saying "Answer" in the line and guide latency a bit

```
Analyze the image and determine the amount of bananas on top of or above the table.

Rules:
- Count only bananas that are within bounds of the brown wooden table. Within bounds means resting on top of or above.
- Do NOT count bananas on the floor. They are out of bounds.
- Ignore all non-banana objects.

Answer on the last line with only one natural number, with no extra words, synbols, or formatting.
For example, do not say "Answer: 2", just say "2".

Example input: An image with a banana in the top left corner of the table, a banana in the top right corner of the table.
Example output:

Counting the bananas:
- There is one banana in the top left corner.
- There is one banana in the top right corner.
- There is one banana on the floor, but it is out of bounds and not counted.

Final answer:
2
```

##### observation

sometimes on edge it still struggles with defining whether its on the table or not, but mostly very strong

0.864 accuracy

#### prompt 7

##### what changed

i made it only output its final answer without thoughts before it

```
Analyze the image and determine the amount of bananas on top of or above the table.

Rules:
- Count only bananas that are within bounds of the brown wooden table. Within bounds means resting on top of or above.
- Do NOT count bananas on the floor. They are out of bounds.
- Ignore all non-banana objects.

Output only one natural number, with no extra words, synbols, or formatting.
For example, do not say "Answer: 2", just say "2".

Example input: An image with a banana in the top left corner of the table, a banana in the top right corner of the table.
Example output:

2
```

0.971 accuracy across 34 trials :mind-blown:

### direction

#### prompt 1

```
Look at the red block and the blue block on the table.

Question: Which direction should the blue block move to get closer to the red block?

Choices:
- left
- right
- up
- down

Use image directions. Compare the centers of the two blocks. Choose the direction with the greatest gap between the two objects
Ignore all other objects.

The last line is your final answer. It should be exactly one choice word and nothing else.
```

##### observations

0.571 accuracy

```
""Greatest gap"" — but gap in what? The phrasing is poor.
```

##### areas to improve

I think we are over prompting again

#### prompt 2

##### what changed

more clear determining horizontal vs vertical

```
Analyze the image and determine which direction the blue block should be moved to be closer to the red block.

If the blocks are separated both horizontally and vertically, move the blue block along the axis with the largest gap.

Choices:
- left
- right
- up
- down

The last line is your final answer. It should be exactly one choice word without formatting or extra text.
```

##### observations

usually when the response is a short one line one its just completely wrong

```
49,163224000000,163.224,down,right,"The blue block is positioned to the left of the red block. To move the blue block closer to the red block, it should be moved to the right.

right",False,True,163.46,163.972,0.5120000000000005,False,False,False,False
```

### summary

1. subtle changes in prompts heavily impact the performance of small VLMs
2. overemphasis on fixing/defining a particular aspect can lead to hallucinations in other aspects
3. so it is very important to word everything you want to cover in a clear and concise way
4. for simple tasks like counting, it may be better to only output the answer, without reasoning before it
5. it is difficult to force the model to output its final answer in a particular format always
6. examples are very useful for alignment
7. VLMs can identify distinct objects in an image and VERY roughly where they are. they still struggle at more complex spatial reasoning, especially inferring 3d information from a 2d image
