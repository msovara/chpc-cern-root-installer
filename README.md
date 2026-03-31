# CERN ROOT (conda-forge) on CHPC / Lengau

This guide installs **[ROOT](https://root.cern/)** via **Conda** using the **`conda-forge`** package. It matches what works on CHPC when the shared Anaconda package cache is unhealthy (see **Troubleshooting**).

Official CHPC references:

- [howto:root](https://wiki.chpc.ac.za/howto:root)
- [guide:python](https://wiki.chpc.ac.za/guide:python?s[]=*python*)

---

## Where to run this

You can use a **login node**, **interactive node** (e.g. `chpclic1`), or **DTN**—anywhere you have loaded the CHPC **Anaconda** module and have network access for Conda downloads. You do **not** have to use DTN only; pick a host where `conda` is available after `module load`.

---

## 1. SSH to Lengau

```bash
ssh <username>@lengau.chpc.ac.za
```

---

## 2. Load Anaconda (required if `conda: command not found`)

```bash
module purge
module load chpc/python/anaconda/3-2024.10.1
```

Use the **current** Anaconda module from `module avail anaconda` if the version differs. Optional: `module add chpc/BIOMODULES` if your site workflow requires it.

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
#PBS -l select=1:ncpus=4:mem=8GB
#PBS -q workq

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
| `CondaVerificationError` … `libstdcxx-devel_linux-64` … **corrupted** | Set **`CONDA_PKGS_DIRS`** to `~/.conda/pkgs` (step 3), run `conda clean -a -y`, recreate env. If the error references `/home/apps/chpc/bio/.../pkgs`, ask CHPC to fix that shared cache. |
| `CondaValueError: prefix already exists` | Step 5. |

---

## Support

Questions about this guide: **msovara@csir.co.za**  
Cluster-specific module paths and queues: **CHPC helpdesk** / [wiki](https://wiki.chpc.ac.za/).
