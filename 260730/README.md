# 260730

## log

### planning action selection algorithm

goal: make src/ltl_planner/scripts/planner.py an all in one replacement for src/vlm/scripts/vlm_node.py that publishes language subgoals derived from each step from the ltl sequences

- analyze only the next immediate step from each sequence
- only consider reach for now
- compare the current propositions, say red: false and blue: false
- choose the step that makes the least changes to the propositions
- for instance, if there is a step that is (red & blue) vs a step that is just (red), you would choose (red)
- to convert to language, we use the same diff from the current propositions to the requested ones. in this case for (red) the diff would just be (red)
- for another example, we might have red: true and blue: false, and steps options that are () which means red turn to false or (blue) which means turn blue to true.
- both are one step, however lets defer support for "negative" propositions (ie turning true to false) turning into actions for now, we may potentially want to support
- it in the future, but if there are no steps without negative propositions, raise an error.
- we assume that every proposition involves one object and one location where that object could be or could not be.
- therefore we can easily just hardcode a map of every proposition to an action, saying "move object x to location y"
- in the future we may want to support MULTIPLE actions. however for now, if the diff includes multiple changed propositions, raise an error
- we need to do this in a loop obviously. in the actual setup we would have a way to observe all the propositions. pretend we have a function that does that, but for now just hardcode it. if the propositions are what we expect (action successful), run the search again from the possible states where we are at and repeat. if not, raise an error.

### hook up planner to ros system

goal:

- hook up planner.py to the rest of the system, replacing the vlm node.
- basically i want to be able to configure two things, each in one line:
  - whether we are running an evaluation or not (ie whether its just one run of ltl planner or vlm, or the multiple trials with resetting)
  - whether we are running the ltl planner or the vlm.
- the idea is that planner and vlm share an identical interface and basically serve the exact same purpose, so that we can easily compare them by changing flags

also, i:

- added observation.py in the ltl_planner package to observe the propositions
- rename shared /vlm topic interface to /planner

### handle failures more gracefully

goal:

- when the expected propositions are not achieved, the planner shouldn't just crash out, it should try to still plan from the existing state
- in the case of a "crash out" we should fail the trial gracefully, not raise an error

### testing with our first big task

#### independent variables

prompt

```
Place the red block in the center of the table, then place the blue block in the center of the table, then place the green block in the center of the table.
```

ltl formula

```
(!blue_in_center U (red_in_center & !blue_in_center)) & (!green_in_center U (red_in_center & blue_in_center & !green_in_center)) & F (red_in_center & blue_in_center & green_in_center)
```

#### data collection (dependent variables)

automatically launched from launchfile. bags appear in `~/.ros`

```xml
  <node if="$(arg record_evaluation)"
        name="hierarchical_evaluation_recorder"
        pkg="rosbag"
        type="record"
        output="screen"
        args="-o $(arg evaluation_bag_prefix)
              /planner/evaluation/start
              /planner/evaluation/result
              /gazebo/model_states
              /controller/stop
              /planner/output
              /llm/used/instruction
              /llm/raw_output
              /llm/output
              /vlm/used/instruction
              /vlm/raw_output" />
```

#### data analysis

```bash
rosrun vlm evaluate_hierarchical_bag.py ordered_eval.bag
```
