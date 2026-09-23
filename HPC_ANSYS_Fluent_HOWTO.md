# HPC–ANSYS Fluent HOWTO

## Purpose

This document describes the complete workflow for transferring an initialized ANSYS Fluent case from a Windows workstation to the laboratory HPC, running it through SLURM on a compute node, saving checkpoints/results, and continuing a simulation from a saved case/data pair.

The workflow used here is:

```text
Windows workstation
        |
        | SCP / SSH
        v
HPC master
        |
        | shared /data
        v
SLURM
        |
        v
Compute node
        |
        v
ANSYS Fluent
        |
        v
/data/wrk_spc/results/
```

---

# 1. HPC setup used

The cluster currently uses:

- Master: `master-ammpl`
- Compute nodes: `compute01`, `compute02`, `compute03`, `compute04`
- Shared filesystem: `/data`
- ANSYS installation: `/data/ansys_inc/v261/`
- Fluent executable: `/data/ansys_inc/v261/fluent/bin/fluent`
- Fluent version: ANSYS 2026 R1
- SLURM: 26.05.2
- License server: `1055@192.168.10.100`

The working directory for this project is:

```text
/data/wrk_spc/
```

---

# 2. Create the working directory on the HPC

## 2.1 SSH into the master

From Windows PowerShell:

```powershell
ssh jena@10.48.15.79
```

After logging in, confirm that you are on the master:

```bash
hostname
```

Expected:

```text
master-ammpl
```

---

## 2.2 Create the directory structure

Run this on the master:

```bash
mkdir -p /data/wrk_spc/{bin,cases,udf,journals,jobs,results,logs,slurm_tests}
```

Check it:

```bash
tree -L 2 /data/wrk_spc
```

If `tree` is not installed, use:

```bash
find /data/wrk_spc -maxdepth 2 -type d
```

The expected structure is:

```text
/data/wrk_spc/
├── bin/
├── cases/
├── udf/
├── journals/
├── jobs/
├── results/
├── logs/
└── slurm_tests/
```

---

# 3. Prepare the Fluent case on Windows

Before transferring the simulation, save an initialized Fluent case and data file.

The two important files are:

```text
*.cas.h5
*.dat.h5
```

The `.cas.h5` file contains the Fluent model/setup.

The `.dat.h5` file contains the current solution state.

For a restartable simulation, keep the matching case and data files together.

For example:

```text
FFF-6-Setup-Output.cas.h5
FFF-6-Setup-Output.dat.h5
```

If the simulation has been initialized at `t = 0`, save both files at that point.

Do not overwrite the original files during HPC testing.

---

# 4. Check SCP on Windows

Open Windows PowerShell.

Run:

```powershell
scp -V
```

If OpenSSH is installed, PowerShell should print the SCP/OpenSSH version.

You can also verify SSH:

```powershell
ssh -V
```

---

# 5. Transfer the case file

Assume the Windows case is stored at:

```text
E:\Swastik\Fluent_sim\micro-stirrer\v2\saves\liq+rotor+mwcnt\
```

Transfer the case file with:

```powershell
scp "E:\Swastik\Fluent_sim\micro-stirrer\v2\saves\liq+rotor+mwcnt\FFF-6-Setup-Output.cas.h5" jena@10.48.15.79:/data/wrk_spc/cases/micro-caster-diagnosis/
```

Enter the HPC password when prompted.

---

# 6. Transfer the data file

Transfer the matching data file:

```powershell
scp "E:\Swastik\Fluent_sim\micro-stirrer\v2\saves\liq+rotor+mwcnt\FFF-6-Setup-Output.dat.h5" jena@10.48.15.79:/data/wrk_spc/cases/micro-caster-diagnosis/
```

If the destination directory does not exist, create it first on the master:

```bash
mkdir -p /data/wrk_spc/cases/micro-caster-diagnosis
```

---

# 7. Verify the transferred files

SSH into the master:

```powershell
ssh jena@10.48.15.79
```

Then:

```bash
ls -lh /data/wrk_spc/cases/micro-caster-diagnosis/
```

You should see both:

```text
FFF-6-Setup-Output.cas.h5
FFF-6-Setup-Output.dat.h5
```

Check their sizes:

```bash
du -h /data/wrk_spc/cases/micro-caster-diagnosis/*
```

The files should have approximately the same sizes as the originals on Windows.

---

# 8. Confirm that compute nodes can see the files

Because `/data` is shared through NFS, the case does not need to be copied separately to every compute node.

From the master:

```bash
ssh compute02
```

Then:

```bash
ls -lh /data/wrk_spc/cases/micro-caster-diagnosis/
```

The same case/data files should be visible.

Exit:

```bash
exit
```

---

# 9. Verify the Fluent installation

On the master:

```bash
ls -lh /data/ansys_inc/v261/fluent/bin/fluent
```

You can also check the Fluent executable:

```bash
/data/ansys_inc/v261/fluent/bin/fluent 3ddp -help
```

Do not start a production simulation with this command. It is only an installation check.

---

# 10. Verify the license from a compute node

SSH to a compute node:

```bash
ssh compute02
```

Set the license server:

```bash
export ANSYSLMD_LICENSE_FILE=1055@192.168.10.100
```

Then check the environment:

```bash
echo $ANSYSLMD_LICENSE_FILE
```

Expected:

```text
1055@192.168.10.100
```

Return to the master:

```bash
exit
```

---

# 11. Understand the SLURM workflow

SLURM controls access to the compute nodes.

The basic sequence is:

```text
sbatch job.sh
       |
       v
SLURM scheduler
       |
       v
compute02
       |
       v
Fluent
```

Do not normally launch a long production Fluent job directly on the master.

---

# 12. Check available nodes

Run:

```bash
sinfo
```

For a more detailed view:

```bash
sinfo -N -l
```

Check the queue:

```bash
squeue
```

An empty `squeue` means there are currently no jobs visible in the queue.

---

# 13. Create a Fluent SLURM job

Create a job directory:

```bash
mkdir -p /data/wrk_spc/jobs/micro-caster-20core
```

Create the script:

```bash
nano /data/wrk_spc/jobs/micro-caster-20core/run_20core.sh
```

A job script should specify:

- SLURM resources
- compute node requirements
- Fluent executable
- number of Fluent processes
- case file
- data file
- journal file
- output/log locations

Keep the actual script used by the laboratory under version control or preserve a known working copy.

---

# 14. Submit the job

Make the script executable:

```bash
chmod +x /data/wrk_spc/jobs/micro-caster-20core/run_20core.sh
```

Submit:

```bash
sbatch /data/wrk_spc/jobs/micro-caster-20core/run_20core.sh
```

SLURM returns a job ID, for example:

```text
Submitted batch job 77
```

Record the job ID.

---

# 15. Monitor the job

Check the queue:

```bash
squeue
```

For a specific job:

```bash
squeue -j 77
```

If the job is running, SLURM will show the allocated node.

For example:

```text
compute02
```

---

# 16. Monitor Fluent output

The SLURM output/log file should be written under the project `logs` directory.

For example:

```bash
ls -lh /data/wrk_spc/logs/
```

To monitor a growing log:

```bash
tail -f /data/wrk_spc/logs/<logfile>
```

Press:

```text
Ctrl+C
```

to stop following the file. This does not stop the SLURM job.

---

# 17. Case/data checkpointing

A Fluent checkpoint should contain both:

```text
checkpoint.cas.h5
checkpoint.dat.h5
```

For example:

```text
/data/wrk_spc/results/micro-caster-debug/
├── debug-final.cas.h5
└── debug-final.dat.h5
```

The matching case/data pair represents a restartable simulation state.

Do not rename one file without renaming its matching partner.

---

# 18. Successful diagnostic run

A 20-core diagnostic run was successfully performed with:

```text
Initial flow time:       0.00 s
Final flow time:         0.20 s
Time step:               0.01 s
Iterations/time step:    20
Fluent processes:        20
Compute node:            compute02
```

The run reached:

```text
Flow time = 0.2 s
```

and exited successfully.

The resulting checkpoint was:

```text
/data/wrk_spc/results/micro-caster-debug/debug-final.cas.h5
/data/wrk_spc/results/micro-caster-debug/debug-final.dat.h5
```

This checkpoint can be used for subsequent continuation tests.

---

# 19. Restarting from a checkpoint

A restart does not require going back to the original Windows case.

For example:

```text
debug-final.cas.h5
debug-final.dat.h5
```

can be loaded by a new journal.

The workflow becomes:

```text
debug-final.cas.h5
debug-final.dat.h5
        |
        v
Load into Fluent
        |
        v
Continue from t = 0.20 s
        |
        v
Save a new checkpoint
```

Always save the new checkpoint under a different name if you want to preserve the previous state.

For example:

```text
debug-0p20.cas.h5
debug-0p20.dat.h5

debug-0p22.cas.h5
debug-0p22.dat.h5
```

---

# 20. Recommended checkpoint naming

Use the physical time in the filename.

For example:

```text
micro-caster-t0p20.cas.h5
micro-caster-t0p20.dat.h5

micro-caster-t0p50.cas.h5
micro-caster-t0p50.dat.h5

micro-caster-t1p00.cas.h5
micro-caster-t1p00.dat.h5
```

Use `p` instead of `.` in filenames.

This makes it immediately clear which simulation state each file represents.

---

# 21. Important rule: protect the original case

Keep the original initialized files here:

```text
/data/wrk_spc/cases/micro-caster-diagnosis/
```

Do not use them as the output destination for a long production run.

Instead:

```text
cases/
    original initialized case

results/
    simulation checkpoints
```

This prevents a failed simulation from destroying the known-good starting state.

---

# 22. Current numerical-debugging checkpoint

The current known-good checkpoint is approximately:

```text
Flow time:       0.20 s
Δt:              0.01 s
Processes:       20
Node:            compute02
```

A continuation test from this checkpoint was used to investigate instability.

The solution becomes unstable around:

```text
0.215–0.220 s
```

A second test using:

```text
Δt = 0.005 s
```

also became unstable around the same physical-time region.

Therefore, the HPC execution system itself is working; the remaining issue is being investigated as a Fluent model/numerical stability problem.

---

# 23. Useful commands

### Check current machine

```bash
hostname
```

### Check SLURM nodes

```bash
sinfo -N
```

### Check running jobs

```bash
squeue
```

### Check your jobs

```bash
squeue -u $USER
```

### Cancel a job

```bash
scancel JOB_ID
```

Example:

```bash
scancel 77
```

### Check files

```bash
ls -lh /data/wrk_spc/results/
```

### Check directory sizes

```bash
du -sh /data/wrk_spc/*
```

### Follow a log

```bash
tail -f /data/wrk_spc/logs/<logfile>
```

### Check Fluent installation

```bash
ls -lh /data/ansys_inc/v261/fluent/bin/fluent
```

### Check license variable

```bash
echo $ANSYSLMD_LICENSE_FILE
```

---

# 24. Complete workflow at a glance

```text
STEP 1
Prepare initialized Fluent case on Windows
        |
        v
STEP 2
Save matching .cas.h5 + .dat.h5
        |
        v
STEP 3
SSH to HPC master
        |
        v
STEP 4
Create /data/wrk_spc structure
        |
        v
STEP 5
Transfer case/data using SCP
        |
        v
STEP 6
Verify files on master
        |
        v
STEP 7
Verify shared filesystem from compute node
        |
        v
STEP 8
Prepare Fluent journal
        |
        v
STEP 9
Prepare SLURM job script
        |
        v
STEP 10
Submit with sbatch
        |
        v
STEP 11
Monitor with squeue/logs
        |
        v
STEP 12
Fluent runs on compute node
        |
        v
STEP 13
Save case/data checkpoint
        |
        v
STEP 14
Continue from checkpoint
        |
        v
STEP 15
Save final results
```

# 25. Final directory structure

After the workflow is fully established, the project should look approximately like:

```text
/data/wrk_spc/
│
├── bin/
│   └── fluent_slurm.sh
│
├── cases/
│   └── micro-caster-diagnosis/
│       ├── FFF-6-Setup-Output.cas.h5
│       └── FFF-6-Setup-Output.dat.h5
│
├── udf/
│   └── ...
│
├── journals/
│   ├── debug-0p20.jou
│   ├── continue-0p20-0p22.jou
│   └── ...
│
├── jobs/
│   └── micro-caster-20core/
│       └── run_20core.sh
│
├── results/
│   └── micro-caster-debug/
│       ├── debug-final.cas.h5
│       └── debug-final.dat.h5
│
├── logs/
│   └── ...
│
└── slurm_tests/
    └── ...
```
This structure should be treated as the standard working structure for future Fluent HPC simulations.
---
# Authors
- Swastik Jena
- Md Tabraiz Imam
