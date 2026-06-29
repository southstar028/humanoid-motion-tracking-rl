# Experiments & the policies produced

The whole-body tracking work ended with **five trained policy variants** (plus a pre-transfer
baseline), produced by a **controlled study that changed one factor at a time** so the effect of
each could actually be attributed.

## Why a controlled study
Early on, multiple things were changed at once (dataset + reward + weights), which made it
impossible to tell what helped. The fix was to split the runs into groups that each isolate a
single variable — see `INSIGHTS.md` (“isolate one variable”).

## Groups
- **Group A — tracking-weight emphasis.** Which joint group's pose-tracking weight to raise:
  `A1` hip-emphasis, `A2` waist-emphasis, `A3` both (waist+hip). Same dataset/reward otherwise.
- **Group C — reward formulation × distillation length.** An alternative reward formulation at
  two teacher→student distillation lengths: `C1` (longer), `C2` (shorter).
- **(Group B — dataset variant.)** My curated whole-body set vs. an alternative source set,
  holding reward/weights fixed.

## The five policies
| tag | variant | outcome |
|---|---|---|
| **A1** | hip-weight emphasis | comparison |
| **A2** | waist-weight emphasis | comparison |
| **A3** | waist + hip (both) | **adopted** — passed the sim2sim gate; deployed policy |
| **C1** | alt-reward, longer distillation | comparison |
| **C2** | alt-reward, shorter distillation | fell during the turning gait → rejected |
| *(pre-transfer)* | baseline before this port | reference only |

All export to ONNX (obs 1432 / action 29, action_scale 0.25); see `POLICY_IO.md`. Weights are
withheld (commissioned IP) — this documents *which experiments produced what*, not the weights.

## Adoption criterion (the gate, not the loss)
A policy was “good” only if it passed the **sim2sim parity gate** — 60 s standing with no fall
and sub-degree jitter, upright tracking through a turning gait, and a dynamic-transition clip.
`A3` (waist+hip) passed; the alternative-reward variants degraded on the turning gait (`C2`
fully fell). Training reward magnitude was *not* used to pick the winner.

## A result worth remembering
Raising the **waist** tracking weight alone did **not** remove upper-body twist — because the
`dof_err` term penalizes deviation from the *reference*, not absolute twist, and the reference
itself lacked a torso-orientation signal. Fixing torso twist properly would need an explicit
orientation term plus body-rotation data the dataset didn't carry. (See `INSIGHTS.md`.) This is
why the controlled study mattered: it showed the knob I expected to work didn't, and why.
