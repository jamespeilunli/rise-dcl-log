# 260710

## tasks

## sources

1. [yolo object detection](https://docs.ultralytics.com/tasks/detect#models)
2. [yolo install](https://docs.ultralytics.com/quickstart#use-ultralytics-with-python)
3. [coco dataset](https://cocodataset.org/#explore)
4. [gazebo objects](https://app.gazebosim.org/dashboard)
5. [urdf objects](https://github.com/yding25/URDF_models)

## learned

### how to access a file from ros

`CMakeLists.txt`:

```
install(
  DIRECTORY assets/
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/assets
)
```

```python
package_path = rospkg.RosPack().get_path("package_name")
image_path = os.path.join(package_path, "assets", "leclerc.jpg")
```

only put assets in the path if you installed that directory in CMakeLists.txt

## log

### direction

#### prompt 3

##### what changed

```
Analyze the image and determine which direction the blue block should be moved to be closer to the red block.

"Up" refers to moving the block closer to the far side of the table, and "down" refers to moving the block closer to the near side of the table.

If the blocks are separated both horizontally and vertically, move the blue block along the axis with the largest gap.

Choices:
- left
- right
- up
- down

The last line is your final answer. It should be exactly one choice word without formatting or extra text.

Example input: An image with a blue block below and slightly to the left of a red block.
Example output:

Horizontally: The blue block is slightly to the left of the red block.
Vertically: The blue block is significantly below the red block.

According to the instructions, the block should be moved along the axis with the largest gap.
Because the vertical distance is greater than the horizontal distance, the blue block should be moved vertically.
Because the blue block is below the red block, it must be moved up to be closer to the red block.

Final answer:
up
```

##### observation

sometimes it can't determine the direction it should go when there are both horizontal and vertical gaps

0.714 accuracy

#### prompt 5

##### what changed

back to simplicity, but still have the definition of up/down.

also in test 2 i put temperature to 0.0 and top_p to 0.8

```
Analyze the image and determine which direction the blue block should be moved to be closer to the red block.

"Up" refers to moving the block closer to the far side of the table, and "down" refers to moving the block closer to the near side of the table.

If the blocks are separated both horizontally and vertically, move the blue block along the axis with the largest gap.

Choices:
- left
- right
- up
- down

The last line is your final answer. It should be exactly one choice word without formatting or extra text.
```

##### observation

test 1

0.884 accuracy

test 2

0.971 accuracy

### multi direction prompt

#### prompt 3

##### what changed

```
Analyze the image and determine which direction or directions the blue block should move to get closer to the red block.

"Up" refers to moving the block closer to the far side of the table, and "down" refers to moving the block closer to the near side of the table.

Choices:
- left
- right
- up
- down

Rules:
- Compare the centers of the red block and blue block.
- The blocks must be significantly far away from each other in a given axis to warrant movement in that axis
  - Only include a horizontal movement if none of the horizontal coordinates of the pixels on the blue block overlap with the horizontal coordinates of the pixels on the red block.
  - Likewise, only include a vertical movement if none of the vertical coordinates of the pixels on the blue block overlap with the vertical coordinates of the pixels on the red block.
- Always list horizontal movement before vertical movement. For example, "left, down" instead of "down, left".
- Ignore all other objects.

The last line is your final answer. It should be one or two choice words separated by a comma and a space, with no formatting or extra text.
Example final answers:
right
right, up
```

##### observation

good response

```
Looking at the image:

- The **blue block** is located toward the upper-left area of the table.
- The **red block** is located toward the lower-center area of the table.

We are to determine which direction(s) the blue block should move to get closer to the red block.

### Step 1: Compare horizontal positions (left/right)

- Blue block’s center is more to the **left** side of the table.
- Red block’s center is more toward the **center** (slightly right of blue).
- So, horizontally, blue is to the left of red → moving **right** would bring them closer.

Check for overlap in x-coordinates:
The blue block is clearly to the left of the red block — no horizontal overlap. So horizontal movement is warranted.

### Step 2: Compare vertical positions (up/down)

- “Up” = closer to far side of table (top of image)
- “Down” = closer to near side of table (bottom of image)

Blue block is higher up (closer to top/far side) than red block, which is lower down (closer to bottom/near side).

So, vertically, blue is above red → moving **down** would bring them closer.

Check for overlap in y-coordinates:
Blue block is clearly above red block — no vertical overlap. So vertical movement is warranted.

### Step 3: Apply rules

- List horizontal movement before vertical.
- Both movements are warranted.

Final answer:
right, down

right, down
```

bad response

```
Looking at the image:

- The **blue block** is on the left side of the table.
- The **red block** is more toward the center-right, but still relatively close to the blue block horizontally.
- Specifically, if we consider horizontal coordinates (left-right), the blue block’s pixels do **overlap** with some of the red block’s pixels — they are not significantly separated horizontally. So no horizontal movement is warranted.
- Vertically (up/down): The blue block is closer to the far side of the table (higher in the image), while the red block is closer to the near side (lower in the image). Therefore, to get closer vertically, the blue block should move **down** (toward the near side).

Since only vertical movement is needed, and we must list horizontal before vertical (but there is none), we only output the vertical direction.

down
```

0.5 accuracy

##### areas to improve

i think that the steps are helping it out. i think we should make it think step by step in a logical manner, simple task by simple task

#### prompt 4

##### what changed

i have this step by step outline so that the vlm can determine things in a logical sequence

```
Analyze the image to determine which direction(s) the blue block should move to get closer to the red block.

Definitions:
- "Up" = toward the far side of the table (top of image).
- "Down" = toward the near side of the table (bottom of image).

Choices:
- left
- right
- up
- down

Think step-by-step:

Step 1: General location
- Determine the general location of the blue block on the table: upper left, upper middle, upper right, etc.
- Determine the general location of the red block on the table.

Step 2: Horizontal check
Compare the left and right positions of the blue and red blocks.
- Are their horizontal positions clearly separated, or do they overlap?
- If separated, which way must the blue block move?
- Horizontal direction: [left / right / none]

Step 3: Vertical check
Compare the top and bottom positions of the blue and red blocks.
- Are their vertical positions clearly separated, or do they overlap?
- If separated, which way must the blue block move?
- Vertical direction: [up / down / none]

Step 4: Final Answer
- Combine the directions from Step 1 and Step 2.
- Always list horizontal movement before vertical movement (e.g., "right, down").
- Ignore all other objects.

The very last line must be the final answer. It should be one or two choice words separated by a comma and a space, with no extra text.
Example final answers:
right
right, down
```

the most important part is the step 1 which asks it to find the general location before checking their relative locations. this prevents a lot of hallucinations in the comparison step since it already looked at the general location in the first step

##### observation

0.829 accuracy

what's interesting is most of the struggle points is in this specific area on the upper left

```

Step 1: General location
- The blue block is located in the upper-left area of the table.
- The red block is located in the upper-left area of the table, to the left of the blue block.

Step 2: Horizontal check
- The red block is to the left of the blue block.
- To get closer, the blue block must move to the right.
- Horizontal direction: right

Step 3: Vertical check
- The red block is slightly higher (further back) than the blue block, but they are very close vertically.
- Moving the blue block "up" would bring it closer to the red block's vertical position.
- Vertical direction: up

Step 4: Final Answer
- Combine the directions: right, up

right, up
```

##### areas to improve

maybe we can change the order of the wording to prevent this confusion

also, sometimes it just outright fails to understand where it is and says

> The red block is located in the upper-left area of the table, to the **right** of the blue block.

maybe because it's so small there?

### yolo object detection

the goal: run a yolo model on a ros node that takes in the image feed from the head camera and outputs bounding boxes and labels for each object in the scene

#### install (source 1)

```
root@depend:~/ros_ws# pip install -U ultralytics
[...]
error: uninstall-distutils-installed-package

× Cannot uninstall psutil 5.5.1
╰─> It is a distutils installed project and thus we cannot accurately determine which files belong to it which would lead to only a partial uninstall.
```

```
pip install -U ultralytics --ignore-installed psutil
```

#### pretrained model object detection

excerpt of running inference and outputting annotated node to a topic

```python
self.model = YOLO("yolo26x.pt")

cv_image = self.bridge.imgmsg_to_cv2(image_msg, "bgr8")

results = self.model.predict(cv_image, verbose=False)
annotated = results[0].plot()

output_msg = self.bridge.cv2_to_imgmsg(
    annotated,
    encoding="bgr8"
)
```

turns out the yolo model is trained on the COCO dataset (source 3), meaning it only recognizes certain objects including banana, cup, and table. however it doesn't recognize the blue and red blocks or plate, so we are replacing them

i checked the COCO dataset for valid items

#### more objects

source 4, source 5

- https://app.gazebosim.org/GoogleResearch/fuel/models/Now_Designs_Bowl_Akita_Black
- https://github.com/yding25/URDF_models/tree/main/food_orange
- https://github.com/yding25/URDF_models/tree/main/food_apple
