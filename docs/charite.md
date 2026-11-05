# nf-core/configs: Charité Configuration

This page documents the `charite` Nextflow profile for running Nextflow and nf-core pipelines on the Charité HPC. See the full configuration at [`conf/charite.config`](../conf/charite.config). For general HPC information, see [Charité Science IT](https://www.charite.de/en/research/research_support_services/research_infrastructure/science_it/).

> Official Contact: Scientific Computing, GB IT, Charité - sc-hpc-helpdesk@charite.de

## Table of Contents

- [Contributors and Acknowledgments](#contributors-and-acknowledgments)
- [Overview](#overview)
- [Requesting Access](#requesting-access)
- [Running Jobs](#running-jobs)
  - [Option 1: Interactive Session](#option-1-interactive-session)
  - [Option 2: Batch Script](#option-2-batch-script) 
- [Environment Setup](#environment-setup)
  - [Global Utility Variables](#global-utility-variables)
  - [Apptainer-specific Variables](#apptainer-specific-variables)
  - [Nextflow-specific Variables](#nextflow-specific-variables)
  - [Nextflow Conda Environment](#nextflow-conda-environment)
- [Running nf-core pipelines](#running-nf-core-pipelines)
- [Job Assignment Logic, Resource Allocation, GPU Acceleration](#job-assignment-logic-resource-allocation-gpu-acceleration)
- [Additional Profiles](#additional-profiles)

## Contributors and Acknowledgments

- Developer: Wassim Salam - BIH/MDC Genomics Platform Data Management Team
- Maintainers: Magnus Hagdorn & Andreas Reppas - Scientific Computing HPC Team

WS ([@wassimsalam01](https://github.com/wassimsalam01)) thanks the following individuals for their support and guidance in developing the profile:

- Scientific Computing HPC Team ([@ScientificComputingCharite](https://github.com/ScientificComputingCharite))
    - Magnus Hagdorn ([@mhagdorn](https://github.com/mhagdorn))
    - Andreas Reppas ([@areppas](https://github.com/areppas))
- BIH/MDC Genomics Platform Data Management Team ([@BIH-MDC-Genomics-Platform](https://github.com/BIH-MDC-Genomics-Platform))
    - Marten Jäger ([@martenj](https://github.com/martenj))
    - Cornelius Fischer ([@cofischer](https://github.com/cofischer))
    - Namuun Battur ([@NmnBttr](https://github.com/NmnBttr))

## Overview

- **Scheduler**: SLURM
- **Default Queue**: `compute`
- **Default QOS**: `normal`
- **Container Engine**: Apptainer
- **Resource Limits of Default Queue**:
  - Maximum CPUs: 192 cores
  - Maximum memory: 1TB
  - Maximum walltime: 2 days
- **Usage Examples**
 
```
# nf-core Pipeline
nextflow run nf-core/<pipeline-name> -profile charite ...

# Your own Nextflow Workflow
nextflow run main.nf -profile charite ...
```

- **Additional Queues**: `dragen_cc05`,`dragen_cc15`,`dragen_gp`,`highmem`,`genomics` (see []() for more info
- **GPU Queues**: `gpu`,`pgpu` (see []() for more info)

## Requesting Access

The Charité internal user documentation/wiki requires a Charité GitLab account. To request access:
1. Email sc-hpc-helpdesk@charite.de to request access to the Charité HPC. This initiates the onboarding process.
2. If you do not have access to git.bihealth.org, contact health-data@charite.de to request GitLab access for the [Charité HPC User Documentation and Onboarding Guide](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/home).

## Running Jobs

> [!CAUTION]
> Do not attempt to start resource-heavy jobs on frontend nodes. The memory on the frontends is restricted to **1 GB** per user. Your session will get killed (see [Frontend Memory Restrictions](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/faq/frontend-restrictions)).

First, `ssh` into a frontend node via your terminal as you normally would (see [Access](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#access)). You then have 2 options:

### Option 1: Interactive Session

```bash
srun --job-name=run-nf-core --partition=compute --ntasks=1 --cpus-per-task=1 --mem=1G --time=08:00:00 --pty bash
```

It is also possible to jump right into an interactive session using the [Open OnDemand Portal](https://s-sc-ood.charite.de/pun/sys/dashboard/batch_connect/sessions) ([Open OnDemand Wiki](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#open-ondemand)). You must be connected the Charité network to use this, either via Ethernet cable or [OpenVPN Connect](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#connection-to-the-cluster-from-outside-the-charite-network-over-vpn).

### Option 2: Batch Script

Create a new file and name it `test.sh`:

```bash
#!/usr/bin/env bash
#SBATCH --job-name=test
#SBATCH --partition=compute
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --mem=1G
#SBATCH --time=00:01:00

echo "Hello World!"
```

Submit with `sbatch test.sh`. Examine the output under `slurm-${SLURM_JOB_ID}.out`.

## Environment Setup

> [!NOTE]
> In order for the nf-core config to work as intended, some variables must first be exported into the environment before running any workflow. This section provides a detailed overview of those variables.
> In Section [4. Running nf-core pipelines](#4-running-nf-core-pipelines), a complete practical example is provided, integrating the exporting of the variables with an actual run of a nf-core pipeline.

The code snippets may be:

1. Copy-pasted into the terminal (e.g. if working in an interactive session). With this method, you must set the variables again every time you start a new session
2. Added to your Batch script
3. Added to your `~/.bashrc` file. After doing so, run `source ~/.bashrc` or start a new session, in order for the variables to be exported. You then no longer need to export them every time

> [!WARNING]
> Be mindful of what you add into your `~/.bashrc` file, as it may unexpectedly alter the desired behavior of other programs.

### Global Utility Variables

```bash
export PROJ_ABBR="" # Add the abbreviation of your project here if you have one
if [ -z "${PROJ_ABBR:-}" ]; then
    export PROJ_SCRATCHDIR_CHARITE_USER="${HOME}"
else
    export PROJ_SCRATCHDIR_CHARITE="/sc-scratch/sc-scratch-${PROJ_ABBR}"
    export PROJ_SCRATCHDIR_CHARITE_USER="${PROJ_SCRATCHDIR_CHARITE}/${USER}"
fi

if [ -z "${PROJ_SCRATCHDIR_CHARITE_USER:-}" ]; then # Fail-safe mechanism, since profile relies on this variable
  echo "[ERROR] Variable PROJ_SCRATCHDIR_CHARITE_USER has not been set. Shutting down..."
  exit 1
fi
```

Users of the Charité HPC are recommended to make use of an assigned project's scratch directory at `/sc-scratch/sc-scratch-${PROJ_ABBR}` whenever possible (see here for more info on [how to access project and scratch directories](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/HOWTOs/how-to-access-project-and-scratch-directories)).

> **No assigned project?** The setup still works by falling back to `${HOME}`, but home directories are limited to **100 GB**. Large pipelines can reach this limit quickly, so users without project scratch should monitor their home-directory usage carefully (see here for more info on [how to apply for project and scratch directories
](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/HOWTOs/how-to-apply-for-project-and-scratch-directories)).

> **You are already a group admin and want to add your collaborator to a project?** See here for more info on [how to manage project and scratch directories
](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/HOWTOs/how-to-manage-project-and-scratch-directories).

### Apptainer-specific Variables

Apptainer requires no module loading, as it is readily available on compute nodes.

```bash
# Control where Apptainer stores cached data and temporary working files
# Must be unique to each user, hence under ${PROJ_SCRATCHDIR_CHARITE_USER}
# See https://apptainer.org/docs/user/main/build_env.html#cache-folders
export APPTAINER_CACHEDIR="${PROJ_SCRATCHDIR_CHARITE_USER}/.apptainer"
export APPTAINER_TMPDIR="${PROJ_SCRATCHDIR_CHARITE_USER}/.apptainer/tmp"
mkdir -p "${APPTAINER_TMPDIR}" # otherwise Nextflow launch fails
```

`NXF_APPTAINER_CACHEDIR` (Nextflow equivalent: `apptainer.cacheDir`) determines where Apptainer will save the built images (`*.img`). The config here dynamically assigns this path a value, depending on if the user chooses to override the default by setting `NXF_APPTAINER_CACHEDIR`

```nextflow
apptainer {
    // ...
    cacheDir    = System.getenv("NXF_APPTAINER_CACHEDIR") ? System.getenv("NXF_APPTAINER_CACHEDIR") :
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

### Nextflow-specific Variables

```bash
# Nextflow-specific Variables
# According to nf-core:
# "In some cases, the Nextflow Java virtual machines can start to request a large amount of memory.
# We recommend adding the following line to your environment to limit this."
export NXF_OPTS="-Xms1g -Xmx4g"
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

### Nextflow Conda Environment

The Conda environment `nf-core-2026-1` is provided by the SC HPC Team, which you can readily use. It contains both the nf-core CLI (v4.2.0) as well as Nextflow (v26.10.4)

```bash
conda activate nf-core-2026-1
```

## Running nf-core pipelines

In order to run nf-core pipelines with the Charité HPC config, you must follow one of the 2 options highlighted in [2. Getting started with running jobs](#2-getting-started-with-running-jobs). This is because Apptainer is not installed on frontend nodes, which will result in using the profile to fail immediately.

Here is an example with a Batch script. In your workspace, save the following script under `nf-core-demo-test.sh` and submit it with `sbatch nf-core-demo-test.sh`:

```bash
#!/usr/bin/env bash
#SBATCH --job-name=run-nf-core-demo-test
#SBATCH --partition=compute
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=02:00:00

# Activate Conda & Nextflow
set +e
source /etc/profile.d/conda.sh
conda activate nextflow-26.04.6
set -e

# 1. Global Utility Variables
export PROJ_ABBR="" # Add the abbreviation of your project here if you have one
if [ -z "${PROJ_ABBR:-}" ]; then
    export PROJ_SCRATCHDIR_CHARITE_USER="${HOME}"
else
    export PROJ_SCRATCHDIR_CHARITE="/sc-scratch/sc-scratch-${PROJ_ABBR}"
    export PROJ_SCRATCHDIR_CHARITE_USER="${PROJ_SCRATCHDIR_CHARITE}/${USER}"
fi

if [ -z "${PROJ_SCRATCHDIR_CHARITE_USER:-}" ]; then # Fail-safe mechanism, since profile relies on this variable
    echo "[ERROR] Variable PROJ_SCRATCHDIR_CHARITE_USER has not been set. Shutting down..."
    exit 1
fi

# 2. Apptainer-specific Variables
# Control where Apptainer stores cached data and temporary working files
export APPTAINER_CACHEDIR="${PROJ_SCRATCHDIR_CHARITE_USER}/.apptainer"
export APPTAINER_TMPDIR="${PROJ_SCRATCHDIR_CHARITE_USER}/.apptainer/tmp"
mkdir -p "${APPTAINER_TMPDIR}" # otherwise Nextflow launch fails

# Optional: uncomment the line below if you wish to override the default profile path where Apptainer images will be stored
# Example path provided assigns a shared location for all collaborators of a project. Change as needed
# export NXF_APPTAINER_CACHEDIR="${PROJ_SCRATCHDIR_CHARITE}/apptainer-images"

# 3. Nextflow-specific Variables
# Recommendation by nf-core
export NXF_OPTS="-Xms1g -Xmx4g"
# Centralize Nextflow cache and Home into one directory
export NXF_HOME="${PROJ_SCRATCHDIR_CHARITE_USER}/.nextflow"
export NXF_CACHE_DIR="${NXF_HOME}"

# Temporary workaround for Nextflow strict syntax
# Add the line below, so long as the pipeline you are using is not compatible with NXF_SYNTAX_PARSER=v2 (default in Nextflow >=26.04)
# This must eventually be removed!
export NXF_SYNTAX_PARSER=v1

# Do the work
nextflow run nf-core/demo \
    -profile charite \
    --outdir demo-test

    # -r <version> \ # Optional: pin a specific revision of a pipeline
    # -resume \ # Optional: resume an interrupted workflow
    # -c custom-params.config \ # Optional: add an extra config file which contains additional user-defined parameters
```

## Job Assignment Logic, Resource Allocation, GPU Acceleration

Jobs will get assigned `--partition=compute` with `--qos=normal` by default. You do not need to assign resource values yourself to any of the tools used in the pipelines; nf-core takes care of that. Tools are tagged with labels such as `process_single`, `process_high`, `process_high_memory`. Depending on the label, jobs get assigned corresponding resources. If they are insufficient, the job will be given more resources on the next attempt (`maxRetries = 5`). See [`nf-core/rnaseq`'s `base.config`](https://github.com/nf-core/rnaseq/blob/master/conf/base.config) as an example.

Some tools can benefit from Nvidia GPU acceleration (`process_gpu` label). There are 2 GPU queues available on the HPC. If a task strictly needs more than 1 GPU, it will get allocated to `pgpu`, otherwise it will get allocated to `gpu`.

## Additional Profiles

The config offers additional profiles, which users can use to run their analyses. Each profile has a dedicated partition with corresponding resource limits. If you are having trouble accessing a partition you should have access to, get in touch with sc-hpc-helpdesk@charite.de.

```bash
nextflow run nf-core/<pipeline> -profile charite,<profile> --input samplesheet.csv --outdir results
```

| Profile  | dragen_gp  | dragen_cc05  | dragen_cc15  |  highmem | genomics |
|---|---|---|---|---|----|
| Partition  |  dragen_gp | dragen_cc05  |  dragen_cc15 |  highmem |  genomics |
|  Max CPUs |  48 | 64  |  64 | 192  |  192 |
|  Max Memory | 240.GB  |  500.GB |  500.GB | 4.TB  |  1.TB |
|  Max Time |  7.d | 7.d  | 7.d  |  2.d |  7.d |


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
