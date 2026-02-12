# NF-IAC plugin

> Infrastructure-as-Code executor for Nextflow. Provision with Terraform, run anywhere, tear it all down when done.

```bash

███╗   ██╗███████╗     ██╗ █████╗  ██████╗
████╗  ██║██╔════╝     ██║██╔══██╗██╔════╝
██╔██╗ ██║█████╗       ██║███████║██║     
██║╚██╗██║██╔══╝       ██║██╔══██║██║     
██║ ╚████║██║          ██║██║  ██║╚██████╗
╚═╝  ╚═══╝╚═╝          ╚═╝╚═╝  ╚═╝ ╚═════╝

NF-IAC 1.0.2
Author: leoustc
Repo: https://github.com/leoustc/nf-iac-plugin.git
------------------------------------------------------
IAC config:
  OCI profile       : YOUR PROFILE
  Compartment ID    : ocid1.compartment.oc1..aaaaaaaa_your_compartment_id
  Subnet ID         : ocid1.subnet.oc1.ap-singapore-1.aaaaaaaa_the_subnet_id
  Image ID          : ocid1.image.oc1.ap-singapore-1.aaaaaaaa_image_id
  CPU factor        : 1
  RAM factor        : 1
  Terraform version : Terraform v1.14.3

.....

executor >  iac (7)
[31/b241ec] NFCORE_DEMO:DEMO:FASTQC (SAMPLE1_PE)     | 3 of 3 ✔
[f1/cc0232] NFCORE_DEMO:DEMO:SEQTK_TRIM (SAMPLE1_PE) | 3 of 3 ✔
[ed/dfe15b] NFCORE_DEMO:DEMO:MULTIQC                 | 1 of 1 ✔
-[nf-core/demo] Pipeline completed successfully-
Completed at: 26-Jan-2026 13:41:16
Duration    : 6m 14s
CPU hours   : (a few seconds)
Succeeded   : 7

-[nf-iac] plugin completed: task resource vs trace summary:
Task                        CPU    LCPU   RAM(GB)  DISK(GB)  PEAK_RSS(GB)  PEAK_VMEM(GB)   %DISK   STARTTIME     RUNTIME
FASTQC(SAMPLE2_PE)            6      12        36      1024          0.57         40.32       1         0.0         0.1
SEQTK_TRIM(SAMPLE2_PE)        2       4        12      1024          0.01          0.02       1         0.0         0.1
FASTQC(SAMPLE1_PE)            6      12        36      1024          0.53         40.32       1         0.0         0.1
SEQTK_TRIM(SAMPLE1_PE)        2       4        12      1024          0.01          0.02       1         0.0         0.1
MULTIQC                       1       2         6      1024          0.69         12.08       1         2.0         0.2
SEQTK_TRIM(SAMPLE3_SE)        2       4        12      1024          0.01          0.02       1         0.0         0.1
FASTQC(SAMPLE3_SE)            6      12        36      1024          0.53         40.32       1         0.0         0.1
------------------------------------------------------
- Goodbye!

```

> current version: nf-iac@1.0.2

nextflow search nf-iac

https://registry.nextflow.io/plugins/nf-iac@1.0.2

NF IAC is a Nextflow executor that provisions and destroys compute through Terraform. Deploy your workloads onto any Terraform-compatible infrastructure (bundled template targets OCI) while Nextflow keeps its usual work directories and polling loop.

## Features

### Task-level image override
- Real heterogeneous computing in one Nextflow run: mix x86, ARM, and GPU hosts with task-level `image`
- Match host images to hardware (x86 image for x86 shapes, ARM image for ARM shapes, GPU image for GPU shapes)
- Shape selection priority: `accelerator.type` > `ext.shape` > `iac.oci.defaultShape`
- Go global by overriding `profile`, `compartment`, and `subnet` per task—your pipeline becomes truly distributed and can target the best resources available

### Per task GPU enabling
- Use GPU as the accelerator per task, mix GPU and CPU pipeline, see below *GPU enable*
- Support profile docker,gpu as global GPU support

### Infrastructure-as-Code Executor
- Provisions compute resources dynamically using Terraform
- No pre-existing cluster or scheduler required
- Infrastructure lifecycle is bound to the workflow run
- Automatic teardown after completion or failure
- Supports ephemeral CPU and GPU workloads

### Multi-Endpoint S3 Support (No Vendor Lock-In)
- Supports multiple S3-compatible endpoints within a single workflow
- Buckets are routed to endpoints independently
- Enables mixing AWS S3, OCI Object Storage, MinIO, or other S3-compatible services
- Eliminates object-storage vendor lock-in without changing pipelines

**Notice:** When using multiple S3 endpoints, create placeholder files in the main AWS bucket to satisfy Nextflow's input file sanity checks (this will be improved in a future release).

### Object-Storage-First Execution Model
- Designed for object storage from day one
- No shared POSIX filesystem or NFS required
- Compatible with fully ephemeral cloud environments


## Why use NF IAC?
- **Provider-agnostic**: point at any Terraform-compatible provider; swap modules without changing your pipeline.
- **Multi-storage aware**: set endpoints and credentials per bucket for S3-compatible object stores.
- **Lifecycle-safe**: retries on apply, best-effort destroy after tasks and again on shutdown.
- **Task-native**: runs `nxf_work.sh` user data so `.command.*` markers and logs behave as standard Nextflow.
- **Debuggable**: per-task artifacts under `.nextflow/iac/<run>/<aa>/<hash>/` (`tfvars.json`, `nxf_work.sh`, `.iac.out/.iac.err`).

## Requirements
- Nextflow 24.10.0 or newer
- Terraform 1.0+ available on the host that launches tasks
- OCI credentials/profile for the target compartment/subnet/image (current template); SSH public key to access nodes if needed
- S3-compatible storage credentials when using an object-store workDir or staged inputs
- `rclone` on worker nodes (auto-installed in `nxf_work.sh` if missing; uses apt or the official install script)

## Quick start
Run any pipeline with the IAC executor:

```bash
nextflow run hello \
  -plugins nf-iac@1.0.2 \
  -process.executor iac \
  -work-dir s3://my-bucket/nf-work/hello
```

## Configuration
Add the executor and IAC block to your `nextflow.config`:

```groovy

plugins {
  id 'nf-iac@1.0.2'
  id 'nf-amazon@3.4.1'
}

process {
  executor = 'iac'
}

workDir = 's3://my-bucket/nf-work/demo'

iac {
  namePrefix = 'nf-iac'
  sshAuthorizedKey = 'ssh-rsa AAAA...'
  wrapperWaitTimeout = '2 min'     // optional; wait for wrapper in workDir
  wrapperWaitInterval = '5 sec'    // optional

  cpu_factor = 2   // scales task cpus to ocpus as round(cpus * cpu_factor), min 1
  ram_factor = 2   // scales task memory_gb to memory_gbs as round(memory_gb * ram_factor), min 1

  oci {
    profile     = 'DEFAULT'
    compartment = 'ocid1.compartment...'
    subnet      = 'ocid1.subnet...'
    image       = 'ocid1.image...'
    defaultShape = '1 VM.Standard.E4.Flex' // "<count> <shape>" format
    defaultDisk  = '1024 GB'
  }

  // Optional per-bucket credentials used inside the worker for rclone object-store sync/copy
  storage = [
    [bucket: 'my-bucket', region: 'us-east-1', accessKey: 'xxx', secretKey: 'yyy', endpoint: 'https://s3.amazonaws.com']
  ]
}

process {
  // Heterogeneous compute (shape/image matching)
  withName: 'TASK_ARM' {
    ext.shape = '1 VM.Standard.A1.Flex'
    ext.image = 'ocid1.image.oc1..arm_image'
  }
  withName: 'TASK_X86' {
    ext.shape = '1 VM.Standard.E4.Flex'
    ext.image = 'ocid1.image.oc1..x86_image'
  }
  withName: 'TASK_GPU' {
    ext.shape = '1 VM.GPU.A10.1'
    ext.image = 'ocid1.image.oc1..gpu_image'
  }

  // Global distribution (region/account/network overrides)
  withName: 'TASK_SINGAPORE' {
    ext.profile = 'SG_PROFILE'
    ext.compartment = 'ocid1.compartment.oc1..sg_compartment'
    ext.subnet = 'ocid1.subnet.oc1..sg_subnet'
    ext.shape = '1 VM.Standard.E4.Flex'
    ext.image = 'ocid1.image.oc1..sg_image'
  }
  withName: 'TASK_SANJOSE' {
    ext.profile = 'SJ_PROFILE'
    ext.compartment = 'ocid1.compartment.oc1..sj_compartment'
    ext.subnet = 'ocid1.subnet.oc1..sj_subnet'
    ext.shape = '1 VM.Standard.E4.Flex'
    ext.image = 'ocid1.image.oc1..sj_image'
  }
}

aws {
  region = 'us-east-1'
  accessKey = 'xxx'
  secretKey = 'yyy'
  client {
    endpoint = 'https://s3.amazonaws.com'
    s3PathStyleAccess = true
  }
}
```

## GPU enable per task

```groovy
process {
  executor = 'iac'
  withName: 'BISMARK_ALIGN' {
    accelerator = [type: 'VM.GPU.A10.1', request: 1]   // use the GPU shape from OCI as example VM.GPU.A10.1, VM.GPU.A10.2, BM.GPU.A10.4, BM.GPU4.8
    containerOptions = '--gpus all'                    // add the GPU flag here, DO NOT add GPU flag in the profile unless all task in your pipeline is GPU compatible
    afterScript = 'hostname >> .command.out; nvidia-smi >> .command.out' // check the .command.out to see GPU is enabled
  }
}
```
## Resource metrics at the end

```bash
-[nf-iac] plugin completed: task resource vs trace summary:
Task                        CPU    LCPU   RAM(GB)  DISK(GB)  PEAK_RSS(GB)  PEAK_VMEM(GB)   %DISK   STARTTIME     RUNTIME
FASTQC(SAMPLE2_PE)            6      12        36      1024          0.57         40.32       1         0.0         0.1
SEQTK_TRIM(SAMPLE2_PE)        2       4        12      1024          0.01          0.02       1         0.0         0.1
FASTQC(SAMPLE1_PE)            6      12        36      1024          0.53         40.32       1         0.0         0.1
SEQTK_TRIM(SAMPLE1_PE)        2       4        12      1024          0.01          0.02       1         0.0         0.1
MULTIQC                       1       2         6      1024          0.69         12.08       1         2.0         0.2
SEQTK_TRIM(SAMPLE3_SE)        2       4        12      1024          0.01          0.02       1         0.0         0.1
FASTQC(SAMPLE3_SE)            6      12        36      1024          0.53         40.32       1         0.0         0.1
------------------------------------------------------
- Goodbye!
```

Each run also writes a persistent summary file at:
`./.nextflow/iac/<run-id>/pipeline-resource-usage-<run-id>.txt`

## Visualization notebook
Use the bundled notebook to visualize pipeline resource usage:
`visualization.ipynb`

## Rclone details
- Config written to `/etc/rclone/rclone.conf` (no `$HOME` fallback).
- Object-store paths use `bucket:bucket/path` (e.g. `nf-data:nf-data/work-demo/...`).
- Input resolution: directory inputs use `rclone sync`; file inputs use `rclone copyto`.
- `send_exitcode` uploads `.command.out`, `.command.err`, `.command.trace`, `.command.log`, `.nxf.log`, `.exitcode` and exits `0` (best effort).

Review the template in `iac.conf` and replace the placeholder SSH key, bucket endpoints, and OCI identifiers with your own values before running.
