# sge

This profile configures [Snakemake](https://snakemake.readthedocs.io/en/stable/) to run on the UCL Computer Science (Sun) Grid Engine.

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

To deploy this profile, run

	mkdir -p ~/.config/snakemake
	cd ~/.config/snakemake
	cookiecutter https://github.com/wir963/sge.git
  
  
Then, you can run Snakemake with

	snakemake --profile sge ...
  
### Cookiecutter options

* `profile_name` : A name to address the profile via the `--profile` Snakemake option.
* `cluster_config` : Path to a YAML or JSON configuration file analogues to the
  Snakemake [`--cluster-config` option](https://snakemake.readthedocs.io/en/stable/snakefiles/configuration.html#cluster-configuration-deprecated).
  This is also used to define custom resources on the SGE cluster.
  
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
