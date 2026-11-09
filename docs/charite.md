# nf-core/configs: Charité Configuration

This page documents the `charite` Nextflow profile for running Nextflow and nf-core pipelines on the Charité HPC. See the full configuration at [`conf/charite.config`](../conf/charite.config). For general HPC information, see [Charité Science IT](https://www.charite.de/en/research/research_support_services/research_infrastructure/science_it/).

> Official Contact: Scientific Computing, GB IT, Charité - sc-hpc-helpdesk@charite.de

## Table of Contents

- [Contributors and Acknowledgments](#contributors-and-acknowledgments)
- [Quick Overview and Usage Examples](#quick-overview-and-usage-examples)
- [Requesting Charité HPC Access](#requesting-hpc-access)
- [Environment Setup](#environment-setup)
  - [Redirecting Output to Scratch](#redirecting-output-to-scratch)
  - [Global Utility Variables](#global-utility-variables)
  - [Apptainer-specific Variables](#apptainer-specific-variables)
  - [Nextflow-specific Variables](#nextflow-specific-variables)
  - [Nextflow Strict Syntax](#nextflow-strict-syntax)
- [Running Nextflow Workflows on the Charité HPC](#running-nextflow-workflows-on-the-charité-hpc)
  - [Conda Environment](#conda-environment)
  - [Option 1: Interactive Session](#option-1-interactive-session)
  - [Option 2: Batch Script](#option-2-batch-script) 
- [Job Assignment Logic, Resource Allocation, GPU Acceleration](#job-assignment-logic-resource-allocation-gpu-acceleration)
- [Known Limitations and Issues](#known-limitations-and-issues)

## Contributors and Acknowledgments

- Developer: Wassim Salam - BIH/MDC Genomics Platform Data Management Team
- Maintainers: Magnus Hagdorn & Andreas Reppas - Scientific Computing HPC Team

[@wassimsalam01](https://github.com/wassimsalam01) thanks the following individuals for their support and guidance in developing the profile:

- Scientific Computing HPC Team ([@ScientificComputingCharite](https://github.com/ScientificComputingCharite))
    - Magnus Hagdorn ([@mhagdorn](https://github.com/mhagdorn))
    - Andreas Reppas ([@areppas](https://github.com/areppas))
- BIH/MDC Genomics Platform Data Management Team ([@BIH-MDC-Genomics-Platform](https://github.com/BIH-MDC-Genomics-Platform))
    - Marten Jäger ([@martenj](https://github.com/martenj))
    - Cornelius Fischer ([@cofischer](https://github.com/cofischer))
    - Namuun Battur ([@NmnBttr](https://github.com/NmnBttr))

## Quick Overview and Usage Examples

- **Scheduler**: SLURM
- **Container Engine**: Apptainer
- **Default QOS**: `normal`
- **Default Queue**: `compute`
- **Resource Limits of Default Queue**:
  - Maximum CPUs: 192 cores
  - Maximum memory: 1TB
  - Maximum walltime: 2 days
- **GPU Queues**: `gpu`,`pgpu`
- **Additional Profiles**: `dragen_cc05`,`dragen_cc15`,`dragen_gp`,`highmem`,`genomics`

See [Job Assignment Logic, Resource Allocation, GPU Acceleration](#job-assignment-logic-resource-allocation-gpu-acceleration) for more info.

- **Usage Examples**:

```
# Your own Nextflow Workflow
nextflow run main.nf -profile charite ...
```

```
# nf-core Pipeline
nextflow run nf-core/<pipeline-name> -profile charite ...
```

```
# Test nf-core Demo Pipeline (small-scale workflow)
# Specifying '-profile test' uses dummy data provided by nf-core for that particular pipeline
nextflow run nf-core/demo -profile test,charite --outdir results/demo-test
```

## Requesting Charité HPC Access

The Charité internal user documentation/wiki requires a Charité GitLab account. To request access:
1. Email sc-hpc-helpdesk@charite.de to request access to the Charité HPC. This initiates the onboarding process.
2. If you do not have access to [git.bihealth.org](https://git.bihealth.org/), contact health-data@charite.de to request GitLab access for the [Charité HPC User Documentation and Onboarding Guide](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/home).

## Environment Setup

> [!NOTE]
> In order for the config to work as intended, some variables must first be exported into the environment before running any workflow. This section provides a detailed overview of those variables.
>
> A complete practical example is provided in the next section, which integrates the exporting of the variables explained here with an actual run of a nf-core pipeline. [Click here to skip ahead to 'Running Nextflow Workflows on the Charité HPC'](#running-nextflow-workflows-on-the-charité-hpc)

### Redirecting Output to Scratch

The Charité HPC profile uses project-specific scratch storage for Nextflow and Apptainer files whenever available. Storage paths are determined automatically based on the `PROJ_ABBR` environment variable:

- **If `PROJ_ABBR` is non-empty:** `PROJ_SCRATCHDIR_CHARITE_USER` is set to `/sc-scratch/sc-scratch-${PROJ_ABBR}`.
- **If `PROJ_ABBR` is empty:** `PROJ_SCRATCHDIR_CHARITE_USER` falls back to `${HOME}`, which has a **100 GB storage quota**.

The profile requires `PROJ_SCRATCHDIR_CHARITE_USER` to be defined and will fail if it is missing.

**Using project scratch storage is strongly recommended**, particularly for large pipelines, as Apptainer caches and Nextflow working directories can quickly exhaust the home-directory quota.

For more information on managing project and scratch storage, refer to the Charité HPC documentation:

- **Existing project members:** [How to access project and scratch directories](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/HOWTOs/how-to-access-project-and-scratch-directories)
- **Users without an assigned project:** [How to apply for project and scratch directories](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/HOWTOs/how-to-apply-for-project-and-scratch-directories)
- **Project administrators:** [How to manage project and scratch directories](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/HOWTOs/how-to-manage-project-and-scratch-directories)


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
    cacheDir    = System.getenv("NXF_APPTAINER_CACHEDIR") ?: "${params.proj_scratchdir_charite_user}/apptainer-images"
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
# Centralize Nextflow cache and home into one directory
export NXF_HOME="${PROJ_SCRATCHDIR_CHARITE_USER}/.nextflow"
export NXF_CACHE_DIR="${NXF_HOME}"
# According to nf-core:
# "In some cases, the Nextflow Java virtual machines can start to request a large amount of memory.
# We recommend adding the following line to your environment to limit this."
export NXF_OPTS="-Xms1g -Xmx4g"
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

### Nextflow Strict Syntax

> [!IMPORTANT]
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

## Running Nextflow Workflows on the Charité HPC

### Conda Environment

The Conda environment `nf-core-2026-1` is provided by the SC HPC Team, which you can readily use. It contains both the nf-core CLI (`v4.2.0`) as well as Nextflow (`v26.10.4`).

```bash
conda activate nf-core-2026-1
```

### Option 1: Interactive Session

> [!CAUTION]
> Do not attempt to start resource-heavy jobs on frontend nodes. The memory on the frontends is restricted to **1 GB** per user. Your session will get killed (see [Frontend Memory Restrictions](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/faq/frontend-restrictions)). It is therefore recommended to work in an interactive session.

#### srun

```bash
# Interactive sessions launched with srun are capped at 8 hours runtime
srun --job-name=run-nf-core --partition=compute --ntasks=1 --cpus-per-task=4 --mem=16G --time=08:00:00 --pty bash
```

#### Open OnDemand Portal

It is also possible to jump right into an interactive session using the [Open OnDemand Portal](https://s-sc-ood.charite.de/pun/sys/dashboard/batch_connect/sessions) ([Open OnDemand Wiki](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#open-ondemand)). You must be connected the Charité network to use this, either via Ethernet cable or [OpenVPN Connect](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#connection-to-the-cluster-from-outside-the-charite-network-over-vpn).

<details closed>
<summary>Click here to reveal the variables code snippet for easy copy-pasting</summary>

```bash
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

# Define your workspace directory // Adjust as needed
MY_NXF_WORKSPACE="${PROJ_SCRATCHDIR_CHARITE}/my-nxf-workspace"
cd ${MY_NFCORE_WORKSPACE}
```

</details>

The above variables code snippet may be:

1. Copy-pasted into the terminal. With this method, you must set the variables again every time you start a new session.
2. Copy-pasted into your `~/.bashrc` file. After doing so, you must run `source ~/.bashrc` or start a new session, in order for the variables to be exported. Using this method, you no longer need to export them every time.

> [!WARNING]
> Be mindful of what you add into your `~/.bashrc` file, as it may unexpectedly alter the desired behavior of other programs.




First, `ssh` into a frontend node via your terminal as you normally would (see [Access](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#access)). You then have 2 options:




### Option 2: Batch Script

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

# Define your workspace directory // Adjust as needed
MY_NXF_WORKSPACE="${PROJ_SCRATCHDIR_CHARITE}/my-nxf-workspace"
cd ${MY_NFCORE_WORKSPACE}

# Do the work
nextflow run nf-core/demo \
    -profile charite \
    --outdir demo-test

    # -r <version> \ # Optional: pin a specific revision of a pipeline
    # -resume \ # Optional: resume an interrupted workflow
    # -c custom-params.config \ # Optional: add an extra config file which contains additional user-defined parameters
```

## Job Assignment Logic, Resource Allocation, GPU Acceleration


```mermaid
flowchart TB
    A(["Process / Job"])
    Q["'normal' QOS"]
    B{"'process_gpu'?"}

    A --> Q --> B

    subgraph ROUTING[" "]
        direction LR

        subgraph GPU["GPU routing"]
            direction TB
            C{"# GPUs?"}
            D["gpu"]
            E["pgpu"]

            C -->|"1 GPU"| D
            C -->|"2+ GPUs"| E
        end

        subgraph CPU["CPU routing"]
            direction TB
            F{"Profile(s)"}

            G["compute"]
            H["dragen_cc05"]
            I["dragen_cc15"]
            J["dragen_gp"]
            K["highmem"]
            L["genomics"]

            F -->|"-profile charite"| G
            F -->|"-profile charite,dragen_cc05"| H
            F -->|"-profile charite,dragen_cc15"| I
            F -->|"-profile charite,dragen_gp"| J
            F -->|"-profile charite,highmem"| K
            F -->|"-profile charite,genomics"| L
        end
    end

    B -->|"Yes — any profile"| C
    B -->|"No"| F

    D --> N(["SLURM"])
    E --> N
    G --> N
    H --> N
    I --> N
    J --> N
    K --> N
    L --> N

    classDef start fill:#dbeafe,stroke:#2563eb,color:#1e40af
    classDef gpu fill:#dcfce7,stroke:#16a34a,color:#166534
    classDef cpu fill:#dbeafe,stroke:#2563eb,color:#1e40af
    classDef decision fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef qos fill:#f3e8ff,stroke:#9333ea,color:#6b21a8

    class A,N start
    class Q qos
    class D,E gpu
    class G,H,I,J,K,L cpu
    class B,C,F decision

    style ROUTING fill:none,stroke:none
    style GPU fill:none,stroke:none
    style CPU fill:none,stroke:none
```

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
nextflow run nf-core/demo -profile test,charite --outdir test-demo
nextflow run nf-core/rnaseq -profile test,charite --outdir test-rnaseq
```

> [!IMPORTANT]
>
> "Order matters. Profiles load in sequence. Later profiles overwrite earlier ones." See [Configuration options](https://nf-co.re/docs/running/configuration/configuration-options) for more info.

## Known Limitations and Issues

### Jobs Pending in `gpu` Partition

Jobs requesting exactly 1 GPU may remain pending for submission to the `gpu` queue for a long time. The SC HPC Team is aware of this issue and is working on a solution. One temporary workaround is to force submission to the `pgpu` queue by requesting at least 2 GPUs.

```nextflow
process {
    withLabel:process_gpu {
        accelerator = 2
    }
}
```

Save the snippet above to `custom.config`, and then run your pipeline like so:

```
nextflow run nf-core/<pipeline-name> -profile charite --outdir results/<pipeline-name> -c custom.config
```