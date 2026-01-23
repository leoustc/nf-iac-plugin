# NF-IAC plugin

> Infrastructure-as-Code executor for Nextflow. Provision with Terraform, run anywhere, tear it all down when done.

```bash

███╗   ██╗███████╗     ██╗ █████╗  ██████╗
████╗  ██║██╔════╝     ██║██╔══██╗██╔════╝
██╔██╗ ██║█████╗       ██║███████║██║     
██║╚██╗██║██╔══╝       ██║██╔══██║██║     
██║ ╚████║██║          ██║██║  ██║╚██████╗
╚═╝  ╚═══╝╚═╝          ╚═╝╚═╝  ╚═╝ ╚═════╝

NF-IAC 0.3.0
Author: leoustc
Repo: https://github.com/leoustc/nf-iac-plugin.git
------------------------------------------------------
IAC config:
  OCI profile       : YOUR PROFILE
  Compartment ID    : ocid1.compartment.oc1..aaaaaaaa_your_compartment_id
  Subnet ID         : ocid1.subnet.oc1.ap-singapore-1.aaaaaaaa_the_subnet_id
  Image ID          : ocid1.image.oc1.ap-singapore-1.aaaaaaaa_image_id
  CPU factor        : 2
  Terraform version : Terraform v1.14.3

```

> current version: nf-iac@0.3.0

nextflow search nf-iac

https://registry.nextflow.io/plugins/nf-iac@0.2.0

NF IAC is a Nextflow executor that provisions and destroys compute through Terraform. Deploy your workloads onto any Terraform-compatible infrastructure (bundled template targets OCI) while Nextflow keeps its usual work directories and polling loop.

## Features

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

## Quick start
Run any pipeline with the IAC executor:

```bash
nextflow run hello \
  -plugins nf-iac@0.1.0 \
  -process.executor iac \
  -work-dir s3://my-bucket/nf-work/hello
```

## Configuration
Add the executor and IAC block to your `nextflow.config`:

```groovy

plugins {
  id 'nf-iac@0.1.0'
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

  oci {
    profile     = 'DEFAULT'
    compartment = 'ocid1.compartment...'
    subnet      = 'ocid1.subnet...'
    image       = 'ocid1.image...'
    defaultShape = '1 VM.Standard.E4.Flex' // "<count> <shape>" format
    defaultDisk  = '1024 GB'
  }

  // Optional per-bucket credentials used inside the worker for S3 sync/cp
  storage = [
    [bucket: 'my-bucket', region: 'us-east-1', accessKey: 'xxx', secretKey: 'yyy', endpoint: 'https://s3.amazonaws.com']
  ]
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
Task                        CPU    LCPU   RAM(GB)  DISK(GB)    %CPU    %RSS   %VMEM   %DISK
SEQTK_TRIM(SAMPLE2_PE)        1       2        12      1024   4.46%    0.1%    0.2%    0.7%
SEQTK_TRIM(SAMPLE1_PE)        1       2        12      1024   4.67%    0.1%    0.2%    0.7%
FASTQC(SAMPLE1_PE)            3       6        36      1024   4.06%    1.4%  112.0%    0.8%
MULTIQC                       1       2         6      1024   3.30%   11.6%  201.4%    0.9%
SEQTK_TRIM(SAMPLE3_SE)        1       2        12      1024   4.46%    0.1%    0.2%    0.7%
FASTQC(SAMPLE3_SE)            3       6        36      1024   4.15%    1.2%  111.8%    0.8%
FASTQC(SAMPLE2_PE)            3       6        36      1024   3.82%    1.2%  112.0%    0.8%
------------------------------------------------------
- Goodbye!
```

Review the template in `iac.config` and replace the placeholder SSH key, bucket endpoints, and OCI identifiers with your own values before running.
