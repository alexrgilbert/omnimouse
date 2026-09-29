# OmniMouse Handoff Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use sensei:subagent-driven-development (recommended) or sensei:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `notebooks/inference.ipynb` and the surrounding repo setup runnable by an external collaborator who has only followed the public README.

**Architecture:** Edits are confined to one notebook, the README, two container config files, and one defensive tweak in `omnimouse/distributed/utils.py`. No model, masking, or eval semantics change. Notebook edits are made via the `NotebookEdit` tool (or `json` round-trip) so the `.ipynb` stays valid nbformat 4.2.

**Tech Stack:** Jupyter (nbformat 4.2), Hydra 1.3 + OmegaConf, rootutils, Docker Compose / devcontainer.

**Linear:** Project [OmniMouse Handoff](https://linear.app/metamorphic/project/omnimouse-handoff-473318476a49) (P-MOD-659) — MOD-1552 … MOD-1558.

**Worktree:** `.worktrees/omnimouse-handoff-cleanup` on branch `alex/omnimouse-handoff-cleanup`.

**Verification constraint:** This machine has no dataset and no CUDA. Every task below is verified *statically*. End-to-end notebook execution is explicitly handed off to a GPU+data agent — see "Handoff Verification Checklist" at the end. Do not claim the notebook runs.

---

## Cell Index Reference

Cell indices in the current notebook (0-based, from `nbformat` cell list):

| Idx | Type | Content |
|---|---|---|
| 1 | code | `sys.path.insert` HACK block |
| 2 | code | all imports + trailing `rootutils.setup_root('..')` |
| 6 | code | checkpoint `snapshot_download` + `compose_omnimouse_config` |
| 7 | md | MLflow setup instructions (internal hostname) |
| 8 | code | `ENABLE_LOGGING` + logger instantiate |
| 15 | md | `dataset_collection` docs (bad collection names, `/data/mouse`) |
| 16 | code | `dataset_collection` + `setup_dataloaders_single_rank` |
| 44 | code | run eval + `log_to_loggers` |

**Indices shift as cells are inserted.** Re-read the cell list before each task rather than trusting these after Task 1.

---

## Task 1: Fix the notebook bootstrap (MOD-1552)

**Files:**
- Modify: `notebooks/inference.ipynb` cells 1, 2

- [ ] **Step 1: Replace cell 1 entirely**

Old cell 1 (delete all of it):

```python
# TODO: Replace with your own username / path. HACK: This is a hack to make sure the local repo is recognized when using the UV kernel
import sys
USERNAME = None
PROJECT_ROOT = f'/src/omnimouse' # f'/home/{USERNAME}/workspace/omnimouse'
if PROJECT_ROOT not in sys.path:
    sys.path.insert(0, PROJECT_ROOT)
```

New cell 1:

```python
# Resolve the repo root and put it on sys.path BEFORE any `omnimouse` import.
#
# `setup_root` walks up from `search_from` looking for the `.project-root` marker, then:
#   - pythonpath=True  -> prepends the repo root to sys.path
#   - dotenv=True      -> loads `.env` (MLFLOW_*, HF_TOKEN, ...)
#   - sets the PROJECT_ROOT env var, which `configs/paths/default.yaml` requires
#     (`root_dir: ${oc.env:PROJECT_ROOT}`), and from which `paths.data_dir` derives.
#
# Searching from Path.cwd() (not '..') means this works whether the kernel was
# started in `notebooks/` or at the repo root.
from pathlib import Path

import rootutils

root = rootutils.setup_root(
    Path.cwd(),
    indicator=".project-root",
    pythonpath=True,
    dotenv=True,
)
print(f"Project root: {root}")
```

- [ ] **Step 2: Remove the now-duplicate rootutils lines from the bottom of cell 2**

Delete these trailing lines from cell 2:

```python


# Setup root directory
root = rootutils.setup_root('..', indicator=".project-root", pythonpath=True)
```

Also remove `import rootutils` from cell 2's import block (it now lives in cell 1). Keep every other import in cell 2 unchanged.

- [ ] **Step 3: Verify the notebook is still valid JSON/nbformat**

Run:
```bash
python3 -c "
import json; nb=json.load(open('notebooks/inference.ipynb'))
print('cells:', len(nb['cells']), 'nbformat:', nb['nbformat'], nb['nbformat_minor'])
print(''.join(nb['cells'][1]['source']))
"
```
Expected: valid JSON, cell 1 is the new bootstrap, no `sys.path.insert` anywhere.

- [ ] **Step 4: Verify no hardcoded container paths remain**

Run:
```bash
grep -n "src/omnimouse\|USERNAME\|sys.path.insert" notebooks/inference.ipynb
```
Expected: no output.

- [ ] **Step 5: Verify cwd-independence of the root search**

Run from the repo root and from `notebooks/`:
```bash
python3 -c "
import rootutils, pathlib
print(rootutils.setup_root(pathlib.Path.cwd(), indicator='.project-root', pythonpath=True, dotenv=True))
"
cd notebooks && python3 -c "
import rootutils, pathlib
print(rootutils.setup_root(pathlib.Path.cwd(), indicator='.project-root', pythonpath=True, dotenv=True))
"
```
Expected: both print the same repo root path. (Requires `rootutils` installed; if unavailable locally, mark this step for the GPU agent.)

- [ ] **Step 6: Commit**

```bash
git add notebooks/inference.ipynb
git commit -m "fix(notebook): replace sys.path hack with cwd-independent rootutils bootstrap

The omnimouse imports ran before setup_root(pythonpath=True), so cell 1
inserted a hardcoded devcontainer path to compensate. Move a single
setup_root call ahead of all imports and search from Path.cwd() so root
resolution no longer depends on the kernel's working directory.

Refs MOD-1552"
```

---

## Task 2: Make MLflow logging optional (MOD-1556)

**Files:**
- Modify: `notebooks/inference.ipynb` cells 7 (md), 8 (code), 44 (code)

- [ ] **Step 1: Replace cell 7 markdown**

New content:

```markdown
### Experiment logging (optional)

This notebook can log metrics to [MLflow](https://mlflow.org/). **It is off by default** and
is not required to run inference.

To enable it, point the run at an MLflow tracking server you control by adding the following
to your `.env` at the repo root (loaded automatically by the bootstrap cell above):

```
MLFLOW_TRACKING_URI=https://your-mlflow-server.example.com/
MLFLOW_TRACKING_USERNAME=your-username
MLFLOW_TRACKING_PASSWORD=your-password
```

Then set `ENABLE_LOGGING = True` in the next cell.

If `MLFLOW_TRACKING_URI` is unset, the next cell skips logging and the rest of the notebook
runs normally.
```

Note: no internal hostname, no shared username, no "ask the team for the password".

- [ ] **Step 2: Replace cell 8 code**

```python
# TODO: Enable/disable logging.
# Requires a reachable MLflow server via MLFLOW_TRACKING_URI in your `.env` (see above).
ENABLE_LOGGING = False

logger = None
if ENABLE_LOGGING:
    if not os.getenv("MLFLOW_TRACKING_URI"):
        # cfg.trainer.logger.tracking_uri is `${oc.env:MLFLOW_TRACKING_URI}` with no default,
        # so instantiating without it raises an opaque OmegaConf resolution error.
        print(
            "Skipping MLflow logging: MLFLOW_TRACKING_URI is not set.\n"
            "Set it in your `.env` to enable logging, or leave ENABLE_LOGGING = False."
        )
    else:
        logger: Logger = hydra.utils.instantiate(cfg.trainer.logger)
        run_url = os.path.join(
            logger._tracking_uri, "#", "experiments", logger.experiment_id, "runs", logger.run_id
        )
        print(f"Logging to MLflow ({run_url})")
```

This also tidies a pre-existing wart. The original embeds a *multi-line* expression inside
f-string braces, which is legal only from Python 3.12 (PEP 701) — fine here, since
`pyproject.toml` requires `>=3.12`, so this is readability, not a bug fix. The actual small
bug it does fix: the original literal `f"Logging to MLFlow ({...}"` never closes its
parenthesis. Hoisting `run_url` out makes both problems go away.

- [ ] **Step 3: Apply the same hoist in cell 44**

The eval cell ends with the same nested-f-string pattern. Replace:

```python
        print(f"Check out your results here! ({os.path.join(
            logger._tracking_uri,'#', 'experiments', logger.experiment_id, 'runs', logger.run_id
        )}")
```

with:

```python
        run_url = os.path.join(
            logger._tracking_uri, "#", "experiments", logger.experiment_id, "runs", logger.run_id
        )
        print(f"Check out your results here! ({run_url})")
```

- [ ] **Step 4: Verify no internal references and no nested f-strings remain**

Run:
```bash
grep -n "enigmatic.stanford.edu\|mlflow-runner\|Ask someone from the team" notebooks/inference.ipynb
```
Expected: no output.

```bash
python3 -c "
import ast, json
nb=json.load(open('notebooks/inference.ipynb'))
for i,c in enumerate(nb['cells']):
    if c['cell_type']!='code': continue
    src=''.join(c['source'])
    try: ast.parse(src)
    except SyntaxError as e: print(f'cell {i}: {e}')
print('syntax check done')
"
```
Expected: `syntax check done` with no cell errors.

- [ ] **Step 5: Commit**

```bash
git add notebooks/inference.ipynb
git commit -m "fix(notebook): make MLflow logging opt-in and remove internal server details

ENABLE_LOGGING defaulted to True against an internal tracking server, and
tracking_uri is \${oc.env:MLFLOW_TRACKING_URI} with no default, so a
collaborator without that env var hit an OmegaConf resolution error from an
optional cell. Default to off, guard on the env var, and rewrite the setup
docs without the internal hostname or password handoff.

Also hoists two multi-line f-string expressions into locals, fixing an
unbalanced parenthesis in one of the printed messages.

Refs MOD-1556"
```

---

## Task 3: Correct the data location and link the README (MOD-1553)

**Files:**
- Modify: `notebooks/inference.ipynb` cell 15 (md), new md cell before the dataloader section
- Modify: `README.md` (anchor only, if needed)

- [ ] **Step 1: Delete the `/data/mouse` note from cell 15**

Remove this line verbatim:

```markdown
**Note:** In the container, available datasets on your device should be found in the mounted directory `/data/mouse`.
```

- [ ] **Step 2: Insert a new markdown cell immediately before cell 14 ("Choose dataset(s) and Build Dataloaders")**

```markdown
#### Before you run this section: download the data

This notebook reads sessions from `cfg.paths.data_dir`, which resolves to
`${PROJECT_ROOT}/data` — the same directory the README's download helper writes to.

If you have not downloaded any sessions yet, follow
[Data Preparation](../README.md#data-preparation) in the main README first. In short:

1. Request access to the [gated dataset](https://huggingface.co/datasets/the-enigma-project/omnimouse-dataset) on the 🤗 Hub
2. `export HF_TOKEN=<your-token>`
3. Fetch the helper and download one session:

```bash
huggingface-cli download the-enigma-project/omnimouse-dataset download_dataset.py \
  --repo-type dataset --local-dir .
python download_dataset.py --experiments dynamic17797-4-7-Video-021a75e56847d574b9acbcc06c675055_30hz
```

That session is the default `dataset_collection` below, so the two steps line up.
```

- [ ] **Step 3: Add the resolved-path print to cell 16**

At the top of cell 16, before `dataset_collection` is defined:

```python
# Where this notebook looks for sessions. Set by `paths.data_dir` in
# configs/paths/default.yaml, which derives from the PROJECT_ROOT env var.
data_root = Path(cfg.paths.data_dir)
print(f"Reading sessions from: {data_root}")
```

and change the `setup_dataloaders_single_rank` call to pass `data_root=data_root`
instead of `data_root=cfg.paths.data_dir` (same value, single source of truth).

- [ ] **Step 4: Confirm the README anchor resolves**

Run:
```bash
grep -n "^## Data Preparation" README.md
```
Expected: one match. GitHub renders that as `#data-preparation`, which the link above uses.

- [ ] **Step 5: Verify `/data/mouse` is gone**

Run:
```bash
grep -n "data/mouse" notebooks/inference.ipynb
```
Expected: no output.

- [ ] **Step 6: Commit**

```bash
git add notebooks/inference.ipynb
git commit -m "docs(notebook): correct data location and link the README download step

The notebook claimed sessions live in /data/mouse, which is neither what the
code reads (cfg.paths.data_dir) nor what the README writes
(\${PROJECT_ROOT}/data), and is not mounted by docker-compose.yml. Replace it
with the resolved data_dir, printed at runtime, and add a prerequisites cell
linking the README's Data Preparation section.

Refs MOD-1553"
```

---

## Task 4: Default to a downloadable session, with a preflight check (MOD-1554)

**Files:**
- Modify: `notebooks/inference.ipynb` cell 16

- [ ] **Step 1: Replace the hardcoded `dataset_collection`**

Old:

```python
dataset_collection = [
    'dynamic29156-11-10-Video-021a75e56847d574b9acbcc06c675055_30hz',
    'dynamic29513-3-5-Video-021a75e56847d574b9acbcc06c675055_30hz',
]
```

New:

```python
# Default: the single session named in the README's `download_dataset.py --experiments`
# example, so README -> notebook works as a literal copy-paste sequence.
# Add more session directory names here once you have downloaded them.
dataset_collection = [
    "dynamic17797-4-7-Video-021a75e56847d574b9acbcc06c675055_30hz",
]
```

- [ ] **Step 2: Add the preflight existence check, directly after `dataset_collection`**

```python
# Fail early with an actionable message rather than deep inside experanto.
missing = [name for name in dataset_collection if not (data_root / name).is_dir()]
if missing:
    raise FileNotFoundError(
        f"{len(missing)} session(s) not found under {data_root}:\n"
        + "\n".join(f"  - {name}" for name in missing)
        + "\n\nDownload them from the repo root with:\n"
        + "  python download_dataset.py --experiments " + " ".join(missing)
        + "\n\nSee the Data Preparation section of the README for HF_TOKEN and access setup."
    )
```

- [ ] **Step 3: Verify the error path renders correctly**

Run:
```bash
python3 - <<'PY'
from pathlib import Path
data_root = Path("/tmp/nonexistent-omnimouse-data")
dataset_collection = ["dynamic17797-4-7-Video-021a75e56847d574b9acbcc06c675055_30hz"]
missing = [n for n in dataset_collection if not (data_root / n).is_dir()]
if missing:
    print(
        f"{len(missing)} session(s) not found under {data_root}:\n"
        + "\n".join(f"  - {n}" for n in missing)
        + "\n\nDownload them from the repo root with:\n"
        + "  python download_dataset.py --experiments " + " ".join(missing)
        + "\n\nSee the Data Preparation section of the README for HF_TOKEN and access setup."
    )
PY
```
Expected: a readable multi-line message naming the session and the exact download command.

- [ ] **Step 4: Confirm the README example uses the same session ID**

Run:
```bash
grep -n "dynamic17797-4-7" README.md notebooks/inference.ipynb
```
Expected: at least one hit in each file, same session ID.

- [ ] **Step 5: Commit**

```bash
git add notebooks/inference.ipynb
git commit -m "fix(notebook): default to the session the README tells you to download

The notebook hardcoded two sessions that the README never instructs anyone to
fetch; the documented smoke test (--num-files 4) pulls random sessions, so the
default almost always failed deep inside the dataloader. Default to the
README's named --experiments example and add a preflight check that names the
missing sessions and the command that downloads them.

Refs MOD-1554"
```

---

## Task 5: Prune non-existent collection names (MOD-1555)

**Files:**
- Modify: `notebooks/inference.ipynb` cell 15 (md)
- Modify: `omnimouse/distributed/utils.py:112-134`

- [ ] **Step 1: Replace the collection menu in cell 15**

Old:

```markdown
- **Small-scale**: `"single_demo"`, `"quick_2"`, `"public_5"`, `"initial_32"`
- **Large-scale**: `"pretraining_316"`, `"colossus"`, `"foundation_32"`
```

New:

```markdown
Predefined collections live in [`omnimouse/experiments/scaling_laws.py`](../omnimouse/experiments/scaling_laws.py).
The ones available in this repo are all **large** — they assume the full 323-session corpus:

| Name | Sessions | Notes |
| :--- | :------: | :---- |
| `validation_5` | 5 | Smallest predefined set; still needs those 5 sessions downloaded |
| `sensorium_2023_10` | 10 | SENSORIUM 2023 test set + `validation_5` |
| `pretraining_316` | 316 | Full pretraining corpus |
| `colossus` | 323 | `pretraining_316` + validation sets |

For a smoke test, do **not** use a predefined name — pass an explicit list of the session
directories you actually downloaded (Option 2 below). That is what the default cell does.
```

Verify those four names before writing them:
```bash
grep -nE "^(validation_5|sensorium_2023_10|pretraining_316|colossus) *=" omnimouse/experiments/scaling_laws.py
```
Expected: four matches. If any is missing, drop it from the table.

- [ ] **Step 2: Fix the `sandbox.py` reference in the same cell**

Replace the sentence attributing collections to `sandbox.py` or `scaling_laws.py` with a
reference to `scaling_laws.py` only. `omnimouse/experiments/sandbox.py` does not exist in the
public repo.

- [ ] **Step 3: Silence the spurious import warning in `get_dataset_list`**

`get_dataset_list` already survives the missing module — it catches `ImportError` and
continues — but it prints `Could not import module 'omnimouse.experiments.sandbox'.` on every
call, which reads like an error. Make the optional modules genuinely optional.

In `omnimouse/distributed/utils.py`, replace:

```python
        except ImportError:
            print(f"Could not import module '{module_name}'.")
```

with:

```python
        except ImportError:
            # Not every module in the list ships in every distribution of the repo
            # (e.g. `sandbox` is internal-only), so a missing module is not an error.
            log.debug(f"Optional dataset-collection module '{module_name}' not present.")
```

Confirm `log` is already defined in that module; if not, use `logging.getLogger(__name__).debug`.

Run:
```bash
grep -n "^log\|RankedLogger\|getLogger" omnimouse/distributed/utils.py | head
```

- [ ] **Step 4: Verify behaviour is unchanged for a real name and a bad name**

Run:
```bash
python3 -c "
import rootutils, pathlib
rootutils.setup_root(pathlib.Path.cwd(), indicator='.project-root', pythonpath=True, dotenv=True)
from omnimouse.distributed.utils import get_dataset_list
print('validation_5 ->', len(get_dataset_list('validation_5')), 'sessions')
try:
    get_dataset_list('public_5')
except AttributeError as e:
    print('public_5 correctly raises AttributeError')
"
```
Expected: a session count for `validation_5`, an `AttributeError` for `public_5`, and **no**
`Could not import module` line. (Needs deps installed; defer to the GPU agent if not.)

- [ ] **Step 5: Commit**

```bash
git add notebooks/inference.ipynb omnimouse/distributed/utils.py
git commit -m "fix(notebook): list only dataset collections that exist in the public repo

Cell 15 advertised single_demo/quick_2/public_5/initial_32/foundation_32, none
of which are defined here, and attributed them to a sandbox.py that did not
ship. Replace with the collections that actually resolve, annotated with
session counts, and steer smoke tests toward an explicit session list.

Also downgrades the missing-optional-module print in get_dataset_list to a
debug log so the absent sandbox module stops looking like an error.

Refs MOD-1555"
```

---

## Task 6: Fix container config references (MOD-1557)

**Files:**
- Modify: `docker-compose.yml:7`
- Modify: `.devcontainer/devcontainer.json:3`

Confirmed by inspection: the shipped `Dockerfile` declares `ARG USER_ID/GROUP_ID/USERNAME`
and uses `/src/omnimouse`, exactly matching what `docker-compose.yml` passes. The shipped
files are the intended public versions — only the *references* are stale. Do not rename files.

- [ ] **Step 1: Point compose at the real Dockerfile**

In `docker-compose.yml`, change `dockerfile: Dockerfile.new` to `dockerfile: Dockerfile`.

- [ ] **Step 2: Point devcontainer at the real compose file**

In `.devcontainer/devcontainer.json`, change
`"dockerComposeFile": "../docker-compose.new.yaml"` to
`"dockerComposeFile": "../docker-compose.yml"`.

- [ ] **Step 3: Verify every referenced file exists**

Run:
```bash
grep -n "dockerfile:\|dockerComposeFile" docker-compose.yml .devcontainer/devcontainer.json
test -f Dockerfile && echo "Dockerfile OK"
test -f docker-compose.yml && echo "docker-compose.yml OK"
grep -rn "\.new\.\|Dockerfile\.new" docker-compose.yml .devcontainer/devcontainer.json || echo "no stale .new references"
```
Expected: both files exist, no `.new` references remain.

- [ ] **Step 4: Verify the compose file parses and resolves the build context**

Run:
```bash
USER_ID=1000 GROUP_ID=1000 USERNAME=test docker compose config >/dev/null && echo "compose config OK"
```
Expected: `compose config OK`. If `docker` is unavailable on this machine, validate YAML only:
```bash
python3 -c "import yaml;yaml.safe_load(open('docker-compose.yml'));print('yaml OK')"
```
and defer the real check to the GPU agent.

- [ ] **Step 5: Commit**

```bash
git add docker-compose.yml .devcontainer/devcontainer.json
git commit -m "fix(container): reference the Dockerfile and compose file that actually ship

devcontainer.json pointed at docker-compose.new.yaml and docker-compose.yml
pointed at Dockerfile.new; neither exists in the public repo, so 'Reopen in
Container' failed on a fresh clone. The shipped Dockerfile already accepts the
ARGs compose passes, so only the references were stale.

Refs MOD-1557"
```

---

## Task 7: Fix the broken README install link (MOD-1558)

**Files:**
- Modify: `README.md` (Installation section)

Depends on Task 6 — the install notes must describe the corrected filenames.

- [ ] **Step 1: Inventory every relative link in the README**

Run:
```bash
grep -oE '\]\(([^)h][^)]*)\)' README.md | sed 's/^](//;s/)$//' | sort -u | while read -r p; do
  [ -e "${p%%#*}" ] && echo "OK   $p" || echo "DEAD $p"
done
```
Expected: exactly one `DEAD` entry — `docs/INSTALL.md`. If others appear, fix them in this
task too and note them in the commit body.

- [ ] **Step 2: Replace the dead link with inline instructions**

In the Installation section, replace:

```markdown
For containerised environments (Docker, Apptainer/Singularity on HPC), see [`docs/INSTALL.md`](docs/INSTALL.md).
```

with:

```markdown
### Containers

A devcontainer is provided for local GPU workstations. Copy `.env.example` to `.env`, add your
user IDs, then "Reopen in Container" in VS Code:

```bash
cp .env.example .env
echo -e "USER_ID=$(id -u)\nGROUP_ID=$(id -g)\nUSERNAME=$(whoami)" >> .env
```

The same image can be built directly for HPC use:

```bash
docker compose build omnimouse-dev
```

For Apptainer/Singularity on clusters without Docker, convert the built image:

```bash
apptainer build omnimouse.sif docker-daemon://${USER}-omnimouse:latest
```
```

This is preferred over writing a new `docs/INSTALL.md`: the README already carries every other
setup instruction, and a separate file is one more thing to drift.

- [ ] **Step 3: Re-run the link check**

Run the Step 1 command again.
Expected: no `DEAD` entries.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: replace dead docs/INSTALL.md link with inline container instructions

The README pointed containerised/HPC users at docs/INSTALL.md, but there is no
docs/ directory in the repo, so the link 404'd for exactly the audience most
likely to need it.

Refs MOD-1558"
```

---

## Task 8: Whole-notebook consistency pass

**Files:**
- Modify: `notebooks/inference.ipynb` (as needed)

- [ ] **Step 1: Confirm the notebook is valid and every code cell parses**

```bash
python3 -c "
import ast, json
nb=json.load(open('notebooks/inference.ipynb'))
assert nb['nbformat']==4
bad=0
for i,c in enumerate(nb['cells']):
    if c['cell_type']!='code': continue
    try: ast.parse(''.join(c['source']))
    except SyntaxError as e: bad+=1; print(f'cell {i}: {e}')
print('cells:', len(nb['cells']), '| syntax errors:', bad)
"
```
Expected: `syntax errors: 0`.

- [ ] **Step 2: Confirm no internal or stale references survive anywhere in the diff**

```bash
grep -rniE "enigmatic\.stanford|mlflow-runner|/data/mouse|src/omnimouse|Dockerfile\.new|docker-compose\.new|single_demo|quick_2|public_5|initial_32" \
  notebooks/inference.ipynb README.md docker-compose.yml .devcontainer/devcontainer.json \
  || echo "clean"
```
Expected: `clean`. (`/src/omnimouse` legitimately remains inside `Dockerfile` and
`docker-compose.yml`'s volume mount — those files are not in this grep set by design.)

- [ ] **Step 3: Strip execution outputs and counts so the diff is reviewable**

```bash
python3 - <<'PY'
import json
p='notebooks/inference.ipynb'
nb=json.load(open(p))
for c in nb['cells']:
    if c['cell_type']=='code':
        c['outputs']=[]; c['execution_count']=None
json.dump(nb, open(p,'w'), indent=1, ensure_ascii=False)
open(p,'a').write('\n')
PY
git diff --stat
```

- [ ] **Step 4: Commit**

```bash
git add notebooks/inference.ipynb
git commit -m "chore(notebook): clear execution outputs for a reviewable diff

Refs MOD-1552"
```

---

## Handoff Verification Checklist (for the GPU + data agent)

Nothing below was verified on the authoring machine — it has no dataset and no CUDA.
Run these in order on a machine with GPU and Hub access:

- [ ] `uv sync` from a clean clone of the branch succeeds
- [ ] Request dataset access, `export HF_TOKEN=...`, then run the README's
      `download_dataset.py --experiments dynamic17797-4-7-...` command from the repo root
- [ ] Start the kernel **from the repo root** and run the notebook top to bottom with zero
      edits → completes and prints eval metrics
- [ ] Repeat with the kernel started **from `notebooks/`** → identical result (proves the
      cwd-independence fix in Task 1)
- [ ] Run once with `ENABLE_LOGGING = True` and `MLFLOW_TRACKING_URI` unset → prints the skip
      message, does **not** raise
- [ ] Run once with `ENABLE_LOGGING = True` and a valid `MLFLOW_TRACKING_URI` → run appears in
      MLflow
- [ ] Delete the downloaded session and re-run cell 16 → preflight raises the actionable
      `FileNotFoundError`, not an experanto traceback
- [ ] "Reopen in Container" on a fresh clone builds and starts (Task 6)
- [ ] Confirm the two previously-hardcoded sessions
      (`dynamic29156-11-10-...`, `dynamic29513-3-5-...`) are present in the public Hub dataset.
      If they are, consider restoring them as a documented multi-session example.

## Open Question for the Author

`assets/` ships interactive HTML visualizers. Not audited in this pass — if any embed the
internal MLflow hostname or other internal URLs, that is a follow-up issue.
