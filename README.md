# Adaptive Cartesian Impedance Control with PPO

A self-contained Google Colab experiment for torque-controlled Franka Panda surface tracking in MuJoCo. An analytical Cartesian impedance controller handles robot dynamics while Proximal Policy Optimization (PPO) adapts translational stiffness online.

The [executed notebook](Adaptive_Panda_Impedance_PPO_colab_v3.ipynb) contains the implementation, controller checks, training, baseline comparisons, statistical tables, plots, and video-generation cells. The saved Colab run completed end to end and passed all 17 controller validation checks.

**Result:** this exploratory run did not demonstrate an overall advantage for PPO. Fixed and scheduled impedance achieved lower average position and filtered-force errors and higher task success rates on both ID and OOD evaluations. The project provides a working experimental framework and a recorded result, without assuming that learned gain adaptation outperforms classical control.

**How the controller works**

```text
Robot state + measured wrench + task reference
                    |
              PPO policy (20 Hz)
                    |
        Bounded changes to log(Kx, Ky, Kz)
                    |
   Cartesian impedance + inertia-based damping (100 Hz)
                    |
       Gravity compensation + null-space posture
                    |
          Torque limits → MuJoCo (500 Hz)
```

The primary policy has 59 observations and three actions. It adjusts translational stiffness between 50 and 1,000 N/m, starting at `(400, 400, 200)` N/m. Each action permits a stiffness ratio between `1/1.2` and `1.2`. Rotational stiffness remains fixed; damping is recomputed from stiffness and approximate effective Cartesian inertia. The primary PPO policy does not output joint torques.

All primary controllers share a bounded normal-force servo that shifts the equilibrium height using measured force error. The experiment therefore tests the incremental value of stiffness adaptation **with a common force servo**. Ten-second episodes include approach, contact acquisition, stabilization, and surface tracking. Safety checks include force, joint and workspace limits, velocity, persistent saturation, contact loss, and numerical validity.

**The uploaded run**

Although the notebook selects `MODE = "standard"`, its automatic runtime budget reduced the workload on the available CPU. The executed plan below is the basis for every result in this README.

| Item | Executed value |
|---|---|
| Runtime | Google Colab, Linux, Python 3.13.15, 2 reported CPU cores |
| Training device / rendering | CPU / OSMesa |
| Global seed | 20261001 |
| Independent training seeds per variant | 1: seed 11 |
| PPO variants | Full, no domain randomization, no stiffness-change penalty |
| Training transitions | 32,768 per variant; 98,304 total |
| Shared validation scenarios | 3 |
| Validation interval / required consecutive windows | 1,536 transitions / 3 windows |
| Evaluation scenarios per controller | 20 ID, 20 OOD, 12 disturbance |
| Evaluated controllers | 3 classical baselines + 3 PPO variants |
| Benchmark episodes | 312 across all controllers and splits |
| Controller validation | 17/17 checks passed |
| Training wall time | 39.7 minutes |
| Total project wall time | 80.9 minutes, excluding installation |

The original standard target was three seeds, 100,352 steps per variant/seed, nine validation scenarios, and 100 cases per ID/OOD split. **That full protocol was not executed.** The K+D and residual-torque extensions were also not run; they belong to extended mode.

All three PPO variants advanced to curriculum stage 2 through consecutive validation windows. Their transitions to stages 3 and 4 used the maximum-stage-step fallback. Completion of training should not be interpreted as convergence or mastery of each stage.

**Measured results**

The following means are transcribed from the saved notebook's section 24. Position RMSE is converted from metres to millimetres; force RMSE is the filtered wrist-normal tracking error. Safety is the fraction of episodes terminated with a safety reason, so lower is better. Each row uses 20 scenarios.

| Split | Controller | Position RMSE (mm) ↓ | Force RMSE (N) ↓ | Success ↑ | Safety termination ↓ |
|---|---|---:|---:|---:|---:|
| ID | Fixed impedance | 16.0 | 0.767 | 60% | 20% |
| ID | Scheduled impedance | 15.1 | 0.807 | 60% | 20% |
| ID | PPO full | 18.0 | 1.12 | 45% | 15% |
| ID | PPO, no randomization | 18.0 | 1.12 | 45% | 15% |
| ID | PPO, no stiffness-change penalty | 18.5 | 1.09 | 45% | 20% |
| OOD | Fixed impedance | 14.1 | 0.733 | 90% | 5% |
| OOD | Scheduled impedance | 13.2 | 0.789 | 90% | 5% |
| OOD | PPO full | 15.7 | 1.14 | 80% | 5% |
| OOD | PPO, no randomization | 15.7 | 1.14 | 80% | 5% |
| OOD | PPO, no stiffness-change penalty | 15.7 | 1.14 | 80% | 5% |

The notebook also reports a joint-PD baseline, raw physical-force RMSE, peak force, jerk, effort, work proxy, completion, settling, and recovery. For example, PPO full's raw physical-force RMSE was 3.34 N on ID and 2.99 N on OOD; the filtered-force column above is not a substitute for that physical contact metric.

For the paired **ID PPO-full minus fixed-impedance** comparison, the notebook reports:

| Metric | Mean difference | Exploratory 95% bootstrap CI |
|---|---:|---:|
| Position RMSE | +1.977 mm | +0.4405 to +3.589 mm |
| Filtered-force RMSE | +0.3511 N | +0.1308 to +0.6432 N |
| Success rate | −15 percentage points | −35 to +5 percentage points |

Positive error differences indicate worse tracking for PPO. These intervals describe scenario variability for one trained seed; they do not establish robustness across independent training runs and are not adjusted for multiple comparisons.

The disturbance suite applies 2, 5, and 8 N pushes in four tangential directions for 0.2 seconds. All six controllers completed all 12 disturbance episodes without safety termination. PPO full achieved 11/12 task successes and had one recovery failure; fixed and scheduled impedance each achieved 12/12 successes with no recovery failures. PPO's mean recovery time was 0.168 s over the 11 recovered cases, versus 0 s under the notebook's recovery criterion for fixed and scheduled impedance. Zero indicates that the criterion was already satisfied after the push, rather than physically instantaneous motion.

The full and no-randomization variants produced identical displayed summaries, and the reported OOD position-error difference was zero. This run does not demonstrate a domain-randomization benefit. Determining why the selected policies behave identically requires inspecting checkpoint selection, training histories, and saved weights; the summary alone cannot establish the cause.

**Run the notebook**

1. Open `Adaptive_Panda_Impedance_PPO_colab_v3.ipynb` in Google Colab using **File → Upload notebook**.
2. Select a Python CPU runtime. The notebook installs pinned dependencies and downloads the pinned Panda assets; no dataset upload or Drive mount is required.
3. Choose a mode in section 5 and run from the top. `debug` checks the workflow, `standard` enables a runtime-budgeted experiment, and `extended` uses longer training and additional ablations.
4. If section 3 explicitly reports already-loaded incompatible packages, restart the session once and run from the top again. The v3 startup guard disables the unused PyTorch TensorBoard integration to avoid its TensorFlow/XLA import path. Logging uses CSV.
5. Preserve the generated output directory before deleting the Colab runtime.

The installation probe checks imports, a MuJoCo simulation step, and rendering. If it fails, use the last `CHECK` line and error output for diagnosis; repeatedly deleting the runtime does not resolve a native-library conflict.

Automatic budgeting depends on measured throughput, so choosing `standard` on another machine can produce a different plan. To repeat this run's **workload**, make a separate copy of the notebook, set the `Config` default `auto_runtime_budget=False`, and add the following immediately after the preset `PLAN` assignment in section 5, before configuration is saved:

```python
PLAN.update(
    seeds=[11], steps=32768, n_test=20, n_val=3,
    eval_every=1536, windows=3,
)
```

Keep `MODE = "standard"` to retain its scenario seed ranges. Use a fresh output root for an independent run, or disable resumption; `resume=True` is the default. A preserved output tree allows checkpoint resumption with optimizer, normalization, and curriculum state, but resumption starts new episodes rather than replaying a partial rollout bit for bit. Floating-point results can differ across environments.

To execute the original full standard protocol instead, disable automatic budgeting and leave the standard preset unchanged. That produces a new experiment, not the uploaded workload.

**Dependencies and model provenance**

| Package | Version reported by the run |
|---|---|
| PyTorch | 2.7.0+cpu |
| MuJoCo | 3.3.2 |
| Gymnasium | 1.1.1 |
| Stable-Baselines3 | 2.6.0 |
| NumPy / SciPy | 2.2.6 / 1.15.3 |
| pandas / Matplotlib | 2.2.3 / 3.10.3 |
| imageio / imageio-ffmpeg / Pillow | 2.37.0 / 0.6.0 / 11.2.1 |

The Panda model is downloaded from MuJoCo Menagerie commit `4d038b3feae26ec82b46a4d586379114012a8ac7`. The notebook records asset hashes and downloads the model's license. It replaces the model's position actuators with torque motors, retaining the source force limits, fixes the gripper open, and adds a spherical contact probe. Upstream package and model licenses remain applicable.

**Generated artifacts**

The run reported the following output layout in Colab. These files are generated separately from the notebook and are included in a repository only if exported and uploaded alongside it.

```text
outputs/standard/
├── configs/       # Configuration, budget adjustment, assets and scenarios
├── checkpoints/   # Versioned models, normalization, RNG and curriculum state
├── logs/          # PPO CSV logs
├── metrics/       # Episodes, summaries, paired effects and failure counts
├── plots/         # PNG and PDF figures
├── videos/        # Three common-scenario demonstrations and their metrics
└── traces/        # Compressed per-episode trajectories
```

For numerical analysis, retain `metrics/episodes.csv`, `metrics/summary_numeric.csv`, `metrics/paired_effects.csv`, and `metrics/failures.csv`. For policy reuse, retain the selected checkpoint bundle and its matching `normalize.pkl`, plus the checkpoint pointer files. The notebook's final cell lists the generated artifacts; embedded notebook output is not a complete checkpoint archive.

**Scope and limitations**

This is a simulation study with one training seed and a small validation set. Early terminations censor continuous metrics, so errors must be read alongside success, completion, safety, and missing/recovery-failure counts. Success requires position RMSE below 2 cm, filtered-force RMSE below 2 N, at least 95% tracking completion, and no safety termination.

Contact compliance is solver-defined, sensing uses idealized compensation plus noise, and the controller has access to the simulated inertial model. Only the probe interacts with the task surface. Time-varying stiffness has no passivity guarantee; empirical safety checks do not establish hardware safety. Stiffness correlations are descriptive and cannot establish causal adaptation benefits.

The notebook's introductory development-status text predates this hosted execution. For this uploaded run, the saved cell outputs establish completed Colab execution of the reduced standard plan documented here.

Exact executed notebook SHA-256 (before any re-save):

```text
8a7f8aefc4bbc95aadb4067e6aa3e1dff8d169ca6570141d67825531436dadb9
```
