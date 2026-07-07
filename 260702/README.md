# 260702

## sources

1. [hierarchical VLA paper](https://arxiv.org/abs/2606.10267)
2. [sparse reward](https://arxiv.org/html/2407.00324v2)
3. [GROD](https://deepmind.google/models/gemini-robotics/gemini-robotics-on-device/)
4. [CodeGraphVLP](https://arxiv.org/abs/2604.22238)
5. [mark based visual prompting](https://arxiv.org/abs/2403.03174)

## learned

AI concepts:

- sparse reward
- non-markovian problems and importance of memory
- policy
- frontier VLM/VLA robotics basic architecture

## log

### some clarification on project goals - difference between setups

> We will start with a VLM+MoCap/Camera+LLM setup first and move to a VLM+VLA setup if we have the time.

(Will be referencing source 1 a lot.)

Both use a VLM to operate in a high-level loop, outputting small instructions to the low-level loop. They differ in what the low-level loop looks like.

VLM+MoCap/Camera+LLM setup

- Must have code that processes between parts of the system
- Importantly, the MoCap/Camera information must be turned into text-based data (e.g. a pose of a block) to be fed as context into the LLM
- Easier in terms of getting it to work/training
- RL fine-tuning can still be done, but you can't really train the whole system as an end-to-end model

VLM+VLA setup

- End-to-end
- The only inputs to the VLA are the direct language instruction from the VLM, and images of the scene
- Harder because of training - because existing VLA are often tuned for a specific setup. Fine-tuning is required to get it to work.
- Training can be done end-to-end

#### RL

(See source 2)

Very important RL concept: _sparse reward_

> Sparse reward reinforcement learning occurs when an agent only receives feedback for completing a complex task

For complex tasks, it may be nearly impossible to train like this because there could be millions of states receiving 0 reward

_reward shaping_: adds intermediate heuristic metrics to guide agent toward final goal

### source 1 hierarchical VLA (Hi-VLA) literature review

#### 1. intro

- monolithic VLA bad at long or abstract reasoning tasks. struggle to generalize
- fine tuning often compromises reasoning
- hierarchical VLA better because VLM plans and orchestrates and VLA executes specific small task
- hierarchical VLA systems can be improved through:
  - better VLMs (better reasoning, spatial awareness)
  - steerable VLA controllers
  - termination rules
  - memory mechanisms for VLM
  - representations of observations for VLM

#### 3. hierarchical VLA overview

Unified Hi-VLA algorithm pseudocode:

```
Input: task instruction

while task isn't complete {
  // High level VLM loop
  observe and process images/observations from environment
  update memory with past observations, commands, and outcomes
  subtask instruction = VLM(memory, task instruction)

  while subtask hasn't reached termination {
    action = VLA(current time observation, subtask instruction)
    perform the action
  }
}
```

#### 4.2. VLM policy

- they tried Gemini 2.5 Lite, Flash, and Pro

conclusion:

- reasoning = good
- model size doesn't matter too much

#### 4.3. VLA policy

- tried Gemini Robotics On-Device (GROD) model (see source 3)

conclusion:

- model size bigger = more "steerable" = good
- fine tuning can lose steerability, ability to generalize, and struggle in long horizon

#### 4.4. termination conditions

ways to terminate:

- fixed frequency: force a duration for VLA (an _execution horizon_) to run each subtask
- success detection: success detector detects success
- VLM termination: VLM generates expected execution duration

conclusion:

- looks like they tried success detection and fixed frequency
- success detection = good
- fixed frequency = they recommend 4-8 seconds for execution horizon. smaller is better but computation cost is a consideration.

#### 4.5. observation representations

ways to represent observations to feed into the VLM:

- raw image
- naive summarization: image + basic summarization from VLM
- summarize with privileged info: sim information passed when generating text summary (e.g. contact between objects)
- summarize with bounding box: image with VLM generated bounding boxes + text description

conclusion:

- bounding box = good
- privileged = good. obviously you can't get privileged info irl.
  - they frame this as a limitation of VLM models as they SHOULD be able to have all that information in the image but currently lack complex spatial understanding capabilities
  - additional sensors (e.g. force sensor) can achieve similar effect in this regard

#### 4.6. memory

> However, summarizing experiences across episodes can positively impact performance. This suggests that for hierarchical systems, extracting affordances from cross- episode information (especially previous successful episodes) is more beneficial than relying on in-episode failure signals for on-the-fly VLM corrections.

This is also interesting to me:

> Future work could explore more powerful techniques, such as reinforcement learning or supervised finetuning of the VLM, to better leverage cross-episodic interactions with the VLA.

conclusion:

- in-episode memory = no effect
- cross-episode memory = good

#### 5. future

- investigate latency sensitive scenarios (dynamic environment)
- RL or supervised finetuning of VLM to better integrate cross-episodic knowledge -> give better prompts to VLA
- hierarchical architecture can "guide low-level continual policy improvement", TODO see https://arxiv.org/abs/2603.11653

#### setup

- MuJoCo ALOHA suite - tabletop manipulation benchmark
- experiments mainly in simulation
- tasks categorized by short-horizon, long-horizon, and reasoning
- to me it looks like they relied on existing solutions from Google for setup (ALOHA, GROD)

interesting tasks

- "Put the item that monkey can eat into the bowl"
- "Put all the cubes into the green cup"
- irrelevant items on table as distractors

### source 4 CodeGraphVLP

#### vocab

- _Markovian_: assumes current visual observation contains all the information needed to predict the next action
- _Non-Markovian_: provides memory of what happened in the past. useful for complex, long-horizon tasks
- _policy_: a function/model that outputs an action based on inputs like vision/language

#### summary

problem

- lack of "grounded task state" in the memory of VLA systems
- basically they struggle when the correct next action depends on previous events (non-Markovian)
- difficulty in cluttered scenes
- latency in traditional language based VLM setups

solution

- create a graph representation where objects are the nodes and edges are relationships between them
  - instance segmentation model finds objects
  - VLM determines which objects matter, naming them
  - objects in multiple views are associated
  - relationships between objects are determined via code
- LLM is given task instruction, graph schema/API, and initial graph. they write a python function `policy(graph)` that outputs a language subtask for the VLA, given the current graph state
- after that, the code executes the plan, and VLA follows each subtask

#### setup

tasks

- Pick-and-Place Twice: "Move the cube to the other plate, then return it to the original plate"
- Place-and-Stack: "Put the cube into the nearest cup, then stack the other cup on top."
- Swap Cups: "Swap two cups, start with blue cup."

some VLA that they compared against

- pi 0 (used this as the one CodeGraphVLP used)
- pi 0 fast
- pi 0.5
- gr00t n1

software impl

- "fine-tune all VLA models from their public checkpoints for 50K iterations using low-rank adaptation (learning rate 1 × 10^−5, batch size 128) on four NVIDIA A6000 GPUs."
- each VLA queried predicts a short sequence of actions (_action chunk_) of size 10. i.e. execute 10 actions before reobserving scene and going back to policy()
- GPT-5 creates `policy` function

hardware

- UR10e robotic arm with a Robotiq 2F-85 parallel-jaw gripper

camera

- rgb
- images 10 hz
- fixed camera over shoulder
- wrist-mounted camera near gripper

### source 5 mark-based visual prompting

#### summary

- uses _mark-based visual prompting_ to convert language instructions + RGB-D into actions
- VLM doesn't directly output 3d actions
- overlays "affordance marks" over image and asks VLM to select keypoints, waypoint regions, and motion attributes in a structured response

#### setup

some tasks ("open-world" tabletop scenes with multiple relevant and distractor objects)

- table wiping: move glasses into a case; sweep trash with a broom
- watch cleaning: put watch into ultrasonic cleaner; press power button
- gift preparation: put filler in gift box; put perfume in gift box
- laptop packing: unplug cable; close laptop lid

hardware

- 7-DoF Franka Emika robot arm
- 2F-85 Robotiq gripper
- 5 hz

cameras

- RGB-D
- two fixed ZED 2.0 cameras
- one ZED mini wrist
- top-down camera as primary camera enables spatial grounding

> This design highlights an emerging strategy in VLM-based manipulation: simplifying robotic spatial reasoning by converting continuous action generation into discrete visual question answering.

### summary of literature review

- source 1 gives us a strong overview of the basic Hi VLA architecture, an emerging technique that allows complex task planning to be done separately from VLA controllers turning basic language instructions into robot actions
- source 4 changes the means of planning using a semantic graph of object relationships and generating code to execute on a plan instead of relying on a VLM every loop
- source 5 uses "affordance marks" to address spatial awareness problems via labelling key points
- all of them emphasize the importance of maintaining spatial and temporal reasoning over a complex task. VLM/VLA struggle with this, (ie VLMs lack complex spatial awareness of objects from raw images alone). additionally, there is the memory aspect, with non-Markovian tasks requiring awareness of past events and success status. therefore, there are many techniques to essentially engineer context beyond the raw image and current observations for these models
- on the setup perspective:
  - experimental tasks emphasize spatial, temporal, or reasoning/interpretation complexity
  - scenes are usually a flat table with gripper(s), various household objects or shapes. importantly, distractor items not relevant to the task simulate an "open-world" environment
  - cameras are RGB, sometimes depth. usually there is the "main camera" that is fixed and see the entire environment from top-down. sometimes there is camera mounted at gripper
  - prompts are relatively simple from what i see. they are clear, concise, direct, and prioritize getting the context across

### stuff to look into further after literature review

- exact prompt engineering strategies (or lack thereof)
- what the VLA controller outputs - just a pose?
- what makes the VLA work itself. it needs GOOD 3D spatial reasoning
  - extra context?
  - for our Vision + LLM, may be easier, but any systems to make poses make sense, or code/libraries we can use, etc.
- couldn't find any sawyer examples. investigate difficulty or difference of running sawyer
