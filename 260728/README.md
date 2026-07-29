# 260728

## sources

1. [GenZ-LTL repo](https://github.com/BU-DEPEND-Lab/GenZ-LTL/tree/main)
2. [GenZ-LTL paper](https://arxiv.org/abs/2508.01561)

## log

### paper review

**automaton**: model systems that process inputs and transition between states

- a finite automaton typically reads an input string one symbol at a time in order, and transitions states based on rules regarding the current state and symbol being read

#### introduction

- RL systems bad at generalizing to complex, long-horizon tasks with safety constraints
- LTL unambiguously defines task objectives and safety constraints
- safety constraints must be completely followed, while at the same time progress should be made toward the task as efficiently as possible
- GenZ-LTL is a learning framework that uses "equivalent Büchi automaton" to decompose an LTL spec into subgoals with a reach component representing task progression and an avoid component representing safety constraints
- surprisingly, the policy is trained to complete subgoals one at a time, because of the innovations
- innovation 1: **state-wise** constraints for solving subgoals
- innovation 2: **subgoal-induced observation reduction** to handle many observation-subgoal combinations

#### preliminaries

LTL

- formulas built from boolean operators:
  - negation
  - conjunction
  - disjunction
- temporal operators:
  - until (U): `x U y` is satisfied if x is satisfied at all time steps before first occurrence of y
  - eventually (F): `F x` is satisfied if x is satisfied at a future time
  - always (G): `G x` is satisfied if x is satisfied from current step onwards
- and atomic propositions

Büchi automata can formally represent LTL formulas

### misc

tried

- `Move the red block to the center of the table. Then, move the green block to center of the table. Finally, move the yellow block to the front right corner of the table.`
- `Move the red block to the center of the table. Then, move the green block to the center of the back left quadrant of the table.`
  they worked

defined close enough to target:

```
10. An object is considered at its target location if it is within 0.1 units of it. Do not move an object to its target location if it is already close enough to it.
```

### what we can reuse from GenZ-LTL

src/evaluation/simulate.py simulate

src/envs/ldba_wrapper.py LDBAWrapper

src/model/agent.py get_action

src/sequence/search/exhaustive_search.py dfs

- define propositions
- create automoton

### evaluation
