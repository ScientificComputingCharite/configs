# nf-core/configs: Charité Configuration

This page documents the `charite` Nextflow profile for running Nextflow and nf-core pipelines on the Charité HPC. See the full configuration at [`conf/charite.config`](../conf/charite.config). For general HPC information, see [Charité Science IT](https://www.charite.de/en/research/research_support_services/research_infrastructure/science_it/).

> Contact: Scientific Computing, GB IT, Charité — sc-hpc-helpdesk@charite.de

This profile uses **Slurm** and **Apptainer**.

## 1. Requesting Access / Onboarding

The Charité internal user documentation/wiki requires a Charité GitLab account. To request access:
1. Email sc-hpc-helpdesk@charite.de to request access to the Charité HPC. This initiates the onboarding process.
2. If you do not have access to git.bihealth.org, contact health-data@charite.de to request GitLab access for the [Charité HPC User Documentation and Onboarding Guide](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/home).

## 2. Contributors and Acknowledgments

- Wassim Salam ([@wassimsalam01](https://github.com/wassimsalam01)) - BIH/MDC Genomics Platform Data Management Team
- Magnus Hagdorn ([@mhagdorn](https://github.com/mhagdorn)) - Scientific Computing HPC Team
- Andreas Reppas ([@areppas](https://github.com/areppas)) - Scientific Computing HPC Team

WS thanks the SC HPC Team for their support and guidance in creating this profile.

## 3. Running nf-core pipelines on the Charité HPC

> [!CAUTION]
> Do not attempt to launch Nextflow on frontend nodes. The memory on the frontends is restricted to **1 GB** per user. Your session will get killed (see [Frontend Memory Restrictions](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/faq/frontend-restrictions)). Additionally, Apptainer is not installed on frontend nodes, which will result in using the profile to fail immediately.

The Conda environment `nextflow-26.04.6` is provided by the SC HPC Team, which you can readily use.  You have 2 options:

### 3.1. Interactive Session

Log into a frontend node via your terminal as you normally would (see [Access](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#access)), then run the following:

```bash
# Adjust resources as needed
srun --job-name=run-nxf-pipeline --partition=compute --ntasks=1 --cpus-per-task=4 --mem=16G --time=12:00:00 --pty bash
```

Followed by:

```
conda activate nextflow-26.04.6
cd /path/to/your/workspace
nextflow run nf-core/<pipeline> \
    -profile charite \
    --input ./samplesheet.cv \
    --outdir ./results
```

It is also possible to jump right into an interactive session using the [Open OnDemand Portal](https://s-sc-ood.charite.de/pun/sys/dashboard/batch_connect/sessions) ([Open OnDemand Wiki](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#open-ondemand)). You must be connected the Charité network to use this, either via Ethernet cable or [OpenVPN Connect](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#connection-to-the-cluster-from-outside-the-charite-network-over-vpn).

### 3.2. Batch Script

Save the following script under `run-nxf-pipeline.sh` and submit it with `sbatch run-nxf-pipeline.sh`:

```bash
# Adjust resources as needed
#!/usr/bin/env bash
#SBATCH --job-name=run-nxf-pipeline
#SBATCH --partition=compute
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=12:00:00

set -euo pipefail
source "${HOME}/.bashrc" # Activates conda: required to activate nextflow
conda activate nextflow-26.04.6

### Export variables here ###

cd /path/to/your/workspace
nextflow run nf-core/<pipeline> \
    -r <version> \ # Optional: pin a specific version of the pipeline
    -resume \ # Optional: resume an interrupted workflow
    -profile charite \
    -params-file my-pipeline-params.yaml # Alternative method to provide inputs
```

`my-pipeline-params.yaml`

```yaml
input: ./samplesheet.csv
outdir: ./results
```

## 4. Environment Variables Setup

In order for the profile to run as intended, the following variables must first be exported before running any workflow.

> [!WARNING]
> Be mindful of what you add into your `~/.bashrc` file, as it may unexpectedly alter the desired behavior of other programs.

The code snippets may be:

1. Copy-pasted into the terminal (e.g. if working in an interactive session): Please not that with this method, you must set the variables again with every new session
2. Added to your batch script (see example above)(recommended)
3. Added to your `~/.bashrc` file: after doing so, run `source ~/.bashrc` or start a new session, in order for the variables to be exported


> [!TIP]
> <details closed>
> <summary>Click here to reveal the full Bash code snippet to be copied, in order to export all required variables</summary>
> 
> ```bash
> # Global Utility Variables
> # export PROJ_ABBR=""
> # Uncomment the line above only if you have an assigned project, and add the initials of your project
> # Leave it commented if you have no assigned project
> if [ -z "${PROJ_ABBR}" ]; then
>     export PROJ_SCRATCHDIR_CHARITE_USER="${HOME}"
> else
>     export PROJ_SCRATCHDIR_CHARITE="/sc-scratch/sc-scratch-${PROJ_ABBR}"
>     export PROJ_SCRATCHDIR_CHARITE_USER="${PROJ_SCRATCHDIR_CHARITE}/${USER}"
> fi
> 
> # Apptainer-specific Variables
> # Control where Apptainer stores cached data and temporary working files
> # Must be unique to each user, hence under ${PROJ_SCRATCHDIR_CHARITE_USER}
> # See https://apptainer.org/docs/user/main/build_env.html#cache-folders
> export APPTAINER_CACHEDIR="${PROJ_SCRATCHDIR_CHARITE_USER}/.apptainer"
> export APPTAINER_TMPDIR="${PROJ_SCRATCHDIR_CHARITE_USER}/.apptainer/tmp"
> mkdir -p "${APPTAINER_TMPDIR}" # otherwise Nextflow launch fails
> # Optional: uncomment the line below if you wish to override the default profile path where Apptainer images will be stored
> # Example path provided assigns a shared location for all collaborators of a project. Change as needed
> # export NXF_APPTAINER_CACHEDIR="${PROJ_SCRATCHDIR_CHARITE}/apptainer-images"
> 
> # Nextflow-specific Variables
> # According to nf-core: "In some cases, the Nextflow Java virtual machines can start to request a large amount of memory.
> # We recommend adding the following line to your environment to limit this"
> export NXF_OPTS='-Xms1g -Xmx4g'
> # Centralize Nextflow cache and Home into one directory
> export NXF_HOME="${PROJ_SCRATCHDIR_CHARITE_USER}/.nextflow"
> export NXF_CACHE_DIR="${NXF_HOME}"
> 
> # Temporary workaround for Nextflow strict syntax
> # Uncomment the line below, so long as the pipeline you are using is not compatible with NXF_SYNTAX_PARSER=v2 (default in Nextflow >=26.04)
> # export NXF_SYNTAX_PARSER=v1
> ```
> 
> </details>

### 4.1. Global Utility Variables

```bash
# export PROJ_ABBR=""
# Uncomment the line above only if you have an assigned project, and add the initials of your project
# Leave it commented if you have no assigned project
if [ -z "${PROJ_ABBR}" ]; then
    export PROJ_SCRATCHDIR_CHARITE_USER="${HOME}"
else
    export PROJ_SCRATCHDIR_CHARITE="/sc-scratch/sc-scratch-${PROJ_ABBR}"
    export PROJ_SCRATCHDIR_CHARITE_USER="${PROJ_SCRATCHDIR_CHARITE}/${USER}"
fi
```

Users of the Charité HPC are recommended to make use of an assigned project's scratch directory at `/sc-scratch/sc-scratch-${PROJ_ABBR}` whenever possible (see here for more info on [how to access project and scratch directories](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/HOWTOs/how-to-access-project-and-scratch-directories)).

> **No assigned project?** The setup still works by falling back to `${HOME}`, but home directories are limited to **100 GB**. Large pipelines can reach this limit quickly, so users without project scratch should monitor their home-directory usage carefully (see here for more info on [how to apply for project and scratch directories
](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/HOWTOs/how-to-apply-for-project-and-scratch-directories)).

> **You are already a group admin and want to add your collaborator to a project?** See here for more info on [how to manage project and scratch directories
](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/HOWTOs/how-to-manage-project-and-scratch-directories).

### 4.2. Apptainer-specific Variables

Apptainer requires no module loading, as it is readily available on compute nodes.

```bash
# Control where Apptainer stores cached data and temporary working files
# Must be unique to each user, hence under ${PROJ_SCRATCHDIR_CHARITE_USER}
# See https://apptainer.org/docs/user/main/build_env.html#cache-folders
export APPTAINER_CACHEDIR="${PROJ_SCRATCHDIR_CHARITE_USER}/.apptainer"
export APPTAINER_TMPDIR="${PROJ_SCRATCHDIR_CHARITE_USER}/.apptainer/tmp"
mkdir -p "${APPTAINER_TMPDIR}" # otherwise Nextflow launch fails
```

`NXF_APPTAINER_CACHEDIR` (Nextflow equivalent: `apptainer.cacheDir`) determines where Apptainer will save the built images (`*.sif`). The config here dynamically assigns this path a value, depending on if the user chooses to override the default by setting `NXF_APPTAINER_CACHEDIR`

```nextflow
apptainer {
    // ...
    cacheDir    = System.getenv('NXF_APPTAINER_CACHEDIR') ? System.getenv('NXF_APPTAINER_CACHEDIR') :
                                                            "${params.proj_scratchdir_charite_user}/apptainer-images"
    // ...
}
```

**Has `${NXF_APPTAINER_CACHEDIR}` been defined as an environment variable by the user?**

- No &rarr; defaults to `${PROJ_SCRATCHDIR_CHARITE_USER}/apptainer-images` &rarr; your folder structure will look like this:

```
/sc-scratch/sc-scratch-${PROJ_ABBR}/${USER}/
├── .apptainer
│   ├── cache
│   └── tmp
└── apptainer-images
```

- Yes &rarr; `${NXF_APPTAINER_CACHEDIR}` overrides the default path assignment.

**How is this useful?** You may set this to a custom path where you are already storing Apptainer images. Another possibility is to store built at a centralized path accessible to all collaborators of a project, if agreed upon by all. This then allows reuse of images between users and keeps unnecessary redownloading of images to the project scratch to a minimum. You can achieve this by executing:

```
export NXF_APPTAINER_CACHEDIR="${PROJ_SCRATCHDIR_CHARITE}/apptainer-images"
```

&rarr; Your folder structure would then look like this:

```
/sc-scratch/sc-scratch-${PROJ_ABBR}/${USER}/
└── .apptainer
    ├── cache
    └── tmp


/sc-scratch/sc-scratch-${PROJ_ABBR}/ # or another custom path
└── apptainer-images
```

### 4.3. Nextflow-specific Variables

```bash
# Nextflow-specific Variables
# According to nf-core:
# "In some cases, the Nextflow Java virtual machines can start to request a large amount of memory.
# We recommend adding the following line to your environment to limit this."
export NXF_OPTS='-Xms1g -Xmx4g'
# Centralize Nextflow cache and home into one directory
export NXF_HOME="${PROJ_SCRATCHDIR_CHARITE_USER}/.nextflow"
export NXF_CACHE_DIR="${NXF_HOME}"
```

Your folder structure should now look like this:

```
/sc-scratch/sc-scratch-${PROJ_ABBR}/${USER}/
├── .apptainer
│   ├── cache
│   └── tmp
├── apptainer-images
├── .nextflow
└── nxf-work # controlled by `workDir` in the config
```

> [!IMPORTANT]
> **Temporary Nextflow / nf-core compatibility workaround**
>
> Nextflow **v26.04** and higher made the **v2 syntax parser** the default as part of the major transition toward strict syntax. However, not all nf-core pipelines have completed this migration yet, and some pipelines can currently fail when parsed with `NXF_SYNTAX_PARSER=v2`. The nf-core training documentation likewise notes that many nf-core pipelines do not yet support the v2 parser.
>
> Until the affected pipelines have been migrated, set the following **temporary compatibility workaround** to force the legacy parser before launching them:
>
> ```bash
> export NXF_SYNTAX_PARSER=v1
> ```
>
> This should be removed once the pipelines you are using becomes compatible with the v2 parser.
> You can read up more about strict syntax [here](https://docs.seqera.io/nextflow/strict-syntax).

## 5. Job Assignment Logic, Resource Allocation, GPU Acceleration

Jobs will get assigned `--partition=compute` with `--qos=normal` by default. You do not need to assign resource values yoursefl to any of the tools used in pipelines; nf-core takes care of that. Tools are tagged with labels such as `process_single`, `process_high`, `process_high_memory`. Depending on the label, jobs get assigned corresponding resources. If they are insufficient, the job will be given more resources on the next attempt (`maxRetries = 5`). See [`nf-core/rnaseq`'s `base.config`](https://github.com/nf-core/rnaseq/blob/master/conf/base.config) as an example.

Some tools can benefit from Nvidia GPU acceleration (`process_gpu` label). There are 2 GPU queues available on the HPC. If a task strictly needs more than 1 GPU, it will get allocated to `pgpu`, otherwise it will get allocated to `gpu`.

## 6. Additional Profiles

The config offers additional profiles, which users can use to run their analyses. Each profile has a dedicated partition with corresponding resource limits. If you are having trouble accessing a partition you should have access to, get in touch with sc-hpc-helpdesk@charite.de.

```bash
nextflow run nf-core/<pipeline> -profile charite,<profile> --input samplesheet.csv --outdir results
```

`debug` is an additional utility profile rather than a one with a dedicated partition. It disables automatic cleanup.

```bash
nextflow run nf-core/<pipeline> -profile charite,<profile>,debug --input samplesheet.csv --outdir results
```

You can also run a pipeline with test data provided by nf-core like this:

```bash
nextflow run nf-core/demo -profile charite,test --outdir test-demo
nextflow run nf-core/rnaseq -profile charite,test --outdir test-rnaseq
```

> [!IMPORTANT]
>
> "Order matters. Profiles load in sequence. Later profiles overwrite earlier ones." See [Configuration options](https://nf-co.re/docs/running/configuration/configuration-options) for more info.
