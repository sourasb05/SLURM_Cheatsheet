# Slurm Cheatsheet (sbatch and job monitoring)

Cluster notes: GPU partition is `haswell` (other partitions: `pelle`, `gpu`, `fat`).
`-h` drops the header line (handy for piping). `-o "..."` picks columns (see `man squeue`).

## 1. Submitting and managing jobs

```bash
sbatch run.sh                          # submit a batch script, prints "Submitted batch job <id>"
sbatch --parsable run.sh               # print only the job id (good for scripting / dependencies)
sbatch -p haswell --gres=gpu:1 run.sh  # override partition / GPU request from the command line
sbatch --array=0-4 run.sh              # job array (e.g. one task per client / seed); use $SLURM_ARRAY_TASK_ID
sbatch --dependency=afterok:<jobid> run2.sh   # start only after <jobid> finishes successfully
sbatch --export=ALL,SEED=1,TAU=0.5 run.sh     # pass variables to the script
scancel <jobid>                        # cancel one job
scancel -u $USER                       # cancel ALL your jobs
scancel -u $USER -t PENDING            # cancel only pending jobs
scancel -u $USER --name=<jobname>      # cancel by job name
scontrol hold <jobid>                  # keep a pending job from starting
scontrol release <jobid>               # release a held job
scontrol update JobId=<jobid> TimeLimit=12:00:00   # change time limit (can usually only lower it)
srun -p haswell --gres=gpu:1 --pty bash            # interactive shell on a GPU node
```

Typical `#SBATCH` header:

```bash
#SBATCH -J fedids
#SBATCH -p haswell
#SBATCH --gres=gpu:1
#SBATCH -c 4
#SBATCH --mem=32G
#SBATCH -t 24:00:00
#SBATCH -o logs/%x_%j.out     # %x = job name, %j = job id
#SBATCH -e logs/%x_%j.err
```

## 2. Your jobs

```bash
squeue -u $USER                                  # all your jobs
squeue -u $USER -h -o "%T %j"                    # state + name
squeue -u $USER -h -o "%T|%j|%i|%r|%S"           # state | name | job id | pending reason | estimated start
squeue -u $USER -h -t PENDING -o "%r" | sort | uniq -c   # count of pending reasons (Priority / Resources)
```

### How many are running / pending

```bash
squeue -u $USER -h -t RUNNING | wc -l            # number running
squeue -u $USER -h -t PENDING | wc -l            # number pending
squeue -u $USER -h -o "%T" | sort | uniq -c      # count per state (RUNNING, PENDING, ...)
```

One-liner summary:

```bash
echo "running: $(squeue -u $USER -h -t RUNNING | wc -l)  pending: $(squeue -u $USER -h -t PENDING | wc -l)"
```

### When running jobs will finish

For running jobs, the expected finish is the **start time + time limit** (the job may end earlier if it completes).

```bash
# job id | name | elapsed | time limit | time left | latest end time
squeue -u $USER -h -t RUNNING -o "%i|%j|%M|%l|%L|%e"
```

| Code | Meaning |
|------|---------|
| `%M` | time used so far |
| `%l` | time limit |
| `%L` | time left before the limit kills the job |
| `%e` | end time (start + limit) |
| `%S` | start time (actual for running jobs, estimated for pending) |

Sorted by soonest to finish:

```bash
squeue -u $USER -h -t RUNNING -o "%e|%i|%j|%L" | sort
```

Note: `%e` is the worst case (job hits its limit). Jobs usually finish earlier; use `%M` against your known per-run duration for a better guess.

### When pending jobs are expected to start

```bash
squeue -u $USER --start                          # Slurm's estimated start time for each pending job
squeue -u $USER -h -t PENDING -o "%i|%j|%r|%S"   # id | name | reason | estimated start
```

Estimates can be `N/A` early on and shift as other jobs finish or are submitted.

Pending reasons: `Priority` (others ahead of you in the queue), `Resources` (waiting for free GPUs/CPUs/memory), `Dependency` (waiting on another job), `JobHeldUser` (you held it), `QOSMaxJobsPerUserLimit` (per-user job limit hit).

## 3. The partition and its GPUs

```bash
sinfo -p haswell -o "%P %a %D %T %G %c %m"       # haswell nodes by state, GPU type (GRES), CPUs, memory
sinfo -o "%P %D %T %G"                           # same for all partitions
sinfo -p haswell -N -h -o "%N %T %G %C"          # per node: state, GPUs, CPUs as allocated/idle/other/total
```

`%C` prints `allocated/idle/other/total` CPUs (e.g. `4/12/0/16`).

## 4. Who is using the GPUs

```bash
squeue -p haswell -t RUNNING -h -o "%u %b" | sort | uniq -c   # running jobs by user and GPU request
squeue -p haswell -h -t RUNNING | wc -l                       # total running in the partition
squeue -p haswell -h -t PENDING | wc -l                       # total pending in the partition
```

### Who is another user and what do they run

```bash
getent passwd <user>                             # user database entry (full name)
squeue -u <user> -h -o "%T %j %M %l"             # state, job name, elapsed time, time limit
```

## 5. Priority, fairshare, details

```bash
sprio -u $USER                  # priority breakdown (age, fairshare, size) for pending jobs
sshare -u $USER                 # your fairshare usage
scontrol show job <jobid>       # full details of one job, including why it is pending
scontrol show node p2001        # full details of one node (GRES, allocated resources, drain reason)
```

## 6. History and finished jobs

```bash
sacct -u $USER -S today                                   # today's jobs with elapsed time and state
sacct -u $USER -S now-7days -o JobID,JobName,State,Elapsed,ExitCode,NodeList
sacct -j <jobid> -o JobID,State,Elapsed,MaxRSS,ExitCode   # one job: memory peak, exit code
seff <jobid>                                              # CPU / memory efficiency of a finished job
```

States to know: `COMPLETED`, `FAILED`, `CANCELLED`, `TIMEOUT` (hit time limit), `OUT_OF_MEMORY`, `NODE_FAIL`.

## 7. Watching logs

```bash
tail -f logs/<name>_<jobid>.out          # follow output live
ls -lt logs | head                       # most recent logs
watch -n 30 'squeue -u $USER'            # refresh queue every 30 s
```

## 8. Quick status dashboard (copy/paste)

```bash
echo "== counts =="; squeue -u $USER -h -o "%T" | sort | uniq -c
echo "== running (id|name|elapsed|left|ends) =="; squeue -u $USER -h -t RUNNING -o "%i|%j|%M|%L|%e" | sort -t'|' -k5
echo "== pending (id|name|reason|est. start) =="; squeue -u $USER -h -t PENDING -o "%i|%j|%r|%S"
```
