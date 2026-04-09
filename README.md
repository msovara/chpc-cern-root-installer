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

## 3. Use a personal package cache (recommended on CHPC)

The site install under `/home/apps/chpc/bio/...` can keep a **corrupted** copy of compiler packages (e.g. `libstdcxx-devel_linux-64`). That leads to `CondaVerificationError` during **Verifying transaction**. Pointing Conda at your **home** `pkgs` directory avoids that broken cache.

```bash
mkdir -p ~/.conda/pkgs
export CONDA_PKGS_DIRS=$HOME/.conda/pkgs
```

To make this persistent:

```bash
echo 'export CONDA_PKGS_DIRS=$HOME/.conda/pkgs' >> ~/.bashrc
```

---

## 4. Create the environment (recommended command)

Use **conda-forge only** for this environment and pin **Python** to keep the solve smaller and more predictable:

```bash
export CONDA_PKGS_DIRS=$HOME/.conda/pkgs
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

If the directory still exists:

```bash
rm -rf ~/.conda/envs/root_env
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
| Errors under `/home/apps/chpc/bio/.../pkgs` | Shared site cache; use **`CONDA_PKGS_DIRS`** (step 3) and/or ask CHPC to repair that directory. |
| Errors under **`$HOME/.conda/pkgs/`** (e.g. `jedi`, `libstdcxx-devel`) | Your **personal** cache is corrupt or partial—**delete that package folder** or wipe `~/.conda/pkgs` (see **Complete reset**). |
| `(root_env)` in prompt but `EnvironmentLocationNotFound` | The env **never finished installing**. Run `conda deactivate` until prompt shows `(base)`, remove `~/.conda/envs/root_env`, then recreate. |
| `CondaValueError: prefix already exists` | Step 5. |

### Complete reset (when verification keeps failing)

Corruption in **`~/.conda/pkgs`** is common after interrupted installs, NFS glitches, or **home directory quota full**. Check space first:

```bash
df -h $HOME
quota -s 2>/dev/null || true
```

Then reset caches and the broken env in one clean sequence:

```bash
module purge
module load chpc/python/anaconda/3-2024.10.1

conda deactivate 2>/dev/null || true
conda deactivate 2>/dev/null || true

rm -rf ~/.conda/envs/root_env
rm -rf ~/.conda/pkgs/*
conda clean -a -y

mkdir -p ~/.conda/pkgs
export CONDA_PKGS_DIRS=$HOME/.conda/pkgs

conda create -n root_env -c conda-forge --override-channels root python=3.12
```

If it still fails, remove **only** the package conda names in the error (example for `jedi`):

```bash
rm -rf ~/.conda/pkgs/jedi-*
```

…then run `conda create` again.

### Faster solver (optional, reduces retries)

If CHPC allows installing into **base**:

```bash
conda install -n base conda-libmamba-solver -c conda-forge
conda config --set solver libmamba
```

Or use **mamba** (`conda install mamba -c conda-forge`) and run `mamba create -n root_env ...` instead of `conda create`.

### Last resort: Miniconda in `$HOME`

A private Miniconda under `$HOME/miniconda3` avoids sharing `base` and sometimes problematic site configuration. Install from [Miniconda](https://docs.conda.io/en/latest/miniconda.html), then repeat steps 3–4 using **that** `conda` only.

---

## Support

Questions about this guide: **msovara@csir.co.za**  
Cluster-specific module paths and queues: **CHPC helpdesk** / [wiki](https://wiki.chpc.ac.za/).
