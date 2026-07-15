# 260713

## tasks

## sources

1. [ros1 vision_msgs api](https://docs.ros.org/en/noetic/api/vision_msgs/html/index-msg.html)
2. [yolo results api](https://docs.ultralytics.com/reference/engine/results)
3. [moveit repo](https://github.com/RethinkRobotics/sawyer_moveit)
4. [moveit tutorial](https://web.archive.org/web/20230217230652/https://sdk.rethinkrobotics.com/intera/MoveIt_Tutorial)

## learned

## log

### replace blocks and plate with new items

i added an apple and a bowl. i also have an orange model that works but i took it out because it was giving like 0.60 confidence in the regular scene

### publish detection data

goal: publish bounding box info that we can subscribe to from the vlm node and feed it as text context to the model.

i initially tried hooking up Result[] from source 2 to Detection2DArray from source 1.

but, Detection2DArray does not provide a place to put the name of the object detected. So, instead I made custom DetectionArray and Detection messages

### use detection data in prompt

- updated `instruction.py` to prepend a prompt about all the detections
- updated `evaluate_bag.py` to log separate prompt per inference attempt

i have two options for the pre-prompt, `pixel_detection_context` and `normalized_detection_context`. pixel gives pixel coordinates, normalized gives coordinates normalized to width and height so that it's between \[0,1\]

### test it out on multi direction prompt

i also updated the multi direction prompt to use apple and bowl since we don't have the block anymore

it's very strong when the objects are significantly separated as expected

but since these numbers are so precise it's only looking at the numbers and not considering the overlap aspect. so it will call a "left" a "left, up" or "left, down" depending on coordinates

#### prompt 2

tried to emphasize overlap should be considered more, works a little

#### prompt 3

##### observations

i notice when they are close together in an upper area, the bounding boxes overlap but they clearly do not overlap in 3d space, leading to the model getting it wrong. i feel like very difficult to fix

### big overarching issue with 2d bounding box

since our camera is at an angle, 2d bounding boxes make things appear more toward the back than they actually are. this actually regresses the accuracy of these models, who inherently do have some 3d reasoning, but when we give them the 2d pixels, they just rely on that instead
