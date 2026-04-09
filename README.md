# CERN ROOT (conda-forge) on CHPC / Lengau

This guide installs **[ROOT](https://root.cern/)** via **Conda** using the **`conda-forge`** package. It matches what works on CHPC when the shared Anaconda package cache is unhealthy (see **Troubleshooting**).

Official CHPC references:

- [howto:root](https://wiki.chpc.ac.za/howto:root)
- [guide:python](https://wiki.chpc.ac.za/guide:python?s[]=*python*)

---

## Where to run this

You should use   **chpclic1** or **DTN**. 

---

## 1. SSH to Lengau

```bash
ssh <username>@lengau.chpc.ac.za
ssh -X chpclic1
```

---

## 2. Load Anaconda (required if `conda: command not found`)

```bash
module purge
module load chpc/python/anaconda/3-2024.10.1
```

Use the **current** Anaconda module from `module avail anaconda` if the version differs. Optional: `module add chpc/BIOMODULES`.

Confirm:

```bash
conda --version
```

---

## 3. Put Conda **envs** and **pkgs** on Lustre (recommended)

**CHPC recommends not storing large Conda environments and package caches under your home directory** (small quota, NFS). Use **Lustre** instead, for example:

```bash
export LUSTRE_CONDA=/mnt/lustre/users/${USER}/conda
mkdir -p "${LUSTRE_CONDA}/pkgs" "${LUSTRE_CONDA}/envs"
```

Register those locations with Conda (once per user; stored in `~/.condarc`):

```bash
conda config --prepend pkgs_dirs "${LUSTRE_CONDA}/pkgs"
conda config --prepend envs_dirs "${LUSTRE_CONDA}/envs"
```

Optional: also set the cache variable in the shell so every tool agrees (matches `pkgs_dirs`):

```bash
export CONDA_PKGS_DIRS="${LUSTRE_CONDA}/pkgs"
```

To make **`CONDA_PKGS_DIRS`** persistent:

```bash
echo 'export LUSTRE_CONDA=/mnt/lustre/users/${USER}/conda' >> ~/.bashrc
echo 'export CONDA_PKGS_DIRS="${LUSTRE_CONDA}/pkgs"' >> ~/.bashrc
```

After this, **`conda create -n root_env ...`** creates **`root_env` under** `${LUSTRE_CONDA}/envs/root_env`, not under `~/.conda/envs`.

**Why:** The site install under `/home/apps/chpc/bio/.../pkgs` can hold a **corrupted** copy of packages; a **personal** `pkgs` tree avoids that. Keeping **both** `pkgs` and `envs` on Lustre avoids filling home and matches CHPC guidance for large software trees.

---

## 4. Create the environment (recommended command)

Use **conda-forge only** for this environment and pin **Python** to keep the solve smaller and more predictable:

```bash
export LUSTRE_CONDA=/mnt/lustre/users/${USER}/conda
export CONDA_PKGS_DIRS="${LUSTRE_CONDA}/pkgs"
conda create -n root_env -c conda-forge --override-channels root python=3.12
```

- Approve with **`y`** when prompted.
- The **solver** step can take **several minutes**; that is normal. For faster solves, install **`conda-libmamba-solver`** (or **mamba**) in `base` if allowed—see [conda-forge ROOT](https://anaconda.org/conda-forge/root).

**Minimal alternative** (wiki-style):

```bash
conda create -n root_env root -c conda-forge
```

---

## 5. If `conda create` fails: “prefix already exists”

Remove the old environment, then retry:

```bash
conda env remove -n root_env -y
```

If the directory still exists (adjust if you used a different `LUSTRE_CONDA`):

```bash
rm -rf /mnt/lustre/users/${USER}/conda/envs/root_env
```

---

## 6. Verify

```bash
conda activate root_env
conda info --envs
conda list | grep -E '^root\s'
root --version
python -c "import ROOT; print(ROOT.gROOT.GetVersion())"
```

You should see **`root_env`** pointing at a path under **`/mnt/lustre/users/.../conda/envs/`**.

---

## 7. Daily use

```bash
module load chpc/python/anaconda/3-2024.10.1
conda activate root_env
root
```

When finished:

```bash
conda deactivate
```

---

## 8. PBS batch jobs (example)

```bash
#!/bin/bash
#PBS -N root_job
#PBS -l select=1:ncpus=24
#PBS -q smp

module load chpc/python/anaconda/3-2024.10.1
export LUSTRE_CONDA=/mnt/lustre/users/${USER}/conda
export CONDA_PKGS_DIRS="${LUSTRE_CONDA}/pkgs"
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate root_env

root -b -q your_macro.C
```

Adjust `module load` and queue/resources to your CHPC policy.

---

## Troubleshooting

| Symptom | What to try |
|--------|-------------|
| `conda: command not found` | Load the Anaconda module (step 2). |
| Hangs at **Solving environment** | Wait 15–30+ minutes, or use **libmamba** / **mamba**, or pin `python` + smaller channel set (`--override-channels`). |
| `CondaVerificationError` … **corrupted** (any package name) | The extracted files under `pkgs/<package>` do not match the manifest—usually **incomplete download/extract** or **full disk**. See **Complete reset** below. |
| Errors under `/home/apps/chpc/bio/.../pkgs` | Shared site cache; use **Lustre `pkgs_dirs`** (step 3) and/or ask CHPC to repair that directory. |
| Errors under **`.../conda/pkgs/`** on Lustre (e.g. `jedi`, `libstdcxx-devel`) | Your **personal** cache is corrupt or partial—**delete that package folder** or wipe `${LUSTRE_CONDA}/pkgs` (see **Complete reset**). |
| `(root_env)` in prompt but `EnvironmentLocationNotFound` | The env **never finished installing**. Run `conda deactivate` until prompt shows `(base)`, remove `${LUSTRE_CONDA}/envs/root_env`, then recreate. |
| `CondaValueError: prefix already exists` | Step 5. |

### Complete reset (when verification keeps failing)

Corruption in the **pkgs** cache is common after interrupted installs, flaky I/O, or **quota full**. Check **Lustre** space (and home if you still use it for anything):

```bash
df -h /mnt/lustre/users/${USER}
df -h $HOME
quota -s 2>/dev/null || true
```

Then reset caches and the broken env (paths match step 3):

```bash
export LUSTRE_CONDA=/mnt/lustre/users/${USER}/conda

module purge
module load chpc/python/anaconda/3-2024.10.1

conda deactivate 2>/dev/null || true
conda deactivate 2>/dev/null || true

rm -rf "${LUSTRE_CONDA}/envs/root_env"
rm -rf "${LUSTRE_CONDA}/pkgs"/*
conda clean -a -y

mkdir -p "${LUSTRE_CONDA}/pkgs" "${LUSTRE_CONDA}/envs"
export CONDA_PKGS_DIRS="${LUSTRE_CONDA}/pkgs"

conda create -n root_env -c conda-forge --override-channels root python=3.12
```

If it still fails, remove **only** the package named in the error, e.g.:

```bash
rm -rf "${LUSTRE_CONDA}/pkgs/jedi-*"
```

…then run `conda create` again.

### Faster solver (optional, reduces retries)

If CHPC allows installing into **base**:

```bash
conda install -n base conda-libmamba-solver -c conda-forge
conda config --set solver libmamba
```

Or use **mamba** (`conda install mamba -c conda-forge`) and run `mamba create -n root_env ...` instead of `conda create`.

### Last resort: private Miniconda on Lustre

Install [Miniconda](https://docs.conda.io/en/latest/miniconda.html) under e.g. **`/mnt/lustre/users/${USER}/miniconda3`** (not under `$HOME` if you are avoiding large trees in home). Repeat steps 3–4 with **that** `conda` binary, and still set **`envs_dirs` / `pkgs_dirs`** to Lustre paths as above.

---

## Support

Questions about this guide: **msovara@csir.co.za**  
Cluster-specific module paths and queues: **CHPC helpdesk** / [wiki](https://wiki.chpc.ac.za/).
