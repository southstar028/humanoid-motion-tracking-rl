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
Each policy has a behavior clip (sim2sim render, unlisted YouTube). ONNX weights for the two
alternative-reward baselines (`C1`, `C2`) are included under [`onnx/`](../onnx/) as runnable
reference; the A-series and the adopted policy are withheld (IP).

| tag | variant | outcome | clip | weights |
|---|---|---|---|---|
| **A1** | hip-weight emphasis | comparison | [▶ clip](https://youtu.be/iiOMQr-a9AY) | withheld |
| **A2** | waist-weight emphasis | comparison | [▶ clip](https://youtu.be/4fY-QMg_2xI) | withheld |
| **A3** | waist + hip (both) | **adopted** — passed the sim2sim gate; deployed policy | [▶ clip](https://youtu.be/4Y55sW4e17w) | withheld |
| **C1** | alt-reward, longer distillation | comparison | [▶ clip](https://youtu.be/AxSlId43vZU) | [`onnx/C1_mfg_stu40k_10k.onnx`](../onnx/) |
| **C2** | alt-reward, shorter distillation | fell during the turning gait → rejected | [▶ clip](https://youtu.be/FXMSjhw95A8) | [`onnx/C2_mfg_stu30k_10k.onnx`](../onnx/) |
| *(pre-transfer)* | baseline before this port | reference only | — | withheld |

All export to ONNX (obs 1432 / action 29, action_scale 0.25); see `POLICY_IO.md`. The included
`C1`/`C2` weights are runnable against the I/O contract; the adopted policy and the A-series
weights are withheld (commissioned IP). This documents *which experiments produced what*.

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
