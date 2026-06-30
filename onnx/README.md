# onnx/

Runnable policy weights for the two **alternative-reward baselines** from the controlled study
(see [`../docs/EXPERIMENTS.md`](../docs/EXPERIMENTS.md), Group C):

| file | tag | variant |
|---|---|---|
| `C1_mfg_stu40k_10k.onnx` | C1 | alt-reward, longer teacher→student distillation |
| `C2_mfg_stu30k_10k.onnx` | C2 | alt-reward, shorter distillation (fell on the turning gait) |

These are provided as **runnable example policies** that exercise the I/O contract end to end. The
A-series (`A1`/`A2`/`A3`) and the **adopted** deployed policy are withheld (commissioned IP).
The files contain inference weights only — no training data, optimizer state, or reward recipe.

## I/O
Both match the contract in [`../docs/POLICY_IO.md`](../docs/POLICY_IO.md):
observation `1432` · action `29` · `action_scale 0.25` · 50 Hz · CPU `onnxruntime`.

```python
import onnxruntime as ort, numpy as np
sess = ort.InferenceSession("onnx/C1_mfg_stu40k_10k.onnx", providers=["CPUExecutionProvider"])
obs = np.zeros((1, 1432), dtype=np.float32)
action = sess.run(None, {sess.get_inputs()[0].name: obs})[0]   # (1, 29)
```

> Behavior clips for every policy (including the withheld A-series) are linked from
> `docs/EXPERIMENTS.md`.
