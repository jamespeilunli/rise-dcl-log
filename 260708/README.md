# 260708

## tasks

set up the evaluation pipeline

instructions to try from Dr. Li

- spatial reasoning relative to objects
- counting
- action plan (text first)

## sources

1. [rosbag](https://wiki.ros.org/rosbag/Commandline)

## learned

## log

### fixing python lsp

turns out i had a 2 year old version of vscode that python extension wasn't supported anymore. i deleted it and reinstalled.

### ground truth

added a ground truth node that finds the banana translation relative to cafe table, and determines its quadrant based on inequalities.

i publish the table relative location to `/vlm/ground_truth/table_relative_position` and the text representation of the quadrant to `/vlm/ground_truth/expected_output`

### misc

i also spent some time unifying the timestamp of everything in one vlm output process, with image_raw as the place the timestamp came from

### evaluation pipeline contract

- `/vlm/used/instruction` (`vlm/StampedString`) should be outputted at least once inside the log

each of these should be associated with the same inference instance and should have the same timestamp

- `/vlm/used/image_raw` (`sensor_msgs/Image`)
- `/vlm/output` (`vlm/StampedString`)
- `/vlm/raw_output` (`vlm/StampedString`)
- `/vlm/ground_truth/expected_output` (`vlm/StampedString`)

here are the files that may be changed across evaluations/experiments, and why:

- `instruction.py`: determines the prompt to the vlm
- `ground_truth.py`: determines what is considered the correct answer
- `vlm_node.py`: processing of `raw_output`, vlm configuration/hyperparameters, etc.
- `sawyer_pick_and_place_demo.launch`: changing the environment
- `ik_pick_and_place_demo.py`: changing robot behavior

### evaluation script

#### how to run the evaluation pipeline

1. launch pick and place:

   ```bash
   roslaunch sawyer_sim_examples sawyer_pick_and_place_demo.launch
   ```

2. start recording data with rosbag (last two are optional, and last one is specific to the banana quadrant experiment):

   ```bash
   rosbag record -O session.bag /vlm/used/instruction /vlm/used/image_raw /vlm/output /vlm/ground_truth/expected_output /vlm/instruction /vlm/ground_truth/table_relative_position
   ```

3. stop the pnp and rosbag when ready
4. run the evaluation script:

   ```bash
   rosrun vlm evaluate_bag.py session.bag --name optional_name
   ```

#### output

it puts all of this in a timestamped folder at `results/`

`summary.json`:

```json
{
  "metadata": {
    "bag": "session.bag",
    "datetime": "2026-07-08T19:55:22.667266+00:00",
    "git_commit": "d848066264316f76f50561ab34d309da587efb3e",
    "git_dirty": true,
    "git_status": "M src/vlm/CMakeLists.txt\n?? session.bag\n?? src/vlm/scripts/evaluate_bag.py",
    "name": "test",
    "prompt": "\n        Find the quadrant of the table that the banana is in.\n\n        Your final answer is the last line of your response. It must be one of these phrases:\n        - top left\n        - top right\n        - bottom left\n        - bottom right\n        - none\n        ",
    "prompt_hash": "61dfbeede825a876e789db107df644983b6368b4",
    "results_dir": "results/2026-07-08_19-55-22_test"
  },
  "summary": {
    "accuracy": 0.8225806451612904,
    "average_latency": 1.1755967741935491,
    "correct": 51,
    "duration": 150.468,
    "end_time": 160.374,
    "missing_expected": 2,
    "missing_image": 0,
    "missing_output": 1,
    "missing_raw_output": 1,
    "scored_trials": 62,
    "start_time": 9.906,
    "trials": 64
  }
}
```

and a csv of each trial at `trials.csv` that looks like this (excerpt):

```csv
trial,stamp_nsec,stamp,expected,output,raw_output,correct,scored,image_record_time,output_record_time,latency,missing_image,missing_output,missing_raw_output,missing_expected
1,9906000000,9.906,,,,False,False,11.92,,,False,True,True,True
2,11866000000,11.866,top left,bottom left,"Looking at the image:

- The table is a large wooden surface in the foreground.
- There is a blue plate on the **left side** of the table.
- A red cube and a blue cube are near the center-right.
- A gray cup is on the far right.
- A robotic arm is holding a banana. The arm is positioned above the **left side** of the table, near the blue plate.

The banana is being held by the robot arm, which is located over the **left half** of the table. Since the blue plate (and thus the general area where the banana is) is on the left side, and it’s also toward the front (bottom) of the scene relative to the camera view, we can consider the table’s layout as:

- Top left / Top right (further back)
- Bottom left / Bottom right (closer to the viewer)

The banana is near the **blue plate**, which is in the **bottom-left** region of the table from the viewer’s perspective.

Therefore, the banana is in the **bottom left** quadrant.

bottom left",False,True,12.093,14.549,2.4559999999999995,False,False,False,False
```

and the prompt at `prompt.txt`
