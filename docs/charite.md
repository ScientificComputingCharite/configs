# nf-core/configs: Charité Configuration

This page documents the `charite` Nextflow profile for running Nextflow and nf-core pipelines on the Charité HPC. See the full configuration at [`conf/charite.config`](../conf/charite.config). For general HPC information, see [Charité Science IT](https://www.charite.de/en/research/research_support_services/research_infrastructure/science_it/).

> Contact: Scientific Computing, GB IT, Charité — sc-hpc-helpdesk@charite.de

The config uses **Slurm** and **Apptainer**. Apptainer requires no module loading.

## Requesting Access

The Charité internal user documentation/wiki requires a Charité GitLab account. To request access:
1. Email sc-hpc-helpdesk@charite.de to request access to the Charité HPC — this initiates the onboarding process.
2. If you do not have access to git.bihealth.org, contact health-data@charite.de to request GitLab access for the [Charité HPC User Documentation and Onboarding Guide](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/home).

## Contributors and Acknowledgments

- Wassim Salam ([@wassimsalam01](https://github.com/wassimsalam01)) - BIH/MDC Genomics Platform Data Management Team
- Magnus Hagdorn ([@mhagdorn](https://github.com/mhagdorn)) - Scientific Computing HPC Team
- Andreas Reppas ([@areppas](https://github.com/areppas)) - Scientific Computing HPC Team

WS thanks the SC HPC Team for their support and guidance in creating this config. If you wish to contribute:
1. Contact sc-hpc-helpdesk@charite.de to discuss your suggestions
2. Fork [nf-core/configs](https://github.com/nf-core/configs) and commit your changes there
3. Get in touch with sc-hpc-helpdesk@charite.de for review once you've tested functionality with your changes
4. Add yourself to the list of contributors
5. Create a PR to officially merge your changes

## Running nf-core pipelines on the Charité HPC

The Conda environment `nextflow` is provided by the SC HPC Team (>=26.04), which you can readily use. Do NOT launch Nextflow on frontend nodes. The memory on the frontends is restricted to **1 GB** per user. Your session will get killed (see [Frontend Memory Restrictions](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/faq/frontend-restrictions)). Additionally, Apptainer is not installed on frontend nodes, which will result in using the config to fail. You have 2 options to run Nextflow:

### Interactive Session

Log into a frontend node via your terminal as you normally would (see [Access](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#access)), then run the following:

```bash
srun --job-name=run-nxf-pipeline --partition=compute --ntasks=1 --cpus-per-task=4 --mem=16G --time=12:00:00 --pty bash
```

Followed by:

```
conda activate nextflow
cd /path/to/workspace
nextflow run nf-core/<pipeline> \
    -profile charite \
    --input ./samplesheet.cv \
    --outdir ./results
```

It is also possible to jump right into an interactive session using the [Open OnDemand Portal](https://s-sc-ood.charite.de/pun/sys/dashboard/batch_connect/sessions). You must be connected the Charité network to use this, either via Ethernet cable or OpenVPN Connect ([Open OnDemand](https://git.bihealth.org/charite-sc-public/sc-wiki/-/wikis/Resources/User-Documentation/User-Guide:-HPC-@Charite#open-ondemand)).

### Batch Script

Save the following script under `run-nxf-pipeline.sh` and submit it with `sbatch run-nxf-pipeline.sh`:

```bash
#!/usr/bin/env bash
#SBATCH --job-name=run-nxf-pipeline
#SBATCH --partition=compute
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=12:00:00

set -euo pipefail
source "${HOME}/.bashrc" # Activates conda: required to activate nextflow
conda activate nextflow
cd /path/to/workspace

nextflow run nf-core/<pipeline> \
    -r <version> \ # optional: pin a specific version of the pipeline
    -resume \ # optional: resume an interrupted workflow
    -profile charite \
    -params-file my-pipeline-params.yaml
```

`my-pipeline-params.yaml`

```yaml
input: ./samplesheet.csv
outdir: ./results
```

## Environment Variables Setup

In order for the profile to run as intended, the following variables must first be exported before running any workflow. The code snippets may be:

- copy-pasted into the terminal if working in an interactive session: once you end your session, you must set the variables again
- added to your batch script:  
- added to your "~/.bashrc" file: must be followed by `source ~/.bashrc`. Upon starting a new session, the variables are already defined

### Global Utility Variables

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


### Apptainer-specific Variables

> Another reason this profile does not work on frontend nodes is that Apptainer is only installed on compute nodes.

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

&rarr;  No: default to `${PROJ_SCRATCHDIR_CHARITE_USER}/apptainer-images` &rarr; your folder structure should eventually look like this:

```
/sc-scratch/sc-scratch-${PROJ_ABBR}/${USER}/
├── .apptainer
│   ├── cache
│   └── tmp
└── apptainer-images
```

&rarr; Yes: setting `${NXF_APPTAINER_CACHEDIR}` overrides the default path assignment.

**How is this useful?** You may set this to a custom path where you are already storing Apptainer images. Another possibility is to store built at a centralized path accessible to all collaborators of a project, if agreed upon by all. This then allows reuse of images between users and keeps unnecessary redownloading of images to the project scratch to a minimum. You can achieve this by executing:

```
export NXF_APPTAINER_CACHEDIR="${PROJ_SCRATCHDIR_CHARITE}/apptainer-images"
```

&rarr; your folder structure should look like this:

```
/sc-scratch/sc-scratch-${PROJ_ABBR}/${USER}/
└── .apptainer
    ├── cache
    └── tmp


/sc-scratch/sc-scratch-${PROJ_ABBR}/ # or another custom path
└── apptainer-images
```

### Nextflow-specific Variables



## Job Assignment Logic, Resource Allocation, GPU Acceleration


## Additional Profiles

The config offers additional profiles, which users can use to run their analyses. Each profile has a dedicated partition with corresponding resource limits.

```bash
nextflow run nf-core/<pipeline> -profile charite,<profile> --input samplesheet.csv --outdir results
```

`debug` is an additional utility profile rather than a one with a dedicated partition. It disables automatic cleanup.

```bash
nextflow run nf-core/<pipeline> -profile charite,<profile>,debug --input samplesheet.csv --outdir results
```
