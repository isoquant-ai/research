# Isoquant on Physics-IQ Verified: I2V and V2V

Isoquant is the evaluated system, with Seedance 2.5 as its video generator.
It was evaluated on two separate Physics-IQ Verified tracks. I2V predicts five
seconds from one official switch frame; V2V predicts the next five seconds from
the complete three-second conditioning video. The tracks use different inputs,
so their results are reported separately.

| Track | Input | Generation system | Prompt | Best-of-N | Four-run result |
| --- | --- | --- | --- | ---: | ---: |
| [I2V](#i2v) | One official switch frame | Seedance 2.5 | BPP | 1 | **53.6869 ± 0.8578** |
| [V2V](#v2v) | Complete three-second conditioning video | Seedance 2.5 | BPP | 1 | **57.1489 ± 0.7887** |

<a id="i2v"></a>
## I2V

The I2V pipeline uses only the official switch frame and best-practice
description for each of the 198 cases. A fixed case-specific continuation
prompt is derived once and applied unchanged across the four Verified runs;
only the generation seed differs.

Each case produces one five-second video at 1280x720 resolution and 24 FPS. No
alternate output is generated or reranked, so this is a direct single-sample
result with Best-of-N equal to one.

| Run | Seed | Physics-IQ Verified score |
| ---: | ---: | ---: |
| `run_01` | 42 | 54.9656158112 |
| `run_02` | 20260830 | 53.1673826350 |
| `run_03` | 314159 | 53.2222188272 |
| `run_04` | 271828 | 53.3923346207 |
| **Mean** | | **53.6868879735** |
| **Standard deviation** | | **0.8578480388** |

<a id="v2v"></a>
## V2V

The V2V pipeline uses only the complete official three-second conditioning
video and best-practice description for each case. A fixed, target-free motion
planner converts them into continuation guidance. The guidance and generation
settings are unchanged across the four Verified runs; only the generation seed
differs.

Each case produces one five-second continuation at 1280x720 resolution and 24
FPS. There is no candidate routing or reranking, so this is also a direct
single-sample result with Best-of-N equal to one.

| Run | Seed | Physics-IQ Verified score |
| ---: | ---: | ---: |
| `run_01` | 42 | 58.1479982698 |
| `run_02` | 20260830 | 57.0311157059 |
| `run_03` | 314159 | 56.2256873041 |
| `run_04` | 271828 | 57.1908079639 |
| **Mean** | | **57.1489023109** |
| **Standard deviation** | | **0.7887011373** |

## Evaluation

Both results use 198 cases per run and report the official
`final_score_view` metric on a 0–100 scale. Means and standard deviations
are calculated across four independently generated runs.

- Benchmark protocol: [Physics-IQ benchmark at `f0e2def48005cac44080e55c245f703b152da808`](https://github.com/google-deepmind/physics-IQ-benchmark/tree/f0e2def48005cac44080e55c245f703b152da808)
- Evaluator: [`physiq/run_physics_iq.py` at the same pinned commit](https://github.com/google-deepmind/physics-IQ-benchmark/blob/f0e2def48005cac44080e55c245f703b152da808/physiq/run_physics_iq.py)
- Dataset: [Physics-IQ Verified at `b65f567dd8446a4076c7f72446335b3c217a8464`](https://huggingface.co/datasets/Anates-Labs-Research/Physics-IQ-Verified/tree/b65f567dd8446a4076c7f72446335b3c217a8464)
