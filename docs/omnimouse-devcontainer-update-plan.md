# OmniMouse Devcontainer Repair + OmniMonkey Parity Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `sensei:subagent-driven-development` or `sensei:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the OmniMouse devcontainer build and start again, then port the container-setup improvements that OmniMonkey has accumulated — *without* importing OmniMonkey-specific infrastructure that OmniMouse has no use for and that would break on a generic host.

**Scope:** `.devcontainer/devcontainer.json`, `Dockerfile`, `docker-compose.yml`, `.env.example`, new `.dockerignore`. No Python source changes.

**Reference:** OmniMonkey worktree `/home/agilbert/dev/omnimonkey/.worktrees/hiera-sweep` (branch `hiera-sweep`). Its `.git` points at the in-container path `/src/omnimonkey/.git`, so git history is not readable from the host — this plan compares **working-tree file contents only**.

**Worktree:** `.worktrees/omnimouse-handoff-cleanup` on branch `alex/omnimouse-handoff-cleanup`, tracking `origin/alex/omnimouse-handoff-cleanup` (fork `alexrgilbert/omnimouse`).

**Status of this document:** untracked scratch plan. Do not commit it.

---

## Root cause of the build failure (reproduced) — ALREADY FIXED ON THIS BRANCH

The committed config at `main` (`a4a5c869`) is self-inconsistent: it references two files that **have never existed in the repo**. `git log --name-status -- Dockerfile.new docker-compose.new.yaml` returns nothing; `Initial commit` (`1903de72`) added only `Dockerfile`, `docker-compose.yml`, and `.devcontainer/devcontainer.json`.

| Reference site | Pointed at | Exists? |
|---|---|---|
| `.devcontainer/devcontainer.json:3` `"dockerComposeFile"` | `../docker-compose.new.yaml` | **No** |
| `docker-compose.yml:7` `dockerfile:` | `Dockerfile.new` | **No** |

Verified failure, still reproducible **on `main`**:

```
$ USER_ID=$(id -u) GROUP_ID=$(id -g) USERNAME=$(whoami) docker compose build
#1 [omnimouse-dev internal] load build definition from Dockerfile.new
failed to solve: failed to read dockerfile: open Dockerfile.new: no such file or directory
```

VS Code fails even earlier than this, on the missing `docker-compose.new.yaml`.

> **This branch already carries the fix**, in `578cf4cb fix(container): reference the Dockerfile and compose file that actually ship` (Refs MOD-1557). See Task 1.

**A second blocker sits behind the first.** `.env` is gitignored and does not exist in a fresh clone or a fresh worktree. Compose auto-loads `.env` for `USER_ID` / `GROUP_ID` / `USERNAME`, and [Dockerfile:74-78](Dockerfile#L74-L78) hard-fails when they are empty. Task 3 covers it; the README half is already done (see below).

---

## Task 1: Repair the dangling filename references (P0 — blocker) — ✅ DONE

Completed on this branch by `578cf4cb`. Verified in the worktree:

```
$ grep -n 'dockerComposeFile' .devcontainer/devcontainer.json
3:  "dockerComposeFile": "../docker-compose.yml",
$ grep -n 'dockerfile:' docker-compose.yml
7:      dockerfile: Dockerfile
```

- [x] `devcontainer.json` → `../docker-compose.yml`
- [x] `docker-compose.yml` → `dockerfile: Dockerfile`
- [x] `docker compose config` exits 0
- [x] `docker compose build` gets past `load build definition` and reaches `stage-0 7/13` (the user-creation step). The `nvcr.io/nvidia/pytorch:25.05-py3` base image is already cached on this host, so this was fast.

**No action required.** Left in the plan as the anchor for the other tasks.

> **Do not** "fix" anything here by creating `Dockerfile.new` / `docker-compose.new.yaml`. The `.new` names were a leftover from an unfinished editing session; the canonical files are the ones under version control.

---

## Task 2: Add a `.dockerignore` (P0 — correctness + a 100× context win)

OmniMouse has no `.dockerignore`. OmniMonkey does. On-disk sizes in the main checkout:

| Path | Size |
|---|---|
| `.git` | **410 MB** |
| `assets` | 3.1 MB |
| everything else | < 1.1 MB |
| **total** | **415 MB** |

**Important caveat, measured rather than assumed.** A build run *from this worktree* transfers only **1.37 MB** of context, because in a linked worktree `.git` is a 78-byte gitlink *file*, not the 410 MB directory. So the dramatic saving applies to the **main checkout and to any fresh clone** — which is what an external collaborator following the README actually has — not to builds launched from here.

The change is still worth making: it protects the fresh-clone path, and it stops a stray local `.venv`, `data/`, or `*.egg-info` from being copied into the image where the `omnimouse-venv` volume would shadow it at runtime. Just do not expect the before/after numbers to move when you test from the worktree.

- [ ] **Step 1:** Create `.dockerignore` at the repo root with OmniMonkey's contents verbatim — they are project-agnostic and all apply here:

```
**/__pycache__
**/*.pyc
**/.pytest_cache
**/.ruff_cache
**/.mypy_cache
.venv
.git
*.egg-info
```

- [ ] **Step 2:** Consider adding `data/` as well. It is gitignored, empty today, and is where the README's `download_dataset.py` helper lands the dataset (`paths.data_dir` = `${PROJECT_ROOT}/data`). Once a developer downloads data it becomes many GB of build context for no reason — the runtime bind mount `.:/src/omnimouse` provides it anyway. **This line is an OmniMouse addition, not an OmniMonkey port.**

- [ ] **Step 3:** Verify the context shrank:

```bash
USER_ID=$(id -u) GROUP_ID=$(id -g) USERNAME=$(whoami) docker compose build 2>&1 | grep 'transferring context'
```

Expect a few MB, not ~415 MB.

---

## Task 3: Make the missing-`.env` failure discoverable (P0 — blocker #2)

The Dockerfile guard already prints a good message, but a devcontainer user never sees build stdout clearly, and nothing in the README points at it.

- [ ] **Step 1:** Confirm the guard's failure mode on a clean checkout:

```bash
env -u USER_ID -u GROUP_ID -u USERNAME docker compose build 2>&1 | tail -20
```

- [x] **Step 2: already done on this branch.** `fb01724b` added a "Containers" subsection to the README carrying exactly this pre-step:

```bash
cp .env.example .env
echo -e "USER_ID=$(id -u)\nGROUP_ID=$(id -g)\nUSERNAME=$(whoami)" >> .env
```

  Nothing further needed here. Note this also means **Task 7 is done** — same commit replaced the dead `docs/INSTALL.md` link.

- [ ] **Step 3:** Do **not** add defaults like `${USER_ID:-1000}` in `docker-compose.yml`. A silent wrong UID produces root-owned files in the bind-mounted worktree on the host, which is worse and much harder to diagnose than the current fail-fast. Keep the guard.

---

## Task 4: Port the VS Code Python-env activation fix (P1 — the highest-value parity item)

This is the single most valuable thing OmniMonkey has that OmniMouse lacks. It is pure `devcontainer.json` config, has no infrastructure dependency, and directly affects anyone running Claude Code in an integrated terminal.

The problem it solves: `python-envs.terminal.autoActivationType` defaults to `"command"`, which types `source .venv/bin/activate` into **already-open** integrated terminals on every reconnect — landing as junk keystrokes in whatever is running there.

- [ ] **Step 1:** In [.devcontainer/devcontainer.json](.devcontainer/devcontainer.json), add `"ms-python.vscode-python-envs"` to `extensions`, immediately after `"ms-python.debugpy"`.

- [ ] **Step 2:** Add `"python-envs.terminal.autoActivationType": "shellStartup"` to `settings`, next to `python.terminal.activateEnvironment`.

- [ ] **Step 3:** Port OmniMonkey's comments verbatim. They encode a non-obvious trap — `python.terminal.activateEnvironment` must stay `true`; setting it `false` makes python-envs map it to `off`, disabling activation entirely **and persisting `off` into the user's own config**. Without the comment someone will "clean up" the apparently redundant setting.

- [ ] **Step 4:** Verify by rebuilding the container, opening a terminal, running something interactive, then reloading the window — no `source .venv/bin/activate` text should appear in the running program.

---

## Task 5: Port the stable ssh-agent socket path (P1)

**Current OmniMouse:** [docker-compose.yml](docker-compose.yml) mounts `$SSH_AUTH_SOCK:/ssh-agent` directly.

**Problem:** the devcontainer records bind mounts at *creation* time. `$SSH_AUTH_SOCK` is a per-SSH-session path (e.g. `/tmp/ssh-XXXXXX6yTMCK/agent.6790` — that is the live value on this host). When that SSH session ends, the path dies and the container refuses to start with `mount ... /ssh-agent: not a directory`. This is a strong candidate for a *recurring* devcontainer failure independent of the Task 1 blocker.

- [ ] **Step 1:** Change the mount to OmniMonkey's form, keeping the fallback so nothing breaks for people who have not set the new variable:

```yaml
      # Prefer a stable socket path (SSH_AGENT_SOCK) over the per-session
      # $SSH_AUTH_SOCK, which bakes a dead path into the container once the
      # SSH session that created it ends. See .env.example.
      - ${SSH_AGENT_SOCK:-${SSH_AUTH_SOCK}}:/ssh-agent
```

- [ ] **Step 2:** Port the corresponding `.env.example` block, including the host `~/.bashrc` symlink snippet that keeps the stable path pointed at the live agent.

- [ ] **Step 3:** Verify `docker compose config` shows the resolved socket path under both settings:

```bash
SSH_AGENT_SOCK=$HOME/.ssh/agent.sock USER_ID=$(id -u) GROUP_ID=$(id -g) USERNAME=$(whoami) \
  docker compose config | grep -A3 ssh-agent
```

---

## Task 6: Align `.env.example` with what OmniMouse actually reads (P1)

`.env.example` has drifted from the README and has no comments explaining required vs. optional.

- [ ] **Step 1:** Fix the Hugging Face token name. `.env.example` declares `HUGGINGFACE_TOKEN`, but the README tells users to `export HF_TOKEN=...`, and `huggingface_hub` reads `HF_TOKEN`. **`HUGGINGFACE_TOKEN` is dead.** Rename it to `HF_TOKEN`, matching OmniMonkey.

  **This is a live bug, not a hypothetical.** The `.env` actually in use on this host sets `HUGGINGFACE_TOKEN=...` and no `HF_TOKEN` — so gated-dataset downloads are running unauthenticated today. Fix `.env.example` *and* tell the user to update their real `.env`.

- [ ] **Step 2:** Port OmniMonkey's comment style for the `USER_ID` / `GROUP_ID` / `USERNAME` block, which spells out the `echo -e ... >> .env` one-liner.

- [ ] **Step 3:** Note in a comment that `GITHUB_TOKEN` is **not required for the build** in this repo — see Task 8 for why.

---

## Task 7: Fix or remove the dead `docs/INSTALL.md` link (P2) — ✅ DONE

Completed on this branch by `fb01724b docs: replace dead docs/INSTALL.md link with inline container instructions` (Refs MOD-1558). The README now has an inline "Containers" subsection covering the devcontainer pre-step, `docker compose build`, and Apptainer conversion.

That commit's message notes two *other* dead relative links it deliberately left alone — `LICENSE` and `configs/experiment/sensorium_probe/` — both needing an author decision. Out of scope here; they belong to [omnimouse-handoff-cleanup-plan.md](omnimouse-handoff-cleanup-plan.md).

---

## Task 8: Decide on the inert `github_token` secret (P2 — low priority, verified harmless)

`docker-compose.yml` declares a `github_token` secret sourced from `$GITHUB_TOKEN` and lists it under `build.secrets`, but **the OmniMouse Dockerfile never mounts it** — [Dockerfile:116](Dockerfile#L116) is a bare `RUN uv sync`. This is vestigial: it was copied from OmniMonkey, where the secret exists because `spiral-dataloader` and `spine` are private repos.

OmniMouse's only git dependency is `experanto`, which is **public**. No credential is needed.

**Verified:** an unset `GITHUB_TOKEN` does *not* break the build. I reproduced this with a minimal compose project — `docker compose config` resolves and `docker compose build` succeeds with the variable unset. So this is cleanliness, not a bug, and is **not** part of the current failure.

- [ ] **Step 1:** Remove the `secrets:` top-level block and the `build.secrets` entry from `docker-compose.yml`.
- [ ] **Step 2:** Do **not** port OmniMonkey's `--mount=type=secret` credential-helper `RUN` block. It exists solely for private dependencies OmniMouse does not have.

---

## Explicitly DO NOT port — these would break OmniMouse

This section is the "don't break it" half of the request. Each item below is real, load-bearing infrastructure in OmniMonkey and actively harmful in OmniMouse.

| OmniMonkey feature | Why it must not be ported |
|---|---|
| `/data` bind from `${SHARED_DATA_DIR:-/var/shared}` with `create_host_path: false` | OmniMouse reads data from `${PROJECT_ROOT}/data`, already visible through the `.:/src/omnimouse` bind mount. `/var/shared` is a `tm-gpu0`-ism. With `create_host_path: false`, **the container refuses to start on any host lacking that path** — converting a working setup into a hard failure. |
| `/mnt` bind from `${HOST_MNT_DIR:-/mnt}` | Same failure mode. OmniMouse reads nothing from `/mnt`. |
| `SPIRAL_CACHE_DIR`, `SPIRAL_CACHE_CAPACITY_BYTES`, `SPIRAL__FRAGMENTS_CACHE__BLOCK_SIZE` | OmniMouse has **no** Spiral dependency (`pyproject.toml` has no `pyspiral` / `spiral-dataloader`, and there is no `omnimouse/utils/spiral_env.py`). These would be confusing dead env vars. |
| `SPIRAL_CLIENT_ID` / `SPIRAL_CLIENT_SECRET` / `SPIRAL_WORKLOAD_ID` in `.env.example` | Same — no Spiral. |
| The `--mount=type=secret` GitHub-credential `RUN` block | See Task 8. OmniMouse has no private dependencies. |
| `AWS_*` / Stanford Ceph credentials in `.env.example` | OmniMonkey-specific W&B reference-artifact plumbing (MOD-687). |
| `.dstack/` profiles | OmniMonkey cluster infrastructure; out of scope for this repo. |

---

## Known-unverified risk (investigate, do not blind-fix)

**`hydra_plugins/` is not copied into the image.** [Dockerfile:112-113](Dockerfile#L112-L113) copies only `pyproject.toml`, `uv.lock*`, `.python-version*`, and `omnimouse/` before `uv sync`, but `pyproject.toml` declares `include = ["omnimouse*", "hydra_plugins*"]`. `hydra_plugins/resolvers.py` registers a resolver that [configs/paths/default.yaml:16](configs/paths/default.yaml#L16) depends on.

Because `uv sync` installs the project editable and the `.venv` lives in a **named volume** (`omnimouse-venv`) that survives rebuilds, a setuptools editable finder generated while `hydra_plugins/` was absent could fail to map the package even though the bind mount makes the directory present at runtime.

This is **unconfirmed** — it may resolve fine via `sys.path`, since `hydra_plugins` has no `__init__.py` and Hydra discovers it by path. **OmniMonkey has the identical pattern**, so it is parity, not a regression, and if it were broken there someone would likely have noticed.

- [ ] Verify inside a built container: `python -c "import hydra_plugins.resolvers"` and run any config compose that uses the resolver.
- [ ] Only if it fails: add `COPY --chown=${USER_ID}:${GROUP_ID} hydra_plugins/ ./hydra_plugins/` before the `uv sync` line. Note this invalidates the (expensive) `uv sync` layer cache whenever a resolver changes — weigh that against the fix.

---

## Verification checklist

Run in order. Do not claim success on a step you have not executed.

- [ ] `docker compose config >/dev/null` exits 0 with `USER_ID`/`GROUP_ID`/`USERNAME` set.
- [ ] `docker compose build` completes. **Requires a ~20 GB pull of `nvcr.io/nvidia/pytorch:25.05-py3` on a cold cache.**
- [ ] Build context reported by `transferring context` is single-digit MB, not ~415 MB.
- [ ] `env -u USER_ID -u GROUP_ID -u USERNAME docker compose build` fails with the *intended* guard message, not a confusing one.
- [ ] VS Code "Reopen in Container" succeeds and lands in `/src/omnimouse`.
- [ ] Inside the container: `id -u` matches the host UID; a file created in the workspace is host-owned by you, not root.
- [ ] Inside the container: `python -c "import omnimouse"` and `python -c "import hydra_plugins.resolvers"` both succeed.
- [ ] `ssh-add -l` inside the container lists host keys (ssh-agent forwarding intact).
- [ ] Open an integrated terminal, start an interactive program, reload the window — no `source .venv/bin/activate` junk appears (Task 4).
- [ ] `nvidia-smi` works inside the container **on a GPU host**. This machine's GPU status was not checked; skip and hand off if unavailable.

**Not verifiable here:** anything needing a GPU, the dataset, or VS Code itself. Tasks 1–3 and 8 were verified statically/with `docker compose` on this host. Tasks 4–7 are config edits whose effects require a rebuilt container in VS Code.

---

## Suggested ordering

Tasks 1, 3 (step 2), and 7 are **already on this branch** — the devcontainer builds again as of `578cf4cb`. Remaining work:

1. **Tasks 2 + 5** — `.dockerignore` and the stable ssh-agent socket. Both are small, self-contained container fixes; one commit.
2. **Task 4** — the VS Code python-envs activation fix. Separate commit; it is `.devcontainer`-only and the comments carry the reasoning.
3. **Tasks 6 + 8** — `.env.example` alignment (including the live `HF_TOKEN` bug) and removing the inert `github_token` secret.
4. The `hydra_plugins` investigation, only once you can exec into a built container.

Everything above is additive to a branch that already builds, so each step can be verified independently.
