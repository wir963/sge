# sge

This profile configures [Snakemake](https://snakemake.readthedocs.io/en/stable/) to run on the UCL Computer Science (Sun) Grid Engine.

> **Deployment model (2026-07): plain git-tracked profile, not cookiecutter.**
> This repo used to be a [cookiecutter](https://cookiecutter.readthedocs.io) *template* that
> generated a profile. That caused drift: real fixes were hand-edited on the deployed copy
> (`~/.config/snakemake/sge/`) and never made it back to the template, and regenerating would
> have clobbered them. We flattened it to a plain profile whose files sit at the repo root, so
> the live profile is a **git checkout of this repo** and deploying is just `git pull` — same
> as the other repos. Do **not** reintroduce cookiecutter unless you need to generate multiple
> distinct profiles from one template (we don't).

## Change log to run on UCL CS Cluster

Don't use `sge-status.py` because it uses tools not supported by UCL cluster and may cause cluster to crash. 

## Known issues/improvements

We are currently restricting the number of cores per node to be 1 to avoid running multiple jobs on the same node when running [grouped jobs](https://snakemake.readthedocs.io/en/stable/executing/grouping.html). This is to solve errors when I download multiple files from SRA at the same time on the same node but this is also inefficient. 

We are currently able to request local temp storage for jobs using `tscratch` (use an integer number for GB; most resources take integers for MB) but allocating the temp storage (and deleting it after the job is done) must be done within the job [HPC docs](https://hpc.cs.ucl.ac.uk/data-storage/) like below. 

```
rule example:
    resources:
        time="12:00:00",
        mem_mb=64000,
        tscratch=60,
    shell:
        "mkdir -p /scratch0/wrobinso/$JOB_ID \n"
        "<INSERT COMMAND>"
        'trap "rm -rf /scratch0/wrobinso/$JOB_ID" EXIT ERR INT TERM'
``` 

## Setup

### Deploy profile

To deploy this profile, clone it to the Snakemake profile search path (the directory name
is what you pass to `--profile`):

	mkdir -p ~/.config/snakemake
	git clone git@github.com:wir963/sge.git ~/.config/snakemake/sge

Then run Snakemake with

	snakemake --profile sge ...

To pick up later changes, `cd ~/.config/snakemake/sge && git pull`. Edit the profile locally,
commit, push, and pull on each machine that uses it.

### Multithreading (SMP) and memory

`sge-submit.py` translates a rule's `threads:` into an SMP reservation and derives both memory
limits from `mem_mb` so they stay consistent and slot-aware:

* `threads: N` (N > 1) -> `-pe smp N -R y` (reserve N slots on one node, with resource
  reservation). This requires `--cores >= N` in the launcher, or Snakemake clamps `threads`.
* `h_vmem` (SGE `consumable=NO`, a per-**process** ceiling) = `mem_mb` in full, so one fat
  process isn't killed under SMP.
* `tmem` (SGE `consumable=YES, FORCED`, per-**slot**) = `ceil(mem_mb / threads)`, so
  `tmem * threads == mem_mb` total and the job schedules. At `threads=1` the two are equal.
  
### Default snakemake arguments
Default arguments to ``snakemake`` maybe adjusted in the ``<profile path>/config.yaml`` file.

### Cluster Files

Per rule configuration can be defined in a cluster file and passed in using --cluser-config. This is a yaml file where the key is the rule name followed by a list of SGE settings to add or override settings set in the _profile_. You can also add options to the `__default__` config. **NOTE that these are _ADDED_ to the default and will be inheritted by any named rules.**

An example local cluster config file (`cluster.yaml`) looks like:

```
__default__
	q: private.q
	
rule1:
	gpu:1
	
rule2:
	time: "4:0:0"
```

which will be used by specifying `snakemake --profile sge --cluster-config cluster.yaml`.

## Parsing arguments to SGE (qsub)
Arguments are overridden in the following order, aliases are also defined and can be defined :

1) `QSUB_DEFAULTS` in `sge-submit.py`
2) Profile `cluster_config` file `__default__` entries
3) Snakefile threads and resources (time, mem)
4) Profile `cluster_config` file <rulename> entries
5) `--cluster-config` parsed to Snakemake (deprecated since Snakemake 5.10)

## Resource and option mapping

To allow more expressive resource requests we map some simple names to the SGE options and resources. These can be used for example in `cluster.yaml` to make the configuration simpler to read.

### Notes

Custom SGE resources can be specified in `__resources__` only in the profile folder (i.e. any `__resources__` in a local `--cluster-config cluster.yaml` will be ignored, but you can request the resources defined in the global profile). Custom resources are specified as a YAML dictionary where the key is the resource name as defined in SGE and the values are any aliases you want to use for this resource. The key will always be avaiable as a name even if you don't specifiy it as an alias. If a key already exists in the resource list the the aliases are just appended to that resource. 

For example:

```
__resources__:
  coproc_v100: 
    - "gpu"
    - "nvidia_gpu"
```

Allows you to request with `coproc_v100=1`, `gpu=1` or `nvidia_gpu=1` in the cluster config files or snakemake rule resources all of which will actually set `-l coproc_v100=1` for qsub.

Memory (`s_vmem`, `h_vmem` and aliases) must be given in **megabytes** (NOTE: this is to support snakemake version >= 7 which sets a default `mem_mb` resource. In older versions of the grid engine profile the memory was in gigabytes).


Custom SGE options can be specified in `__options__` in the profile folder in the same way as resources.  

For example:

```
__options__:
  jc: 
    - "jc"
    - "job_class"
```


A full list of the default supported SGE options and resource requests with their aliases is:


| SGE Option       | Accepted aliases                             |
| -----------------|----------------------------------------------| 
| binding          | binding                                      |
| cwd              | cwd,                                         |
| e                | e, error                                     |
| hard             | hard,                                        |
| j                | j, join                                      |
| m                | m, mail_options                              |
| M                | M, email                                     |
| notify           | notify,                                      |
| now              | now,                                         |
| N                | N, name                                      |
| o                | o, output                                    |
| P                | P, project                                   |
| p                | p, priority                                  |
| pe               | pe, parallel_environment                     |
| pty              | pty,                                         |
| q                | q, queue                                     |
| R                | R, reservation                               |
| r                | r, rerun                                     |
| soft             | soft,                                        |
| v                | v, variable                                  | 
| V                | V, export_env                                |
| qname            | qname,                                       |
| hostname         | hostname,                                    |
| calendar         | calendar,                                    |
| min_cpu_interval | min_cpu_interval,                            |
| tmpdir           | tmpdir,                                      |
| seq_no           | seq_no,                                      |
| s_rt             | s_rt, soft_runtime, soft_walltime            |
| h_rt             | h_rt, time, runtime, walltime                |
| s_cpu            | s_cpu, soft_cpu                              |
| h_cpu            | h_cpu, cpu                                   |
| s_data           | s_data, soft_data                            |
| h_data           | h_data, data                                 |
| s_stack          | s_stack, soft_stack                          |
| h_stack          | h_stack, stack                               |           
| s_core           | s_core, soft_core                            |
| h_core           | h_core, core                                 |
| s_rss            | s_rss, soft_resident_set_size                |
| h_rss            | h_rss, resident_set_size                     |
| slots            | slots,                                       |
| s_vmem           | s_vmem, soft_memory,  soft_virtual_memory    | 
| h_vmem           | h_vmem, mem_mb, mem, memory,  virtual_memory | 
| s_fsize          | s_fsize, soft_file_size                      |
| h_fsize          | h_fsize, disk_mb, file_size                  |

## Non Requestable Resources

On some cluster configurations some resources may be non-requestable. 
