# HPC, Linux, and Coding Manual

> A practical record for daily work on GitHub, Linux, and high-performance computing (HPC) systems.
>
> Last updated: 2026-09-15

## 1. Purpose

This manual records commands, errors, causes, fixes, and verification steps encountered during scientific computing. It is written as a living document: when a new problem is solved, add the solution while the details are still fresh.

The main rule is:

> Do not record only the successful command. Also record where it was run, which environment was active, what failed, why it failed, and how the fix was verified.

## 2. Understand the different layers

Many confusing problems come from mixing up the roles of GitHub, Codespaces, Linux, HPC, software environments, and scientific programs.

| Layer | What it is | What it stores or controls |
|---|---|---|
| GitHub repository | Remote version-controlled storage | Files that have been committed and pushed |
| GitHub Codespace | A remote Linux computer connected to a repository | Saved local files, installed software, terminal sessions, and Git changes |
| HPC login node | Entry point to the cluster | File management, editing, and job submission |
| HPC compute node | Machine assigned by Slurm | CPU- or GPU-heavy calculations |
| Conda/Micromamba environment | Isolated software installation | Python and package versions for one workflow |
| Apptainer container | Portable software image | A fixed software stack in a `.sif` file |
| Scientific software | The program that performs the calculation | GNINA, AutoDock Vina, GROMACS, AlphaFold, and related tools |
| Code or script | Instructions given to the software | Python, Bash, R, configuration files, and Slurm scripts |

These layers are related, but they are not the same thing. Learning Python does not automatically teach Slurm, Linux permissions, Conda dependency management, or container usage.

## 3. GitHub, Git, and Codespaces

### 3.1 Saving is not the same as pushing

The complete path from an edited file to the GitHub repository is:

```text
Edit file -> Save -> Stage -> Commit -> Push -> Verify on github.com
```

- **Save** writes the file to the current computer or Codespace.
- **Stage** selects changes for the next commit.
- **Commit** creates a local Git record.
- **Push** sends local commits to the GitHub repository.
- **Verify** confirms that the correct branch on GitHub contains the files.

A file can survive closing and reopening a Codespace without appearing on the GitHub repository page. This means the file was saved to the Codespace disk but was not necessarily committed and pushed.

### 3.2 Safe end-of-session Git workflow

Run these commands from the repository directory:

```bash
git status
git branch --show-current
git diff
```

Review the listed files. Do not commit passwords, access tokens, SSH keys, `.env` files, confidential data, or very large scientific outputs.

Stage the intended files:

```bash
git add README.md
git add scripts/example.py
```

To stage all changes only after reviewing `git status`:

```bash
git add .
```

Inspect the staged changes:

```bash
git diff --cached
```

Commit and push:

```bash
git commit -m "Add HPC troubleshooting notes"
git push -u origin HEAD
```

Then open the repository on GitHub, select the same branch shown by `git branch --show-current`, and confirm that the files are visible.

### 3.3 Useful Git commands

| Goal | Command |
|---|---|
| Check changed and untracked files | `git status` |
| Show current branch | `git branch --show-current` |
| Show local and remote branches | `git branch -a` |
| Show unstaged changes | `git diff` |
| Show staged changes | `git diff --cached` |
| Show recent commits | `git log --oneline --decorate -10` |
| Show configured remote repository | `git remote -v` |
| Download remote information without changing files | `git fetch --all --prune` |
| Push the current branch | `git push -u origin HEAD` |

Do not use `git reset --hard`, forced push, or file-deletion commands until the exact branch and commit state are understood.

### 3.4 Recommended `.gitignore`

Large data, software environments, containers, and credentials usually should not enter a Git repository.

```gitignore
# Credentials and local settings
.env
*.pem
*.key
id_rsa*

# Python
__pycache__/
*.py[cod]
.venv/
.ipynb_checkpoints/

# Conda or Micromamba environments
envs/
conda-env/

# Containers
*.sif

# Large scientific data and trajectories
data/raw/
*.xtc
*.trr
*.dcd

# Temporary files
*.tmp
*.swp
```

Keep small example inputs and selected result summaries in GitHub. Store raw datasets, molecular dynamics trajectories, database files, and container images in approved institutional storage.

## 4. Linux filesystem basics

### 4.1 Always identify the current location

Before running a command that creates, moves, or deletes files, check:

```bash
pwd
ls -lah
```

`pwd` prints the current directory. `ls -lah` shows files, permissions, sizes, and hidden files.

### 4.2 Absolute and relative paths

An absolute path begins at `/`:

```text
/engrfs/project/group/project1/input.pdb
```

A relative path begins at the current directory:

```text
input/input.pdb
```

Useful path symbols:

| Symbol | Meaning |
|---|---|
| `.` | Current directory |
| `..` | Parent directory |
| `~` | User home directory |
| `/` | Filesystem root or path separator |

Linux paths are case-sensitive. `Protein.pdb` and `protein.pdb` are different filenames.

### 4.3 Common file commands

| Goal | Command |
|---|---|
| Enter a directory | `cd path/to/folder` |
| Go to the parent directory | `cd ..` |
| Create a directory | `mkdir -p results/run01` |
| Copy a file | `cp source.txt destination.txt` |
| Copy a directory | `cp -r source_dir destination_dir` |
| Move or rename a file | `mv old_name.txt new_name.txt` |
| Identify a file type | `file filename` |
| Show directory size | `du -sh directory_name` |
| Show filesystem space | `df -h` |
| Read a text file safely | `less filename` |
| Show the end of a log | `tail -n 50 job.out` |
| Follow a running log | `tail -f job.out` |
| Search text recursively | `rg "search term"` |
| Find a file by name | `find . -name "*.pdb"` |

Quote paths containing spaces:

```bash
cd "folder with spaces"
```

### 4.4 Permissions

Inspect permissions:

```bash
ls -l script.sh
```

Make your own script executable:

```bash
chmod u+x script.sh
```

`Permission denied` can mean that the file is not executable, the directory is not accessible, the filesystem is read-only, or the user does not own the target. Do not apply `chmod 777` as a general fix.

### 4.5 Deletion safety

Before deleting anything, print the current directory and inspect the exact target:

```bash
pwd
ls -lah path/to/target
```

Avoid recursive deletion until the target path is fully confirmed. GitHub is not a backup for files that were never committed and pushed.

## 5. Connecting to the WashU Engineering HPC

Replace `WUSTL_USERNAME` with the actual WashU username:

```bash
ssh WUSTL_USERNAME@shell3.engr.wustl.edu
```

The login host is for light work such as editing files, moving data, checking environments, and submitting jobs. CPU- or GPU-heavy programs should run on a compute node allocated by Slurm.

After login, confirm the system and location:

```bash
hostname
whoami
pwd
```

If the cluster asks you to enter a Linux lab node before working, follow the current Engineering HPC message or documentation. Cluster rules and node names can change.

## 6. Slurm job management

### 6.1 Important terms

| Term | Meaning |
|---|---|
| Partition | A group of compute nodes, such as CPU or GPU nodes |
| Account | The allocation charged or authorized for the job |
| QOS | Quality-of-service rules controlling priority and limits |
| Job | A requested calculation managed by Slurm |
| Node | A physical or virtual compute machine |
| Task | A process launched within a job |
| CPU | Processor core resource |
| GPU | Graphics processor resource |
| Memory | RAM requested for the job |
| Wall time | Maximum elapsed time requested for the job |

### 6.2 Check available access

On systems where the Slurm commands are provided through a module:

```bash
module load slurm
```

Check accounts and QOS values:

```bash
sacctmgr show assoc user="$USER" format=User,Account,Partition,QOS
```

Known Engineering account used for this work:

```text
engr-lab-joshua.yuan
```

Do not assume that the same account or partition name will work on RIS Compute2 or another cluster.

### 6.3 Interactive GPU allocation

The following command is a known Engineering-cluster example. Confirm current resource names with `sinfo` before reusing it later.

```bash
srun -p general-gpu \
  -A engr-lab-joshua.yuan \
  --gpus=a100_10gb:1 \
  -c 2 \
  --mem=16G \
  -t 01:00:00 \
  -J gpu_test \
  --pty bash
```

After allocation:

```bash
hostname
nvidia-smi
```

When finished:

```bash
exit
```

### 6.4 Batch job template

Create the log directory before submitting because Slurm does not create missing parent directories for log files:

```bash
mkdir -p logs
```

Example `run_gpu.slurm`:

```bash
#!/usr/bin/env bash
#SBATCH --job-name=example_gpu
#SBATCH --partition=general-gpu
#SBATCH --account=engr-lab-joshua.yuan
#SBATCH --gpus=a100_10gb:1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err

set -euo pipefail

echo "Start: $(date)"
echo "Host: $(hostname)"
echo "Directory: $(pwd)"
nvidia-smi

# Prefer an explicit environment call in batch jobs.
micromamba run -n ENVIRONMENT_NAME python script.py

echo "End: $(date)"
```

Submit it:

```bash
sbatch run_gpu.slurm
```

### 6.5 Monitor and diagnose jobs

| Goal | Command |
|---|---|
| List your queued and running jobs | `squeue -u "$USER"` |
| Inspect one job | `scontrol show job JOB_ID` |
| Check completed-job accounting | `sacct -j JOB_ID --format=JobID,State,ExitCode,Elapsed,MaxRSS,AllocTRES` |
| Cancel one job | `scancel JOB_ID` |
| Inspect partitions and nodes | `sinfo` |

Common job states:

| State | Meaning |
|---|---|
| `PD` | Pending; resources have not been assigned |
| `R` | Running |
| `CG` | Completing |
| `COMPLETED` | Finished successfully according to Slurm |
| `FAILED` | Program or job step failed |
| `OUT_OF_MEMORY` | Memory request was too small |
| `TIMEOUT` | Wall-time limit was reached |
| `CANCELLED` | Job was cancelled |

A `COMPLETED` Slurm state only means the process exited successfully. Scientific outputs must still be checked for completeness and validity.

## 7. Conda and Micromamba environments

### 7.1 What an environment does

An environment isolates software versions for one workflow. It is not the scientific workflow itself.

```text
HPC -> environment -> scientific software -> input files -> output files
```

Use separate environments when programs have conflicting dependencies. Avoid installing every package into `base`.

### 7.2 Basic checks

```bash
conda env list
conda activate ENVIRONMENT_NAME
which python
python --version
conda list
```

For Micromamba:

```bash
micromamba env list
micromamba activate ENVIRONMENT_NAME
which python
python --version
micromamba list
```

When a command fails, record both `which PROGRAM_NAME` and `PROGRAM_NAME --version`. The active environment name alone does not prove that the intended executable is being used.

### 7.3 Save a reproducible environment description

For a concise Conda file containing explicitly requested packages:

```bash
conda env export --from-history -n ENVIRONMENT_NAME > environment.yml
```

For an exact package record useful on a similar platform:

```bash
conda list --explicit -n ENVIRONMENT_NAME > explicit-spec.txt
```

Recreate from `environment.yml`:

```bash
conda env create -f environment.yml
```

Record operating system, GPU model, CUDA requirements, channel order, and any packages installed with `pip`.

### 7.4 Diagnose environment confusion

```bash
echo "$CONDA_DEFAULT_ENV"
which python
which PROGRAM_NAME
python -c "import sys; print(sys.executable)"
python -m pip --version
```

Using `python -m pip` is safer than a bare `pip` because it shows which Python installation receives the package.

## 8. Apptainer containers

Check availability:

```bash
which apptainer
apptainer --version
```

Run a command inside a container:

```bash
apptainer exec software.sif PROGRAM_NAME
```

Enable NVIDIA GPU access:

```bash
apptainer exec --nv software.sif nvidia-smi
```

Bind an HPC project directory into the container:

```bash
apptainer exec --nv \
  --bind /path/on/hpc:/work \
  software.sif \
  PROGRAM_NAME --input /work/input.pdb
```

The path outside the container and the path inside the container can differ. When a program reports that an input file does not exist, check the bind mount and the path visible inside the container:

```bash
apptainer exec --bind /path/on/hpc:/work software.sif ls -lah /work
```

Do not place large `.sif` images in GitHub.

## 9. Coding practices for scientific workflows

### 9.1 Keep inputs, code, logs, and outputs separate

A useful repository structure is:

```text
Washu-HPC/
├── README.md
├── docs/
│   ├── linux.md
│   ├── slurm.md
│   ├── environments.md
│   └── troubleshooting.md
├── scripts/
├── slurm/
├── examples/
├── environment/
│   └── environment.yml
└── .gitignore
```

Large raw inputs and calculation outputs should remain outside the Git repository. The repository should contain code, small test inputs, configuration files, selected summary results, and instructions for locating the full data.

### 9.2 Make scripts traceable

A calculation record should include:

- Git commit ID;
- date and time;
- cluster and node;
- working directory;
- software and version;
- environment or container;
- exact command;
- input filenames and checksums when needed;
- random seed when relevant;
- Slurm job ID and requested resources;
- output location;
- validation result.

Get the current commit ID:

```bash
git rev-parse HEAD
```

Create a checksum:

```bash
sha256sum input_file
```

### 9.3 Bash error handling

For new Bash workflow scripts, consider:

```bash
set -euo pipefail
```

- `-e` stops after an unhandled command failure.
- `-u` reports use of an undefined variable.
- `pipefail` reports a failure inside a command pipeline.

Test the script on a small input before requesting a long GPU job.

## 10. Troubleshooting workflow

When a problem occurs, do not change many things at once. Use this order:

1. Copy the exact command and full error message.
2. Record `hostname`, `pwd`, current branch, and active environment.
3. Confirm that every input path exists.
4. Confirm the actual program path and version.
5. Check permissions, disk space, and quota.
6. Check Slurm resources and job exit state.
7. Reduce the problem to the smallest test input.
8. Change one factor.
9. Rerun the test and record whether the change worked.
10. Add the verified solution to this manual.

Useful diagnostic block:

```bash
date
hostname
whoami
pwd
git status
git branch --show-current
echo "$CONDA_DEFAULT_ENV"
which python
python --version
df -h
```

### 10.1 Common errors

| Error or symptom | Likely cause | First checks |
|---|---|---|
| `command not found` | Program is not installed, environment is inactive, or `PATH` is wrong | `which command`; environment list; module list |
| `ModuleNotFoundError` | Package is absent from the Python currently running | `which python`; `python -m pip show package` |
| `No such file or directory` | Wrong directory, filename case, relative path, or container bind path | `pwd`; `ls -lah`; inspect absolute path |
| `Permission denied` | Missing execute/read/write permission | `ls -l`; ownership; filesystem rules |
| Job remains `PD` | Requested resource is unavailable or request violates a limit | `squeue`; `scontrol show job`; pending reason |
| `OUT_OF_MEMORY` or process is killed | Requested RAM or GPU memory is too small | Slurm state; `MaxRSS`; program memory use |
| CUDA is unavailable | No GPU was allocated, container lacks `--nv`, or software/CUDA mismatch | `nvidia-smi`; Slurm request; program version |
| Works interactively but fails in `sbatch` | Batch shell did not load the same module, environment, path, or variables | Log environment and use explicit paths |
| `^M` or bad interpreter | Windows line endings in a Linux script | `file script.sh`; convert to Unix line endings |
| GitHub shows old files | Changes are uncommitted, unpushed, or on another branch | `git status`; branch; log; push; GitHub branch menu |
| Disk or quota error | Home, project, or scratch storage is full | `df -h`; `du -sh`; cluster quota command |

## 11. Problem record template

Copy this section for each new issue:

````markdown
## YYYY-MM-DD — Short problem title

**System:** Engineering HPC / RIS Compute2 / Codespace / local computer  
**Host or node:**  
**Working directory:**  
**Git branch and commit:**  
**Environment or container:**  
**Software version:**  
**Slurm job ID:**  

### Goal

What was the intended result?

### Exact command

```bash
paste the command here
```

### Error or symptom

Paste the complete error and describe the observed output.

### Cause

What caused the problem? Separate confirmed facts from possible explanations.

### Fix

```bash
paste the verified fix here
```

### Verification

What result showed that the fix worked?

### Prevention

What check or change can prevent the same problem next time?
````

## 12. Recorded issue 001: Codespace files missing from GitHub webpage

**Date:** 2026-09-15  
**System:** GitHub Codespaces  
**Codespace:** `redesigned-goldfish`  
**Repository:** `Washu-HPC`

### Symptom

Files created in the Codespace were still present after closing and reopening the editor, but the GitHub repository webpage displayed only `README.md`.

### Cause

The files were saved on the Codespace virtual machine. Saving and reopening a Codespace does not mean that the files have been committed and pushed to the GitHub repository. Another possible contributing factor is viewing a different branch on the GitHub webpage.

### Diagnosis

```bash
git status
git branch --show-current
git log --oneline --decorate -10
git remote -v
```

### Fix

After reviewing files for secrets and unwanted large outputs:

```bash
git add path/to/file1 path/to/file2
git diff --cached
git commit -m "Add HPC manual files"
git push -u origin HEAD
```

On GitHub, select the same branch and verify that the files appear.

### Prevention

Before ending each Codespace session:

```bash
git status
git add path/to/file1 path/to/file2
git diff --cached
git commit -m "Describe the completed work"
git push
git status
```

The final `git status` should show a clean working tree and no unpushed commits. Verification on the GitHub webpage is still recommended.

## 13. End-of-work checklist

Before leaving a Codespace or HPC session:

- Save all edited files.
- Run `git status`.
- Review changes and check for credentials or large data.
- Commit and push the intended source files and notes.
- Confirm the correct branch on GitHub.
- Record the environment or container version.
- Record the Slurm job ID and output path.
- Check whether long jobs are still running.
- Move irreplaceable results to approved persistent storage.
- Stop the Codespace or exit the interactive HPC allocation when finished.

## 14. Reference documentation

- [GitHub: Using source control in a Codespace](https://docs.github.com/en/codespaces/developing-in-a-codespace/using-source-control-in-your-codespace)
- [GitHub: Understanding the Codespace lifecycle](https://docs.github.com/en/codespaces/about-codespaces/understanding-the-codespace-lifecycle)
- [Slurm documentation](https://slurm.schedmd.com/documentation.html)
- [Conda environment management](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html)
- [Apptainer user guide](https://apptainer.org/docs/user/latest/)

---

When adding a solution, include enough context that the command can be understood six months later without relying on memory.
