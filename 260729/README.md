# 260729

## log

### minimal LTL pipeline snippet

Zijian provided this snippet that is the minimal LTL to reach/avoid sequence code:

```python
from sequence.search import ExhaustiveSearchSafety
from ltl.logic import Assignment
from ltl.automata import ltl2ldba, LDBA, LDBASequence

# LTL formula, encoded with temporal and logical operators + the propositions
formula = "F blue"
# propositions, verifiable characteristics of the environment
props = ["blue", "green", "yellow", "magenta"]
# generate all possible assignments (basically potential observations) from the propositions and our environment assumption of zero or one.
# zero or one means only max one can be true at once. we could change this.
possible_assignments = Assignment.zero_or_one_propositions(set(props))

search = ExhaustiveSearchSafety(None, None, props, num_loops=1)

# convert ltl to ldba automaton, a traversable graph
# nodes are "states". just think of them as like state 0, state 1, state 2. they encode progress
# directed edges are transitions. a transition has a logical condition that must be satisfied to go from one state to another
# a given assignment can either satisfy or not satisfy a transition's condition
ldba = ltl2ldba(formula, props, simplify_labels=False)
# remove transitions that are impossible under the set of possible_assignments
ldba.prune(possible_assignments)
# if an assignment is not handled by the existing outgoing transitions, this method adds a transition to a rejecting sink state
# sink state = once entered, can't leave (self loop). this is good because if you mess up, you shouldn't be able to un-mess-up
ldba.complete_sink_state()
# an SCC (strongly connected component) is a group of states where every state can reach every other state in that group.
ldba.compute_sccs()
# find initial SCC that the initial state is in
initial_scc = ldba.state_to_scc[ldba.initial_state]
# initial_scc.bottom means there is no transition from this SCC to another SCC
# not initial_scc.accepting means there is no transition in the SCC that is accepting
# together they means no proposition sequence can satisfy the formula
if initial_scc.bottom and not initial_scc.accepting:
    raise ValueError(f'The language of the LDBA for {formula} is empty.')

# each sequence is a list of steps like (reach_assignments, avoid_assignments)
# the reach set describes observations that advance along the chosen accepting path.
# the avoid set describes observations that would lead into rejecting or otherwise unusable parts of the automaton.
# we are searching for "acceptance": an infinite run that satisfies the LTL formula (usually self loops in satisfied state at end)
sequences = search.all_sequences(ldba, [ldba.initial_state])

print(f"sequences: {sequences}")
```

after some dependency/import setup and fixes, i got it running and working as `src/main.py` in the GenZ-LTL repo

### porting over non RL GenZ-LTL stuff to ros

next, i copied the LTL to reach/avoid sequence pipeline into its own standalone folder, aiming to exclude all RL and agent related code. this meant dependency reduction:

- removed PyTorch, Gymnasium, preprocessing, agent, model
- produced lightweight tookkit with only SymPy, Joblib, Java, and standard library dependencies

as a recap, here is what the entire pipeline looks like. this is what the minimal snippet is calling.

```
LTL formula
    → Rabinizer
    → HOA text
    → HOAParser
    → LDBA automaton
    → ExhaustiveSearch
    → reach-avoid sequences
```

vocab recap:

- **HOA**: Hanoi Omega-Automata format, a plain-text interchange format for representing automata over infinite sequences
- **LDBA**: Limit-Deterministic Büchi Automaton

next, i added shared unit tests for both the original GenZ-LTL and the toolkit to confirm the extraction behaves like the legacy implementation

next, i converted the toolkit to python 3.8

next, i packaged the toolkit into our ros 1 codebase

installing:

```bash
cd RISE-2026
rosdep install --from-paths src --ignore-src -r -y
catkin_make
```

and Rabinizer needs Java:

```bash
apt-get update
apt-get install -y openjdk-11-jre-headless
java -version

/root/ros_ws/src/ltl_toolkit/rabinizer4/bin/ltl2ldba \
  -i 'F blue' -p -d -e | head
```

run example:

```bash
rosrun ltl_toolkit ltl_toolkit_example.py
```

also, i converted to using all possible assignments (2^n), not just limited by zero or one, because it suits most of our potential tasks better

### replacing the vlm with the ltl solver
