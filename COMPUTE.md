# Compute

<!--TOC-->

---

**Table of Contents**

- [Office of Research Computing and Data (ORCD)](#office-of-research-computing-and-data-orcd)
  - [Login and Compute Nodes](#login-and-compute-nodes)
  - [SSH](#ssh)
    - [Recovering a stale master](#recovering-a-stale-master)
    - [`rsync` over SSH](#rsync-over-ssh)
    - [Keeping the `ControlMaster` alive](#keeping-the-controlmaster-alive)
    - [Avoiding repeated MFA on Windows](#avoiding-repeated-mfa-on-windows)
    - [SSH Into Compute Nodes](#ssh-into-compute-nodes)
      - [Port-forwarding to a compute node](#port-forwarding-to-a-compute-node)
  - [Web Portal](#web-portal)
  - [Slurm](#slurm)
    - [Interactive allocations](#interactive-allocations)
    - [Accounts and QoS tiers](#accounts-and-qos-tiers)
    - [`uv`-managed environments](#uv--managed-environments)
  - [GPUs](#gpus)
  - [Filesystems](#filesystems)
    - [Group storage](#group-storage)
  - [Python (Miniforge)](#python-miniforge)
  - [Ray](#ray)
- [Economics](#economics)
- [VPN](#vpn)

---

<!--TOC-->

## Office of Research Computing and Data (ORCD)

Links:

- Home: <https://orcd.mit.edu/>
- Docs: <https://orcd-docs.mit.edu/>
  - Docs source: <https://github.com/mit-orcd/orcd-docs-edit>

The main HPC cluster at MIT is called "Engaging"
and is available to all MIT research projects (not department-specific).
It uses Slurm for job management.
You connect to it using [SSH](#ssh) or the [web portal](#web-portal).

From <https://orcd-docs.mit.edu/orcd-systems> as of 6/22/2026:

> It has around 50,000 x86 CPU cores and over 1000 GPU cards including A100, RTX6000, L40S, H100, and H200 GPUs.

Per [this MGHPCC shutdown][mghpcc-shutdown] blog,
Engaging is part of the [Massachusetts Green High Performance Computing Center][mghpcc-wikipedia]
in western Massachusetts (Holyoke).

[mghpcc-shutdown]: https://orcd.mit.edu/news/mghpcc-power-shutdown-june-15-18
[mghpcc-wikipedia]: https://en.wikipedia.org/wiki/Massachusetts_Green_High_Performance_Computing_Center

### Login and Compute Nodes

When you `ssh orcd` you land on a login node. `orcd-login.mit.edu` is a single public
address that distributes your connection to one of 10 login nodes (`login001` through `login010`).
The number of login nodes may change over time, this was discerned manually on 6/29/2026.
`/home` and `/orcd` are NFS mounted identically on every login and compute node,
so all nodes see the same files and it doesn't matter which login node you land on.
See [Filesystems](#filesystems) for sizes, backends, and details.

Each login node is bare-metal hardware, not a VM or Kubernetes pod.
As of 6/29/2026 the login nodes were a Dell PowerEdge R6625
with two AMD EPYC 9734 CPUs (448 hardware threads) and 768 GiB RAM,
on Rocky Linux 8.10, with your shell session inside an Apptainer container.
That capacity is shared by everyone logged in (dozens at a time), so login nodes are
for light work only. Per [ORCD's Getting Started][getting-started] guide:

> The login node, as its name suggests, is where you log in and is for editing code and files,
> installing packages and software, downloading data, and starting jobs to run your code on one
> of the compute nodes. [...] It is very important not to run anything unless it is submitted
> properly through the scheduler.

Run real computation on a compute node through [Slurm](#slurm), never on a login node.
Compute nodes are not directly reachable from the internet
(they have only private IPs and no public DNS); you reach an allocated compute node
by jumping through a login node (see [SSH](#ssh)).
Each compute node's SSH login service also gates who may connect using Slurm.
Attempts to SSH into a node where you don't have a running job
will be denied by Slurm PAM (Pluggable Authentication Modules):

> Access denied by pam_slurm_adopt: you have no active jobs on this node

On SSH login, each compute node's Slurm PAM looks for an active job of yours on the node.
If it finds one it adopts your session into the job's control group; otherwise it denies the connection.

[getting-started]: https://orcd-docs.mit.edu/getting-started/

### SSH

Generate an SSH key via `ssh-keygen -t ed25519 -C "user@mit-orcd"`.
Then copy it to the cluster via `ssh-copy-id user@orcd-login.mit.edu`.
Also make this `~/.ssh/config` file entry (per [ORCD's control-channel docs][ssh-controlchannel]):

```none
Host orcd
    HostName orcd-login.mit.edu
    ControlMaster auto
    ControlPath ~/.ssh/%r@%h:%p
    ControlPersist 8h
    User user
```

If you want, you can add `ForwardAgent yes` to also forward your GitHub SSH key.

ORCD's docs show `ControlPersist 300s`;
we bumped to `8h` so the master survives long gaps between connections while long-ish scripts run
(see [Keeping the `ControlMaster` alive](#keeping-the-controlmaster-alive)).

Since `ControlPath`'s name (the local file where the `ControlMaster` socket will live)
is derived from the general `orcd-login.mit.edu`, not a specific node you reach,
every `ssh orcd` reuses that socket and MFA only prompts once upon the first (master) SSH connection.

To confirm the setup:

<!-- pyml disable-next-line line-length -->

1. SSH key install: `ssh orcd "grep -F \"$(cut -d' ' -f2 ~/.ssh/id_ed25519.pub)\" ~/.ssh/authorized_keys && echo KEY_PRESENT"`
   - Expected output: your key and then "KEY_PRESENT"
1. Master SSH connection: `ssh -O check orcd`
   - Expected output: something like "Master running (pid=12345)"
1. (if you set `ForwardAgent`) GitHub connection: `ssh -T git@github.com`
   - Expected output: "Hi user! You've successfully authenticated, but GitHub does not provide shell access."

#### Recovering a stale master

The `ControlMaster` socket lives at the `ControlPath` file and is reused by every `ssh orcd`.
Due to our specification of `ControlPersist`,
in normal operation the master SSH session will exit and the socket file will be deleted.
The next `ssh orcd` session will have to re-authenticate using MFA.
An annoying case happens when the connection dies (e.g. laptop sleep, login-node restart),
so `ssh orcd` will hang. Running `ssh -O check orcd` will check on the local process
(but does not check whether the link is alive); here's a few possible outputs:

```text
# if `ssh orcd` still hangs -> the local master's connection is stale
Master running (pid=12345)
# To fix, run `ssh -O exit orcd` then `ssh orcd`

# master already gone
Control socket connect(/path/to/.ssh/...): No such file or directory
# Just run `ssh orcd`
```

#### `rsync` over SSH

An `rsync` file transfer running over SSH will be [multiplexed][openssh-multiplexing] through the `ControlMaster`.
If the master's connection has gone stale, the `rsync` invocation will hang indefinitely too.
To avoid this pitfall, pass `--timeout=SECONDS` to `rsync` (e.g. `rsync --timeout=300`)
so a stalled transfer will eventually error out.
Note that `--timeout` is an I/O-inactivity timeout (time since data last moved),
not a cap on total transfer time, so 300-sec is a useful and conservative value.

Also consider passing `--info=stats1` to print just one end-of-transfer summary
to better align with script logs, instead of one line per transferred file.
Here's a sample output for a first-time transfer (`rsync` performed the full sync):

```text
sent 428,942 bytes  received 73 bytes  858,030.00 bytes/sec
total size is 441,453  speedup is 1.03
```

#### Keeping the `ControlMaster` alive

`ControlPersist` is a rolling idle-timeout (that on expiry closes the master `ssh` process),
not a cap on the `ControlMaster`'s total lifetime.
This behavior is documented in [this SSH multiplexing guide][ssh-multiplexing-lowe]:

> Subsequent SSH sessions made while the master connection is open
> will leverage the master connection and will reset the idle timer.

Suppose a coding agent running on your machine is performing periodic status checks on a job
(e.g. `ssh orcd 'sacct -j JOBID'` to query the job's state).
Any check cadence shorter than the `ControlPersist` value doubles as a keepalive;
the master `ssh` process never expires while the laptop stays awake.

What matters is client connections, not traffic.
The `ControlPersist` countdown runs only while the session count is zero:

- Opening a session (count → ≥1): stops the countdown entirely.
  No timer is running while any session is open,
  and no new countdown starts until the session closes.
- Closing the last session (count → 0): starts a fresh, full-length countdown.
  "Idle (with no client connections)" only begins once the last connection has closed.

For example, if idle for 7.5 hours, then a job status check's `ssh orcd` sessions took place
for 6-seconds (0.1-hours), then a new countdown starts and the master `ssh` process lives until hour 15.6.

#### Avoiding repeated MFA on Windows

The `ControlMaster` setup above is OpenSSH connection multiplexing (aka connection sharing),
the industry-standard name for what [ORCD's docs][ssh-login-2fa] call a "control channel."
Native Windows OpenSSH doesn't implement connection sharing:
the [Win32-OpenSSH project][win32-scope] lists "Client ControlMaster"
among features that are "scoped out and will not work on Windows yet."
Windows users can still get connection sharing by running SSH from WSL,
where `ControlMaster` works exactly [as above](#ssh).

Otherwise, Native-Windows users can log into the [OnDemand web portal](#web-portal) first.
Per [ORCD's docs][ssh-login-2fa]:

> Logging into ORCD OnDemand will allow you to log in with an ssh key for a short period of time,
> without the need to enter your MIT Kerberos password and respond to a Duo push.

ORCD's docs don't state the duration; an ORCD staff member mentioned roughly a day.
This shortcut still requires your public SSH key to be [installed on the cluster](#ssh);
logging into OnDemand waives the MIT Kerberos password and Duo push for a short window afterward,
but not the requirement for an installed SSH key.

[ssh-login-2fa]: https://orcd-docs.mit.edu/accessing-orcd/ssh-login/
[win32-scope]: https://github.com/PowerShell/Win32-OpenSSH/wiki/Project-Scope

#### SSH Into Compute Nodes

Compute nodes aren't reachable directly (see [Login and Compute Nodes](#login-and-compute-nodes)),
so you reach an allocated node by jumping through a login node.
Once you have an allocated node, these SSH config entries are handy shorthands.
As the compute node name changes every allocation, instead of hand-editing a `HostName` each session,
define a `Host` entry whose `ProxyCommand` runs `squeue` on the login node to find your current job's node.
Then `ssh orcd-cpu`/`gpu` always lands on whatever node you currently hold in that Slurm partition,
and an editor pointed at the `orcd-cpu`/`gpu` host keeps working across allocations (without config changes):

<!-- pyml disable-num-lines 31 line-length -->

```none
Host orcd-cpu
    # Each compute node has its own fixed host key, but this alias points at a different node
    # almost every allocation, so the key it sees changes nearly every time. That's expected
    # (not tampering, and you reach the node via the trusted login node), so don't verify it
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null  # Keep these one-off node keys out of ~/.ssh/known_hosts
    # `-p mit_normal`: General-purpose CPU partition
    # NOTE: as `squeue` runs on the login node; %%N escapes ssh's own percent-expansion.
    # `head -1` takes the first node `squeue` lists (a consistent order, not random), so with 2+
    # jobs in this partition you may reach one you didn't intend; add e.g. `--name=<job>` to pick
    ProxyCommand ssh orcd 'n=$(squeue --me -h -t R -p mit_normal -o %%N | head -1); [ -n "$n" ] || { echo "orcd-cpu: no running job in mit_normal; salloc one first" >&2; exit 1; }; exec nc "$n" 22'
    # No ControlMaster here; reuses orcd's master socket
    User user

Host orcd-gpu
    # Each compute node has its own fixed host key, but this alias points at a different node
    # almost every allocation, so the key it sees changes nearly every time. That's expected
    # (not tampering, and you reach the node via the trusted login node), so don't verify it
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null  # Keep these one-off node keys out of ~/.ssh/known_hosts
    # `-p mit_normal_gpu`: General-purpose GPU partition
    # NOTE: as `squeue` runs on the login node; %%N escapes ssh's own percent-expansion.
    # `head -1` takes the first node `squeue` lists (a consistent order, not random), so with 2+
    # jobs in this partition you may reach one you didn't intend; add e.g. `--name=<job>` to pick
    ProxyCommand ssh orcd 'n=$(squeue --me -h -t R -p mit_normal_gpu -o %%N | head -1); [ -n "$n" ] || { echo "orcd-gpu: no running job in mit_normal_gpu; salloc one first" >&2; exit 1; }; exec nc "$n" 22'
    # No ControlMaster here; reuses orcd's master socket
    User user
```

As with the login host, you can add `ForwardAgent yes`
to forward your SSH key through the jump (e.g. for Git).

If you'd rather resolve the compute node yourself,
hard-code (or update) the one `HostName` line each session:

```none
Host orcd-cpu
    HostName PLACEHOLDER   # Node from `squeue --me` (run on a login node); update each session, e.g. node5678
    ProxyJump orcd  # Tunnel through a login node, since the compute nodes aren't directly reachable
    # No ControlMaster here; reuses orcd's master socket
    User user
```

##### Port-forwarding to a compute node

Since a compute node's loopback interface is not public
(only processes on that same node can reach it),
a web server (e.g. Jupyter, a monitoring dashboard)
bound to `localhost` on a compute node cannot be reached from a login node or your machine.
Thus, an SSH tunnel must be used to connect to the web server from your machine.
Note if you have multiple running jobs in the partition,
first make sure `orcd-cpu` resolves to the node running the web server
(see the `head -1` comment in `orcd-cpu`'s `ProxyCommand` above):

```bash
ssh -L 8888:localhost:8888 orcd-cpu   # then open http://localhost:8888
```

Slurm PAM (see [Login and Compute Nodes](#login-and-compute-nodes)) admits the tunnel
only while your job runs on that node, and it drops when the job ends.
Afterwards, tunneling attempts get rejected post-authentication with exit code 255:

> Access denied: user user (uid=123456) has no active jobs on this node.
> Access denied by pam_slurm_adopt: you have no active jobs on this node
> Connection closed by UNKNOWN port 65535

[openssh-multiplexing]: https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing
[ssh-multiplexing-lowe]: https://blog.scottlowe.org/2015/12/11/using-ssh-multiplexing/
[ssh-controlchannel]: https://orcd-docs.mit.edu/accessing-orcd/control-channels/#use-of-ssh-controlchannel

### Web Portal

The cluster's web portal is built on the open-source [Open OnDemand](https://www.openondemand.org/).
Hence the `ood` portion of <https://orcd-ood.mit.edu>.

### Slurm

For Slurm basics (job lifecycle, `sbatch`/`srun`/`squeue`), see the
[ORCD job-scheduler overview][slurm-overview].
To test Slurm connection, run `sinfo --summarize`.
Then to try out a job:

```bash
srun --time=00:01:00 --ntasks=1 --cpus-per-task=1 --mem=1G \
    bash -c 'echo "Hello world from $(hostname)."'
```

Eventually once allocated resources, you'll get: "Hello world from node1234."

[slurm-overview]: https://orcd-docs.mit.edu/running-jobs/overview/

#### Interactive allocations

From the login node (`ssh orcd`), request resources with `salloc ... --no-shell &`,
then read the assigned node using `squeue --me`
and connect to it via [`orcd-cpu`/`gpu`](#ssh-into-compute-nodes).

8 CPU cores for 6 hours on `mit_normal`:

```bash
salloc -p mit_normal -c 8 --mem=32G --time=6:00:00 --no-shell &
squeue --me   # read the node from the NODELIST column, e.g. node1234
```

GPU equivalent, 2 L40S GPUs for 6 hours on `mit_normal_gpu`:

```bash
salloc -p mit_normal_gpu -G l40s:2 -c 16 --mem=48G --time=6:00:00 --no-shell &
```

Notes:

- `--no-shell` makes `salloc` reserve the nodes and return instead of opening a shell
  on the allocation, so you connect to the node separately via the `orcd-cpu` / `orcd-gpu`
  aliases (see [SSH Into Compute Nodes](#ssh-into-compute-nodes)).
- The `&` frees your shell prompt while `salloc` waits for the nodes.
  - Because `--no-shell` leaves the allocation as a Slurm job with no attached process,
    it keeps its nodes even if your `ssh orcd` connection closes or you log out,
    until `--time` expires or you `scancel JOBID`.
- Partitions like `mit_normal` and `mit_normal_gpu` are detailed on
  [ORCD's Available Resources][available-resources] page.

#### Accounts and QoS tiers

The default tier is free and needs no extra flags:
omit `--account` / `--qos` and Slurm runs your job under account `mit_general`, QoS `normal`.
This default covers both the CPU (`mit_normal`) and GPU (`mit_normal_gpu`) partitions.

Unlike the resource-shape flags (`-p`, `-c`, `-G`, `--mem`)
covered in the general [ORCD requesting-resources docs][requesting-resources],
`--account` / `--qos` opt into an Advanced Compute Access tier.
`mit_amf_advanced_cpu` / `mit_amf_advanced_gpu` are ORCD's MIT-wide Advanced accounts.
Advanced is a paid per-account upgrade that raises priority and per-session ceilings
(see [ORCD's Compute Services][compute-services];
and pricing on ORCD's [Storage and Compute Services][storage-compute-services]).
Always pass `--qos` together with `--account`,
because the account alone does not switch the QoS.
A job submitted with only `--account=mit_amf_advanced_gpu` still runs under QoS `normal`
(observed on 7/9/2026 after hitting unexpected job queueing,
then seeing the job start immediately after resubmitting with both flags specified).
To check what you can actually use and/or what a job actually got:

```bash
sacctmgr show assoc where user=$USER format=Account,QOS,Partition  # what you can use
squeue -j JOBID -h -o "%T reason=%r"    # job state and its pending reason
sacct -j JOBID -X -o JobID,Account,QOS  # the account/QoS pair a job really has
```

Attempts to submit jobs with an account you aren't subscribed to
(e.g. `salloc --account mit_amf_advanced_gpu` without an Advanced tier)
will be rejected with:

> Invalid account or account/partition combination specified

#### `uv`-managed environments

Projects managed with [uv](https://docs.astral.sh/uv/) need care inside Slurm jobs.
Concurrent jobs running a plain `uv run` can concurrently rebuild a project's editable install,
racing for the uv cache lock (in the NFS location `~/.cache/uv`, shared across all nodes).
The latter job(s) can hit a uv lock timeout:

> Failed to acquire lock ... is another uv process running?
> You can set `UV_LOCK_TIMEOUT` to increase the timeout.

Run `uv sync` once from a login node when the code changes;
jobs otherwise should stick to `uv run --no-sync`
or activate the virtual environment directly and skip `uv run`.

### GPUs

GPUs are available on the default (free) tier. That tier is the default Slurm association:
the account `mit_general` together with the Quality of Service (QoS) `normal`.
`salloc` uses this account/QoS pair when neither `--account` nor `--qos` is specified.
The account is what marks the tier as free versus paid;
the Advanced upgrade swaps in a different account (`mit_amf_advanced_*`) and its matching QoS.

The tiers differ in the per-user ceiling on `mit_normal_gpu`
and in scheduling priority (Advanced is higher).
Advance Rentals reserve specific hardware for guaranteed access.

| Per-user ceiling on `mit_normal_gpu`           | Free tier        | Advanced tier          |
| ---------------------------------------------- | ---------------- | ---------------------- |
| GPUs                                           | 2                | 4                      |
| CPUs                                           | 32               | 64                     |
| RAM                                            | 515 GiB          | 1 TiB                  |
| Derived from `sacctmgr show qos X` (6/30/2026) | `mit_normal_gpu` | `mit_amf_advanced_gpu` |

`mit_normal_gpu` is heterogeneous; without a model specified Slurm hands you whatever's free,
which per [ORCD's requesting-resources docs][requesting-resources] defaults to an L40S.
Pin a model with `-G <type>:N` (e.g. `-G l40s:2`).

| GPU  | GPUs/node | GPU mem | CPUs/GPU budget |
| ---- | --------- | ------- | --------------- |
| L40S | 4         | 44 GB   | 16              |
| H100 | 4         | 79 GB   | 16              |
| H200 | 8         | 140 GB  | 15              |

- GPUs/node is each node's physical GPU count, not a required request size: nodes are shared
  ([Slurm's `select/cons_tres` plugin][cons_tres] plus Linux cgroups place multiple jobs on one node,
  each confined to its own disjoint slice of GPUs, CPUs, and RAM), so you can take a subset.
  - The free-tier 2-GPU ceiling still gets you L40S, H100, or H200 (up to two of them);
    it just can't claim a full 4-GPU L40S/H100 node or 8-GPU H200 node.
  - Every GPU node holds a single model, so a 2-GPU job on one node gets
    two of the same type (e.g. two L40S), never a mix (e.g. one L40S + one H100).
- GPU memory and per-node specs are from [ORCD's Available Resources][available-resources] page,
  as of 6/29/2026.
- CPUs/GPU budget is each node's CPU cores divided by its GPUs.

To discover what's actually in the pool:

```bash
sinfo -p mit_normal_gpu -N -o "%N %G %f"                     # gres + features per node
scontrol show node <node> | grep -E "State|Gres|Features|Reservation"
```

The strings after `gpu:` in `Gres` (`l40s`, `a100`, `h100`, `h200`)
are what you pass to `salloc -G <type>:N`.

For guaranteed access, ORCD offers paid Advance Rentals;
see [ORCD's Compute Services][compute-services].

[requesting-resources]: https://orcd-docs.mit.edu/running-jobs/requesting-resources/
[available-resources]: https://orcd-docs.mit.edu/running-jobs/available-resources/
[compute-services]: https://orcd-docs.mit.edu/services/compute-services/
[cons_tres]: https://slurm.schedmd.com/cons_tres.html
[storage-compute-services]: https://orcd.mit.edu/resources/storage-and-compute-services

### Filesystems

All `/orcd/*` and `/home` paths are NFS served from storage servers (not local disk),
mounted identically on every node (login and compute),
so you never need `scp`/`rsync` between them.
`/home` and `/orcd` are separate shares, not one volume:
`/home` is a single NFS share, while `/orcd` is an autofs tree
that automounts several distinct shares on demand, each from its own server
(for instance the group storage at `/orcd/compute/sendhil/001` is served over InfiniBand).
Per [ORCD's filesystems docs][filesystems], they differ in speed, size, and whether they're backed up:

| Path                        | Quota  | Backed up?                  | Use for                                       |
| --------------------------- | ------ | --------------------------- | --------------------------------------------- |
| `/home/$USER`               | 200 GB | Yes (snapshots)             | Code, configs, small artifacts                |
| `/orcd/compute/sendhil/001` | group  | No                          | Datasets, checkpoints, outputs (group-shared) |
| `/orcd/pool/$USER`          | 1 TB   | No                          | Staging for large datasets not in active use  |
| `/orcd/scratch/$USER`       | 1 TB   | No (purged after 6 mo idle) | Data a running job is actively using (fast)   |

Quotas and backup information are from [ORCD's filesystems docs][filesystems] as of 6/29/2026,
and can be verified using `quota -s`.

#### Group storage

`/orcd/compute/sendhil/001` is the group's shared allocation on one of ORCD's storage servers.
Its NFSv4.2 transport is RDMA over InfiniBand with 1 MB transfers,
which makes it faster than `/home` (plain NFS over TCP with 64 KB transfers)
for large reads and writes.

Access is controlled by the MIT Moira list [`orcd_ug_pi_sendhil_all`][moira-sendhil].
Per [ORCD's Accessing Group Resources][accessing-group-resources] docs,
that list controls the directory's on-disk Unix group.
Note that access does not propagate instantaneously:
a membership change must propagate from Moira into the cluster's Unix groups,
after which you must start a fresh login for the new group to take effect (confirm with `id`).
Afterwards, `cd /orcd/compute/sendhil/001` works directly (the path automounts on access).

Files and subdirs you create inherit that group and stay accessible to the rest of the group;
make your own `/orcd/compute/sendhil/001/$USER/`.

[filesystems]: https://orcd-docs.mit.edu/filesystems-file-transfer/filesystems/
[moira-sendhil]: https://groups.mit.edu/webmoira/list/orcd_ug_pi_sendhil_all
[accessing-group-resources]: https://orcd-docs.mit.edu/services/accessing-group-resources/

### Python (Miniforge)

ORCD provides Python (plus `conda`/`mamba`) through Miniforge modules
(see [ORCD's Python docs][orcd-python]); load it each session
(ORCD advises against `conda init` / `mamba init`):

```bash
module load miniforge
which python  # /orcd/software/core/001/pkg/miniforge/25.11.0-0/bin/python
python --version --version  # Python 3.12.12 | packaged by conda-forge | (main, Oct 22 2025, 23:25:55) [GCC 14.3.0]
```

Prefer creating virtual environments over installing packages into the conda `base` environment.
In a Slurm `sbatch` script, run `module load miniforge` (before `source .venv/bin/activate`)
inside the script itself: a batch job starts a fresh shell on the compute node
and does not inherit the module or virtualenv you loaded in an interactive login-node shell.

[orcd-python]: https://orcd-docs.mit.edu/software/python/

### Ray

[Ray's own default][ray-tmpdir-docs] session directory is `/tmp/ray`.
Note that:

1. `/tmp` is per-node (but not per user):
   each compute node has its own local `/tmp`.
2. Within each node the filesystem is shared across jobs, regardless of the job owner
   (Slurm shares nodes between users; see [GPUs](#gpus)).

So `/tmp/ray` is a fixed path that multiple users' jobs on the same node contend for,
owned by whichever user's job created it first.
To avoid cross-user collisions (in the form of a permission error),
give Ray a per-user temp directory in the job script:

```bash
export RAY_TMPDIR="/tmp/$USER-ray"
```

Then Ray sessions land under `/tmp/$USER-ray/ray/session_...`.

[ray-tmpdir-docs]: https://docs.ray.io/en/latest/ray-core/configure.html#logging-and-debugging

## Economics

See [Research Computing][mit-econ-compute] within the MIT Economics Knowledge Base.

[mit-econ-compute]: https://sites.mit.edu/econ-help/kb-category/research-computing/

## VPN

Follow the setup instructions at [Prisma Access VPN Landing Page][prisma-access]:

1. Install Prisma Access VPN Client
   - On 6/22/2026 on Windows, GlobalProtect App Version 6.2.8-431 was installed
2. Enter the portal address
3. Login with MIT Touchstone

Note the VPN won't work ("Connection Failed") when on campus with MIT SECURE Wi-Fi.

[prisma-access]: https://mit.service-now.com/esc?id=kb_article&sysparm_article=KB0030519
