# Motion dataset construction

The whole-body-tracking policy is only as good as its reference motion. The dataset combines
retargeted public motion with self-captured teleoperation, curated for quality.

## Sources

- **Public motion** — AMASS and OMOMO, retargeted to the robot via GMR (see the retargeting
  work for how per-robot alignment/scale is derived).
- **Self-captured VR** — teleoperation motion recorded with a PICO headset (XRoboToolkit),
  retargeted online through GMR. This captures task-like, in-distribution motion that public
  datasets lack.

## Steps

1. **Retarget** public clips to the robot (`retarget_wbc.py`, `omomo*_to_igris.py`), producing
   per-clip joint trajectories plus root pose.
2. **Key-body targets (lbp)** — compute root-local key-body positions by forward kinematics
   with an **identity root** (see `INSIGHTS.md` §1 — getting this convention wrong silently
   breaks tracking).
3. **Curate** — drop clips with self-collision, foot-sliding, ground penetration, or
   waist-twist artifacts. Curation is done both by eye and by metric; clip count matters less
   than clip quality.
4. **Mix** — combine curated public motion with the VR-captured set. The mix *ratio* is a real
   tuning knob (the exact production ratio is proprietary and withheld here).
5. **Serialize** for the trainer — re-save under the consuming environment's NumPy
   (see `INSIGHTS.md` §2).

## Notes

- The progression went **walk-only → whole-body**: a smaller walking set first to validate the
  pipeline, then expansion to a full whole-body set once the gate passed.
- Quiet-standing augmentation was added after observing idle jitter (`INSIGHTS.md` §5).
- AMASS and OMOMO must be obtained from their own sources under their licenses; they are not
  redistributed here.
