# Engineering insights & debugging notes

The problems that actually decided whether the policy learned, ordered roughly by the time they
cost. Each is written as *symptom → diagnosis → fix → lesson*.

---

## 1. A silent double-rotation bug capped key-body tracking

**Symptom.** Across every training run (walk, whole-body, teacher), the key-body tracking
error started at ~0.13 m and stayed flat from iteration 0 — the policy never improved at
tracking key bodies, while every other reward learned normally.

**Diagnosis.** The key-body targets (`local_body_pos`, "lbp") are produced by the retargeting
stage. The training environment assumes lbp is **root-local** (joint pose only, root rotation
removed) and re-applies the root rotation at run time to place the targets in the world. The
generator had instead baked the root rotation into lbp already:

```python
# buggy: root_rot applied before FK  →  xpos is world-frame
qpos[3:7] = [w, x, y, z]            # actual root rotation
lbp = fk(qpos).xpos[body_ids]       # = R_root @ local_offset
```

So the environment computed `R_root @ (R_root @ local)` — a **double rotation**. Measured
against a faithful reconstruction, the targets were off by **22–48 cm**, which is why the
reward was unlearnable.

**Fix.** Generate lbp with an **identity root quaternion** so it is genuinely root-local:

```python
qpos[3:7] = [1, 0, 0, 0]            # identity → lbp is root-local
```

Only the lbp had to be recomputed (joint and root data were correct), so no full re-retarget
was needed.

**Verification.** Buggy data reconstructed to 22–48 cm error; after the fix, **0.000 cm** —
matching the expected root-local convention exactly.

**Lesson.** A reward that is *flat from iteration 0* is usually a wrong target, not a weak
policy. Reconstruct the training targets and diff them against the expected convention before
blaming the learner.

---

## 2. Cross-version pickle incompatibility silently broke the loader

**Symptom.** The dataset loader failed on the motion `.pkl` files with a `numpy._core` import
error.

**Diagnosis.** The motions had been pickled under **NumPy 2**, but the training environment is
pinned to **NumPy 1.23** (an IsaacGym-era constraint). NumPy 2 changed internal module paths, so
1.23 cannot unpickle 2.x arrays.

**Fix.** Re-serialize every motion file from within the consuming environment's NumPy.

**Lesson.** Data artifacts must round-trip through the *exact* runtime that loads them. Pinning
the training env is not enough if the data was produced elsewhere — cross-version pickle is a
silent, late-binding trap.

---

## 3. A plausible cause, disproved by measurement

**Symptom.** Standing "chatter" — the robot looked like it trembled at rest.

**Hypothesis.** The simulation's floor contact was very stiff (`solref 0.001 1`, ~one timestep)
versus a softer `solref 0.02 1`, which would cause foot-floor chatter.

**Test.** A controlled A/B on the *same* policy and server, changing only the MJCF floor
(stiff / fixed / soft) over 60 s of standing. Result: **all three** had zero falls and
sub-degree jitter. The floor stiffness was **not** the cause.

**Conclusion.** The real "trembling" was dynamic — it appeared under live teleoperation input,
not in the static simulation.

**Lesson.** A mechanistically plausible explanation is still a hypothesis. Isolate one variable
and measure; a clean A/B avoids "fixing" something that was never broken.

---

## 4. Idle jitter was missing data, not a missing reward

**Symptom.** The policy jittered when asked to stand still.

**Insight.** Quiet standing simply wasn't represented in the motion dataset, so the policy had
never learned to be still — the jitter was a gap in the data, not a missing reward term.

**Fix.** Augment the dataset with synthetic quiet-standing plus natural standing and transition
clips, then retrain.

**Lesson.** "The policy does X badly" frequently reduces to "X isn't in the data." Check the
data distribution before tuning rewards.

---

## 5. Curation beats volume

Retargeted public motion is noisy: clips with self-collision, foot-sliding, or waist-twist
teach the policy bad habits. Curating these out (by eye and by metric) mattered more than raw
clip count, and the *ratio* of clean public motion to self-captured data was a real tuning knob.
Over-precise per-clip fidelity, conversely, invites over-fitting — track what is kinematically
decisive and let the rest be approximate.

---

## 6. The smallest error message held the real fix

A wave of training segfaults presented as opaque core dumps. The decisive clue was a boring
import error underneath them — `libpython3.8.so.1.0: cannot open shared object file` — not the
core dumps themselves. Setting `LD_LIBRARY_PATH="$CONDA_PREFIX/lib:$LD_LIBRARY_PATH"` so the
conda env's libpython was found stabilized training. **Lesson:** the smallest error message
often holds the real fix — chase it before rebuilding environments, Docker images, or running
sanitizers for days.

---

## 7. A saturated target can't be tracked (shuffling gait)

**Symptom.** Shuffling gait; feet not planting flat. **Diagnosis.** In 16–27% of training
frames the ankle-roll *target* sat at the joint limit — the retargeted target itself over-drove
that joint, so the policy physically couldn't follow it. **Fix.** Move that target from ankle to
toe, or drop its rotation weight to 0 (position-only IK) → saturation 36% → 0%; unify ground
height. (Fine flat-foot contact is ultimately better solved by the sim's physics during training
than by perfecting the kinematic target.) **Lesson:** check whether the *target* is feasible
before blaming the policy.

---

## 8. "Same ONNX, different behavior" is an environment-parity problem

The same policy weights can produce different behavior across two environments. Identical weights
and URDF — but MJCF physics (timestep/solver, joint limits, friction/armature/damping, contact,
COM), observation normalization and history priming, PD gains, device (CUDA vs CPU determinism),
and the message channels **all** have to match; any one mismatch changes behavior. **Lesson:**
reproducibility is an environment-parity problem — pin and diff the whole stack, not just the
network.

---

## 9. A silent crash ran for hours unnoticed → cron health-check

A run died early (a module-path / library-version error) and went unnoticed for hours. Fix:
move health monitoring from a fragile daemon to a **cron check + auto-restart** on the same
config, with 100-iteration checkpoints so any run is resumable. **Lesson:** for unattended
training, assume it will die and design the monitor + restart policy up front.

---

## 10. Know exactly what a reward term measures before tuning it

Raising the waist tracking weight was expected to remove upper-body twist; it didn't. The
`dof_err` term penalizes deviation from the *reference* pose, not absolute twist — and the
reference pose carried no torso-orientation signal. Fixing it properly would need an explicit
orientation term plus body-rotation data the dataset didn't have. **Lesson:** a reward term that
is misunderstood is a knob that doesn't do what you think.
