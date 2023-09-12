# Hannah's ultimate guide to all useful things Snakemake

## What is snakemake?
Snakemake is a program that can be used to string together workflows in a way that is reproduceable and shareable. It essentially defines a set of rules (or steps) and the dependencies of those steps.

## Installation and getting started
Snakemake can be run in a conda environment.

To install create a file called `environment.yml` that has the following contents:
```yml
name: snakemake-minimal
channels:
  - conda-forge
  - bioconda
dependencies:
  - snakemake-minimal >=7.3
  - cookiecutter # optional but good for setting up cluster environment
```
Next use the environment file as instructiosn to conda to create a snakemake conda environment.

```bash
module load anaconda/colsa # Load the conda module on preimse
conda env create -f environment.yml # creates the environment
```

After the installation is complete, you can activate the snakemake environment like this:

```bash
conda activate snakemake-minimal
```

## Running with slurm

To run snakemake on a cluster environment, you first need to set up a profile that gives snakemake guidance about how to run the program.

### 1) Set the default profile [optional but recommended]
To start we can make a default profile that provides specifications for launching tasks on the login node. Since no major computation should be done on the login node, these should be fairly bare-bones.

We'll put these files in a folder located under the hidden folder `~/.config/snakemake`,  as this is where snakemake looks for profiles by default. *Note: If you have any sort of cluster_config.yml file in your project directory, it will take precedence over the files here leading to odd interactions.*

```bash
mkdir -p ~/.config/snakemake/default # creates folder for default configuration
```

Using nano ([intro to nano here](https://linuxize.com/post/how-to-use-nano-text-editor/)), or your favorite commandline text editor, open a file in this folder to specify defaults:

```bash
nano ~/.config/snakemake/default/config.yaml
```

```yml
# non-cluster defaults; to be run when on the login node
restart-times: 3
local-cores: 1
latency-wait: 5
use-conda: True
jobs: 1
keep-going: True
rerun-incomplete: True
printshellcmds: True
```
Type `ctrl + O` to save changes, `Enter` to accept the changes, and `Ctrl + X` to exit nano.  

### 2) Set a basic slurm profile

While you can set the basic slurm profile by hand in the same way the default profile was set, lovely people on the internet have created some really nice template tools that allow snakemake to interact with slurm behind the scenes. We will use the tool "cookiecutter" to help download these templates and customize them to premise.

Load the snakemake conda environment:
```bash
module load anaconda/colsa
conda activate snakemake-minimal
```

Setup the configuration
```bash
mkdir -p ~/.config/snakemake
cd ~/.config/snakemake
cookiecutter https://github.com/Snakemake-Profiles/slurm.git
```

The `cookiecutter` command will download the slurm profile folder into your `~/.config/snakemake` folder.

Cookiecutter will then lead you through some interactive settings. 

Here we will mimic settings in Toni's premise sbatch template `threaded.slurm`. Type the commands or text inside the triangle brackets <> and then press enter (don't worry if you type some wrong, or make mistakes, we can always modify them later):
```bash
(snakemake-minimal) [userid@login01 snakemake]$ cookiecutter https://github.com/Snakemake-Profiles/slurm.git
  [1/17] profile_name (slurm): <slurm.threaded>
  [2/17] Select use_singularity
    1 - False
    2 - True
    Choose from [1/2] (1): <2>
  [3/17] Select use_conda
    1 - False
    2 - True
    Choose from [1/2] (1): <2>
  [4/17] jobs (500): <Enter>
  [5/17] restart_times (0): <3>
  [6/17] max_status_checks_per_second (10): <Enter>
  [7/17] max_jobs_per_second (10): <Enter>
  [8/17] latency_wait (5): <Enter>
  [9/17] Select print_shell_commands
    1 - False
    2 - True
    Choose from [1/2] (1): <2>
  [10/17] sbatch_defaults (): <account=ernakovich>
  [11/17] cluster_sidecar_help (Use cluster sidecar. NB! Requires snakemake >= 
7.0! Enter to continue...): <Enter>
  [12/17] Select cluster_sidecar
    1 - yes
    2 - no
    Choose from [1/2] (1): <2>
  [13/17] cluster_name (): <Enter>
  [14/17] cluster_jobname (%r_%w): <smk-%r-%w>
  [15/17] cluster_logpath (logs/slurm/%r/%j): <logs/slurm/%r/%r_%w-%j>
  [16/17] cluster_config_help (The use of cluster-config is discouraged. Rather, set snakemake CLI options in the profile configuration file (see snakemake documentation on best practices). Enter to continue...): <Enter>
  [17/17] cluster_config (): <Enter>

```

After running this, you'll have a folder called slurm.threaded that contains several files:
```bash
(snakemake-minimal) [userid@login01 snakemake]$ ls slurm.threaded/
config.yaml  CookieCutter.py  settings.json  slurm-jobscript.sh  slurm-sidecar.py  slurm-status.py  slurm-submit.py  slurm_utils.py

```

The basic settings you just entered are in `config.yml`. If you've made a mistake or want to change any of them, add it to the config.yml file. *Note: not all of the settings you responded to above are in this file initially, but they can still be changed. Checkout the [template github](https://github.com/Snakemake-Profiles/slurm) for more information about specific parameters.

```bash
(snakemake-minimal) [userid@login01 snakemake]$ cat slurm.threaded/config.yaml

cluster-cancel: "scancel"
restart-times: "3"
jobscript: "slurm-jobscript.sh"
cluster: "slurm-submit.py"
cluster-status: "slurm-status.py"
max-jobs-per-second: "10"
max-status-checks-per-second: "10"
local-cores: 1
latency-wait: "5"
use-conda: "True"
use-singularity: "True"
jobs: "500"
printshellcmds: "True"

```

## Building a workflow

### Rules and directives
The Snakefile contains your workflow by default (a different file can be specified with the `-s` option when invoking snakemake). Each step in the workflow is defined by a *rule* (here called "NAME") that is made up of several *directives* (here called "input", "output", and "shell"), and takes the format below:

```yaml
rule NAME:
    input:
        in1="path/to/inputfile",
        in2="path/to/other/inputfile"
    output:
        out1="path/to/outputfile"
    shell:
        "shellcommand {input.in1} {input.in2} > {output.out1}"
```

Below is a (non-comprehensive) list of possible directives and their purpose (fundamental ones marked with an asterisk):
  1. *`input` - A list of input files and/or directories for your action (comma-separated!!); to specify a directory, surround it with `directory("path/to/directory")`
  2. *`output` - the list of output files and/or directories for your action (comma-separated!!); to specify a directory, surround it with `directory("path/to/directory")`
  3. *`shell` the shell command you wish to execute. Must be inside quotes.
  4. *`resources` - a list of resources for the job to use. Use for things like threads, memory, running time, etc.
  5. *`threads` - another way to specify the number of threads each job should use.
  6. `log` - specifies location of log file. 
  7. `run` and `script` - either python code to run in lieu of the `shell` directive, or the location of a script to run in lieu of the `shell` directive. For example, an R script might use the `script` directive
  8. `wildcard_constraints` - uses regular expressions to specify the kinds of characters or pattern to expect for different sets of wildcards (see wildcard section below). This can help prevent ambiguation when using multiple sets of wildcards. More on regular expressions [here](https://docs.python.org/3/library/re.html) and even more detail [here](https://docs.python.org/3/library/re.html).
  9. `message` - sends a message to the console about the rule as it is being run.
  10. `priority` - For rules that can be executed simultaneously, priority will allow you to set the priority of which should be sent ot the job scheduler first.
  11. `conda` - a path to a .yml environment file specifying the conda environment in which the rule should be run. Pariticularly useful if different rules need different kinds of software.
  12. `retries` - specifies a number of retries for rules you expect may fail due to outside issues. For example a download.
  13. `localrule` - specifies `True` or `False` whether or not a rule needs to be run on the cluster or can be run locally


## Extra formulas to specify in rules
 1. `temp()` - surround a temporary output file by `temp()` to remove it after all it's dependencies are created.
 2. `protected()` - the opposite of `temp()` surround an output file that took a long time to generate, or shouldn't be modified by `protected()` to write-protect it after it has been generated.
 3. `ensure()` - ensure properties of a file. For example checksum compliance or that a file is not empty. (Often useful for downloads or for programs that may generate empty fastqs without an error)
 4. `touch()` - can be used to create an empty file that a rule 'depends' on when the rule generates no output files but the task still needs to be marked as completed for the workflow.



## Dry run

```bash snakemake -np <outputfile>```

- -n is dry run
- -p is print the commands being run

## Using different software modules on HPC system

To define a rule using software from an anaconda/colsa module:
```bash
rule NAME:
    input:
        in1="path/to/inputfile",
        in2="path/to/other/inputfile"
    output:
        out1="path/to/outputfile",
        out2="path/to/another/outputfile"
    shell:
        """
        module load anaconda/colsa
        conda activate xxx
        Rscript path/to/script.R {input.in1} {input.in2} {output.out1} {output.out2}
        """
```

To define a rule using software from an linuxbrew/colsa module:
```bash
rule NAME:
    input:
        in1="path/to/inputfile",
    output:
        out1="path/to/outputdir",
    shell:
        """
        module load linuxbrew/colsa
        fastqc {input.in1} -o {output.out1} -t {threads}
        """
```

## Resource specification

## Shortcuts and wildcards
Brackets {} are used to call out wildcards

## Vocabulary
- DAG - "Directed acyclic graph" - the path of jobs where edges are dependencies and nodes are jobs/rules


## Setting up for running on slurm
To run on slurm you'll need a configuration profile. 

There's a simple profile here we'll base ours off of.

```yaml
cluster:
  mkdir -p logs/{rule} &&
  sbatch
    --partition={resources.partition}
    --qos={resources.qos}
    --cpus-per-task={threads}
    --mem={resources.mem_mb}
    --job-name=smk-{rule}-{wildcards}
    --output=logs/{rule}/{rule}-{wildcards}-%j.out
default-resources:
  - partition=<name-of-default-partition>
  - qos=<name-of-quality-of-service>
  - mem_mb=1000
restart-times: 3
max-jobs-per-second: 10
max-status-checks-per-second: 1
local-cores: 1
latency-wait: 60
jobs: 500
keep-going: True
rerun-incomplete: True
printshellcmds: True
scheduler: greedy
use-conda: True
```