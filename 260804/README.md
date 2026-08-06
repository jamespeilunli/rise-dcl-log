# 260804

## log

### spatial order task

evaluation/gt changes:

- spawn location doesn't avoid center
- scoring changed
- dynamic optimal command count
- distances not considered

controller changes:

```
- For ordinary non-relative tasks, use the target's center x and y coordinates as the drop location and its bounding-box zmax as the placement height.
- For left/right tasks, preserve the source x and use the table zmax. Place it 0.1 m beyond the leftmost/rightmost reference center y. A smaller y is left. Keep the source bounding box within the table bounds.
```

the vlm change is simple:

```
Arrange the blocks so that the red block is to the left of the blue block and the blue block is to the left of the green block.
```

#### ltl changes

deciding the propositions is actually nontrivial.

suppose we keep all 6 combinations of one block in relation to another block

- red_left_of_blue
- red_right_of_blue
- red_left_of_green
- etc.

it doesn't make a lot of sense because they are interlinked. for example if red_left_of_blue is true then red_right_of_blue must be false (ignoring center)

instead you might describe every pair

- red_left_of_blue
- red_left_of_green
- blue_left_of_green

but actually there are still dependence. because if red_left_of_blue and blue_left_of_green, then necessarily red_left_of_green

why is this sort of dependence important?

because the way we choose the action based on desired propositions, we only want one proposition change in a transition. with this every pair setup you are guaranteed to have multiple changes in certain situations

additionally, there is the question of how to convert these desired propositions into a command. if we had certain multiple conditions, we could tell the controller to move a given block relative to both of them (like to the left of both, or in between both). but with only one proposition change, it's ambiguous where to put it in relation with the not mentioned block

i chose this setup:

```python
PROPOSITION_TO_ACTION = {
    "red_left_of_blue": (
        "Move the red block to the left of both the blue and green blocks."
    ),
    "blue_left_of_green": (
        "Move the green block to the right of both the red and blue blocks."
    ),
}
```

the thing is, idk if this is cheating because it's using knowledge about the task order to determine the optimal action given the proposition

anyway, this actually didn't help the accuracy. we got:

```
VLM: 92.0%
LTL: 89.5%
Exact p-value: 0.4583
```

and discordance had 17 vlm-only successes, and 12 ltl-only successes. why 17 ltl failures when vlm succeeded?

the planner kept repeating the same command because the low level controller struggled with reasoning about two blocks. specifically:

- it wasn't good at following "Move the green block to the right of both the red and blue blocks."
- it frequently used only one reference (often `red_y + 0.1`) instead of the required: `target_y = max(red_y, blue_y) + 0.1`
- after LTL had moved red far left, that mistake placed green left of blue. LTL correctly observed that blue_left_of_green was still false, but then retried the same problematic command until the 10-command limit
