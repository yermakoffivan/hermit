# Prompt-only

The simplest Evolver. At each training step one model call reads the current `ADAPTER.md` and the step's results
(each task's input, the agent's answer, the judge's score and feedback) and writes a new `ADAPTER.md`. It needs no
agent and no validation split; the run's `llm` is the optimizer.

## One step

1. The Trainer runs the current adapter on `data.step_size` training tasks and scores them.
2. The `llm` sees up to `max_examples` of them, lowest scores first, with the task's `task-info/README.md`, the
   adapter's `EVOLVER.md` and its own summaries of the last five steps.
3. It replies with the complete new file and a one-line summary, kept in `state/notes.jsonl`.
4. The Trainer checks the new adapter against the template's specification and commits it.

## Knobs

| Setting | Meaning |
|---|---|
| `llm` | The optimizer's endpoint and model |
| `data.step_size` | Training tasks per step |
| `evolver.params.files` | The Markdown files it rewrites (default `[ADAPTER.md]`) |
| `evolver.params.max_examples`, `max_chars` | Tasks shown per step (8) and characters per field (3000) |
| `evolver.params.max_tokens`, `temperature` | The model call (32000, 0.3) |

All parameters and defaults: [`harnessadapter/evolvers/prompt_only/evolver.yaml`](../../harnessadapter/evolvers/prompt_only/evolver.yaml).

## Examples

| Folder | Task | Configs |
|---|---|---|
| [`officeqa/`](officeqa/) | [OfficeQA](../tasks/officeqa/); GLM 5.3 Flash acts and optimizes | `smoke.yaml`: 4 training questions in 2 steps, 2 test questions<br>`try.yaml`: 12 training questions in 3 steps, 10 test questions<br>`run.yaml`: 60 training questions in steps of 10, 2 epochs, 60 test questions |
| [`visual_showcase/`](visual_showcase/) | [Visual showcase](../tasks/visual_showcase/); GLM 5.3 Flash writes an animated web page and optimizes from the score and the judge's feedback text only, never the page or its screenshots | `smoke.yaml`: case `waves`, 2 steps<br>`run_waves.yaml`, `run_lava.yaml`, `run_silk.yaml`: one adapter per case, 6 steps, 3 eval repeats, as in [`mwm_visual/`](../mwm_visual/) |

## Run

```bash
python examples/tasks/officeqa/prepare.py /path/to/officeqa/prepared
export FIREWORKS_API_KEY=...
harnessadapter check --config examples/prompt_only/officeqa/smoke.yaml
harnessadapter train --config examples/prompt_only/officeqa/smoke.yaml
harnessadapter eval  --config examples/prompt_only/officeqa/smoke.yaml --adapter final
harnessadapter eval  --config examples/prompt_only/officeqa/smoke.yaml --adapter none
```

`--adapter none` runs the agent without an adapter, the baseline to compare with.

The visual showcase needs Node 20 or newer and Google Chrome (the
[task's requirements](../tasks/visual_showcase/README.md#requirements)):

```bash
npm ci --prefix examples/tasks/visual_showcase/runtime
python examples/tasks/visual_showcase/prepare.py
export FIREWORKS_API_KEY=...
harnessadapter check --config examples/prompt_only/visual_showcase/smoke.yaml --no-probe
harnessadapter train --config examples/prompt_only/visual_showcase/smoke.yaml
harnessadapter train --config examples/prompt_only/visual_showcase/run_waves.yaml
for n in 1 2 3 4 5; do
  harnessadapter eval --config examples/prompt_only/visual_showcase/run_waves.yaml --adapter waves-po/ckpt-step-$n
done
```

`waves-po/ckpt-step-N` is `ADAPTER.md` after N rewrites (W1 to W5), the adapter of the same step as `mwm_visual`'s
`waves-r5/ckpt-step-N`; `final` is W6, a sixth rewrite that training never ran, so compare the methods on the
checkpoints. Group eval results by `output.arm`: a checkpoint whose rewrites changed nothing is still W0 and runs as
arm B. `--adapter none` is arm B, the same baseline as `mwm_visual`'s. The scores are laid out as in
[`mwm_visual`](../mwm_visual/README.md#run), under `examples/prompt_only/visual_showcase/runs/waves-po/`.
