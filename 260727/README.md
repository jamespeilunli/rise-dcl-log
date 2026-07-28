# 260727

## tasks

- define several complex (spatially and temporally) tasks for the system to do and potentially fail at
- implement the tasks into the existing hi vlm setup
- devise an evaluation pipeline to measure accuracy at completing the tasks
- implement specification defined system
- test the spec system on the complex tasks with evaluation pipeline

what is this spec system?

- write the correct spec
- convert to automata
- get the right sequence of subgoals

see [https://github.com/BU-DEPEND-Lab/GenZ-LTL/tree/main](), `src/evaluation/eval_test_tasks_finite.py`, `src/sequence/search/exhaustive_search.py`, `src/ltl/automata/ldba.py`

misc

- improve gripping? since stacking tasks not very good...

## log

### task brainstorm

temporal orders of pnp (attempted, hi vlm worked)

cube spatial order at center (attempted, hi vlm wasn't close to working)

cube should not collide when moving

stacking/unstacking cubes (attempted, 2-stack works somewhat, but after that vlm doesn't really say place it on top so llm just makes them collide)

### misc changes

- tried to improve friction of grippers in sim, but didn't seem to improve anything
- messed with the coordinate system prompts for vlm and llm, since it's confusing because #1, +y is left when the model by default thinks it's right, and #2, the camera is 180 deg from the robot so x and y are basically flipped. TODO figure this out
- instructed llm to lift after placing down
