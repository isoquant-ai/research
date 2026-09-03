# Isoquant on Physics-IQ Verified: I2V and V2V

**Technical report · September 2026**

## Abstract

Isoquant is an inference-time physics-guidance system for video generation. It
uses track-specific multimodal planners to convert the permitted Physics-IQ
inputs into compact physical motion plans, compiles those plans into
deterministic generation prompts, and supplies the prompts to an unchanged
Seedance 2.5 generator. No Seedance weights were trained or fine-tuned for these
results.

We evaluate two separate Physics-IQ Verified tracks. I2V receives one official
switch frame and predicts the following five seconds. V2V receives the complete
three-second conditioning video and predicts a five-second continuation. Across
four independently generated runs, Isoquant obtains **53.6869 ± 0.8578** on I2V
and **57.1489 ± 0.7887** on V2V. Both are direct single-sample results: one
candidate is generated per case, with no output selection, routing, or
reranking.

| Track | Permitted visual input | Prompt policy | Candidates per case | Four-run result |
| --- | --- | --- | ---: | ---: |
| [I2V](#i2v-method) | One official switch frame | BPP + Isoquant guidance | 1 | **53.6869 ± 0.8578** |
| [V2V](#v2v-method) | Full three-second conditioning video | BPP + Isoquant guidance | 1 | **57.1489 ± 0.7887** |

## System definition

The submitted systems are named **Seedance 2.5 + Isoquant**. Seedance 2.5 is
the video generator; Isoquant is the planning and prompt-compilation layer used
before generation. The `bpp` label on the submission cards identifies the
official best-practice prompt as the starting prompt policy. It does not mean
that the unmodified BPP text was sent directly to the generator.

The two inference paths are:

```text
I2V: BPP row + switch frame
     -> structured physical plan -> deterministic I2V prompt
     -> Seedance 2.5 with one image -> five-second video

V2V: BPP row + three-second conditioning clip
     -> structured motion-timing plan -> deterministic V2V prompt
     -> Seedance 2.5 with one reference video -> five-second continuation
```

I2V provides no temporal evidence, so its planner represents causal motion,
direction, completion time, and physical invariants from the BPP description
and single switch frame. V2V observes motion directly, so its planner focuses
on whether that motion settles, continues, or remains quiescent, and when
settling occurs.

The planners use `gemini-3.7-flash` with high thinking, temperature 0, and
schema-constrained JSON output. A planner is run once per benchmark case. Its
validated record is compiled locally into the final prompt, and the resulting
198-prompt map is then frozen and reused for all four generation seeds.

## Method

### Shared information boundary

Each planner receives only the inputs available to its track and the
corresponding public BPP row. Neither planner receives the five-second target
video, target masks, generated candidates, Physics-IQ scores, per-case evaluator
results, or benchmark outcomes. The generator likewise receives only the final
prompt and the visual input allowed by the track.

The source BPP file is
[`descriptions/best_practice/descriptions_base.csv`](https://github.com/google-deepmind/physics-IQ-benchmark/blob/f0e2def48005cac44080e55c245f703b152da808/descriptions/best_practice/descriptions_base.csv)
at the pinned benchmark revision, with SHA-256
`20ffd208acc0b0f50d4638d1da69218168e78336e96118244a53d0ae046729c8`.

The prompt methods were selected using Physics-IQ Original. Before evaluation
on Physics-IQ Verified, the final per-case prompts, Seedance version, generation
settings, and four seeds were fixed. Verified targets, masks, generated outputs,
and evaluator results were not used to alter prompts or select videos.

### I2V method

For each of the 198 cases, the I2V planner receives:

1. the official take-1 BPP row, including its description and physics category;
2. exactly one official switch-frame image.

The compiler consumes a validated physics-plan record with the following
fields:

| Field | Meaning |
| --- | --- |
| `motion_class` | One of ballistic, collision, deformation, field interaction, flow, optical change, oscillation, phase change, quiescent, rolling, rotation, sliding, or translation |
| `interactions` | One to six causal contacts or interactions visible or implied by the allowed inputs |
| `direction` | Expected motion in image-relative coordinates |
| `settling_time_seconds` | `0` for a stationary scene; `null` or `5` for motion through the full window; or a value strictly between `0` and `5` for finite completion |
| `family` | Solid Mechanics → solid; Fluid Dynamics → fluid; Optics → optics; Magnetism → magnetic; Thermodynamics → thermal |
| `invariants` | One to six constraints intended to preserve the relevant physical state and interaction |

The compiler validates the record and inserts it into one fixed template:

```text
{BPP}

Use the supplied official switch frame as the exact first frame. Keep the
camera fixed in one continuous real-time shot. Animate only the described
natural physical motion for 5 seconds. Motion plan: {motion_class}.
Interactions: {interactions}. Direction: {direction}. {timing_instruction}
Physics family: {family}. Preserve these physical invariants: {invariants}.
```

Here and below, `{BPP}` denotes the exact `description` cell from the pinned BPP
CSV. The timing instruction is deterministic. A value strictly between `0` and
`5` asks the generator to complete the motion by that time and hold the result;
`null` or `5` continues naturally through the full window; `0` keeps the setup
at rest. For example, case `0001` is represented as ballistic motion directed
downward, with gravitational fall and pillow impact as the causal interactions,
a 1.2-second completion time, the solid family, and invariants covering gravity,
impact dissipation, and object rigidity. The complete generator-facing text for
every case is public in the
[I2V descriptions file](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/blob/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3/i2v/descriptions.csv).

The frozen I2V descriptions file has SHA-256
`5f8b36f9f4bd310c9518be9d2b0e066bdd497a67da227a59214e4b87b52dec6e`.
Every run uses this same file; only the Seedance seed changes.

### V2V method

For each case, the V2V planner receives the complete official three-second
conditioning clip and its BPP description. It emits exactly two fields:

| Field | Allowed value and interpretation |
| --- | --- |
| `motion_class` | `settling`, `continuous`, or `quiescent` |
| `stop_time_seconds` | Strictly between `0` and `5` for settling; `null` or `5` for continuous motion; `null` or `0` for quiescence |

This separates a common continuation failure into an explicit timing decision:
whether existing motion should settle during the prediction window, continue
through it, or remain absent. The record is rendered into a fixed continuation
template:

```text
Extend [Video1] forward by 5 seconds. {BPP} Continue directly from the final
frame of [Video1]. The exactly 5-second continuation begins immediately after
the final frame as a single static locked-off continuous shot with no cuts. Do
not replay, reset, or show the conditioning segment; keep everything else the
same. {timing_instruction}
```

For `settling`, the timing instruction continues the existing motion until the
predicted stop time and then holds the settled state. For `continuous`, it asks
for natural motion throughout all five seconds. For `quiescent`, it asks for no
new motion. Continuous and quiescent values are normalized to `5` and `0`
before prompt compilation. The complete generator-facing text is public in the
[V2V descriptions file](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/blob/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3/v2v/descriptions.csv).

The frozen V2V descriptions file has SHA-256
`9082b2baa52f18b7f4914d4c4c4f200a1d3178727008f0960ee89488a46a4ea9`.
Its motion plans comprise 107 settling cases, 72 continuous cases, and 19
quiescent cases. The same prompt map is reused across all four seeds.

## Video generation

Both tracks use the hosted Replicate endpoint for `bytedance/seedance-2.5` at
the immutable provider version:

```text
ca38262bae0952bf80a7f10eda58af860a0eae7d48957a099e32632792b8f116
```

| Setting | I2V | V2V |
| --- | --- | --- |
| Visual request field | One `image` | One-element `reference_videos` list |
| Aspect ratio | Adaptive | Adaptive |
| Requested duration | 5 seconds | 5 seconds |
| Requested resolution | 720p | 720p |
| Output format | MP4 | MP4 |
| Audio | Disabled | Disabled |
| Watermark | Disabled | Disabled |
| Seeds | 42, 20260830, 314159, 271828 | 42, 20260830, 314159, 271828 |
| Candidates per case | 1 | 1 |
| Routing or reranking | None | None |

Every case uses the same Seedance model, immutable version, and generation
configuration; there is no per-case model or checkpoint route. Each case has
one usable generated candidate. Retries are limited to terminal provider
failures that returned no output.

The submitted artifacts are H.264 MP4s at 1280×720, 24 FPS, 120 frames, and
5.000 seconds. I2V outputs are normalized to the media contract with FFmpeg:
24-FPS conversion, last-frame padding when required, a 120-frame limit, no
audio, and H.264 encoding with the medium preset, CRF 10, and `yuv420p` pixel
format. V2V outputs already satisfy the frame contract; they are stream-copy
remuxed without a lossy video re-encode, with audio and container metadata
removed.

## Evaluation protocol

- Benchmark and evaluator: [`google-deepmind/physics-IQ-benchmark` at `f0e2def48005cac44080e55c245f703b152da808`](https://github.com/google-deepmind/physics-IQ-benchmark/tree/f0e2def48005cac44080e55c245f703b152da808)
- Verified dataset: [`Anates-Labs-Research/Physics-IQ-Verified` at `b65f567dd8446a4076c7f72446335b3c217a8464`](https://huggingface.co/datasets/Anates-Labs-Research/Physics-IQ-Verified/tree/b65f567dd8446a4076c7f72446335b3c217a8464)
- Cases per run: 198
- Runs per track: 4 independent generation seeds
- Reported metric: official `final_score_view × 100`
- Dispersion: standard deviation across the four runs, using the `n - 1` denominator

The results below are self-run outputs of the pinned official evaluator. They
are not described as organizer-verified or leaderboard-accepted results.

| Run | Seed | I2V score | V2V score |
| ---: | ---: | ---: | ---: |
| `run_01` | 42 | 54.9656158112 | 58.1479982698 |
| `run_02` | 20260830 | 53.1673826350 | 57.0311157059 |
| `run_03` | 314159 | 53.2222188272 | 56.2256873041 |
| `run_04` | 271828 | 53.3923346207 | 57.1908079639 |
| **Mean** | | **53.6868879735** | **57.1489023109** |
| **Standard deviation** | | **0.8578480388** | **0.7887011373** |

The official component metrics, aggregated across the same four runs, are:

| Metric (`× 100`) | I2V mean ± standard deviation | V2V mean ± standard deviation |
| --- | ---: | ---: |
| `final_score_view` | **53.6869 ± 0.8578** | **57.1489 ± 0.7887** |
| `score_mse_view` | 46.8143 ± 0.6262 | 50.8349 ± 0.8772 |
| `score_spatial_view` | 67.9775 ± 1.3223 | 67.9751 ± 1.4266 |
| `score_spatiotemporal_view` | 45.5339 ± 0.8686 | 53.9895 ± 0.7188 |
| `score_weighted_spatial_view` | 54.4218 ± 1.3056 | 55.7960 ± 1.4000 |

## Public evidence and reproduction

The immutable artifact release is
[`isoquant-labs/physics-iq-verified@a69bd49f`](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/tree/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3).
It contains all 1,584 evaluated MP4s, the exact final prompts, per-run result
CSVs and metric JSON files, and score evidence.

| Artifact | I2V | V2V |
| --- | --- | --- |
| Exact final prompts | [`descriptions.csv`](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/blob/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3/i2v/descriptions.csv) | [`descriptions.csv`](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/blob/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3/v2v/descriptions.csv) |
| Four generated runs | [`i2v/run_01`–`run_04`](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/tree/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3/i2v) | [`v2v/run_01`–`run_04`](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/tree/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3/v2v) |
| Evaluator outputs | [`i2v/scores`](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/tree/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3/i2v/scores) | [`v2v/scores`](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/tree/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3/v2v/scores) |
| Score record | [`score_evidence.json`](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/blob/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3/i2v/score_evidence.json) | [`score_evidence.json`](https://huggingface.co/datasets/isoquant-labs/physics-iq-verified/blob/a69bd49f19b04afbcb5a59781a89ce9ae5d149b3/v2v/score_evidence.json) |

After downloading a track's four run directories and the pinned Verified
benchmark, the official evaluator can be rerun with:

```bash
uv run physiq/run_physics_iq.py \
  --input_folders run_01 run_02 run_03 run_04 \
  --output_folder evaluation_output \
  --descriptions_file descriptions.csv \
  --benchmark_base_folder <folder-containing-physics-IQ-benchmark-verified>
```

The four generated result CSVs can then be aggregated with the official helper:

```bash
uv run physiq/aggregate_runs_from_csvs.py \
  evaluation_output/physics-IQ-benchmark-verified/results/run_01.csv \
  evaluation_output/physics-IQ-benchmark-verified/results/run_02.csv \
  evaluation_output/physics-IQ-benchmark-verified/results/run_03.csv \
  evaluation_output/physics-IQ-benchmark-verified/results/run_04.csv \
  --score-type verified
```

Because Seedance 2.5 is a hosted generator, a new API invocation is not claimed
to reproduce the submitted MP4 bytes bit-for-bit. The immutable videos and
evaluator records above are the exact artifacts used for the reported scores;
the disclosed prompts, version, inputs, settings, and postprocessing define the
corresponding inference procedure.
