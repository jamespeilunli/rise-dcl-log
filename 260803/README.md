# 260803

### experimental setup

- require non dirty git before starting an evaluation
- evaluation, planner type, trials, seed are configurable arguments in the launchfile

the command that i run overnight in tmux for any experiment:

```bash
roslaunch sawyer_sim_examples sawyer_pick_and_place_demo.launch trials:=200 use_ltl_planner:=false; roslaunch sawyer_sim_examples sawyer_pick_and_place_demo.launch trials:=200 use_ltl_planner:=true
```

#### misc changes i had to do over the weekend

- handle all trial exceptions gracefully as failures, so that we can move on to next trial
  - `except Exception`s in all `controller/` scripts (i believe vlm scripts already had them)
- make the bag recording in the launchfile `required=true` because it was refusing to record because out of storage and didn't crash the experiment, so i ran it without collecting data overnight :skull:

### experiments

- three block temporal task
- two block temporal task

### analysis

created two analysis scripts not hooked up to ros:

```bash
python src/analysis/analyze_hierarchical_results.py --baseline results/evaluation_vlm_2026-08-02-15-08-46 --method results/evaluation_ltl_2026-08-03-13-58-25 --baseline-label VLM --method-label LTL --match-tolerance 0.02 --output-dir analysis2
```

- meant to give a detailed report comparing the two planners on the same task

```bash
python generate_poster_visuals.py --show
```

- meant to give the 4 focused poster visuals: task success, task success discordance, minimum-subgoal completion rate, planner subgoal-count distribution
