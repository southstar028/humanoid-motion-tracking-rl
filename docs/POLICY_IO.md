# Policy I/O contract

The adopted policy's weights are not included (commissioned research output); two alternative-
reward baselines (`C1`, `C2`) are provided under [`../onnx/`](../onnx/) and run against this
contract. This is the interface every policy here exposes, so the training setup is legible
even where the weights are withheld, and the result is consistent with the deployment side.

- **Format** — ONNX (exported from the IsaacGym/rsl_rl checkpoint), run on CPU at deployment.
- **Observation** — `1432`-dim (proprioception + tracking-reference / mimic features).
- **Action** — `29`-dim, one target offset per controlled joint.
- **Control rate** — 50 Hz.
- **Action scaling** — `pd_target = clip(action, -10, 10) * action_scale + default_dof_pos`;
  `action_scale` is a policy-specific training constant carried into deployment.
- **Joints** — 29 controlled DoF; the neck (2 DoF) is held, not driven by the policy.
- **Reward terms** — standard whole-body mimic tracking (joint pose, key-body position,
  root/velocity), with the key-body target in the root-local convention described in
  `INSIGHTS.md` §1.

The real-robot side (low-level control server, sim2sim-over-DDS verification) is documented in a
separate deployment repository.
