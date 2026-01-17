# NF IAC plugin

> Infrastructure-as-Code executor for Nextflow. Provision with Terraform, run anywhere, tear it all down when done.

Current version

> nf-iac@0.1.0

nextflow search nf-iac

https://registry.nextflow.io/plugins/nf-iac@0.1.0

NF IAC is a Nextflow executor that provisions and destroys compute through Terraform. Deploy your workloads onto any Terraform-compatible infrastructure (bundled template targets OCI) while Nextflow keeps its usual work directories and polling loop.

## Features

### Infrastructure-as-Code Executor
	•	Provisions compute resources dynamically using Terraform
	•	No pre-existing cluster or scheduler required
	•	Infrastructure lifecycle is bound to the workflow run
	•	Automatic teardown after completion or failure
	•	Supports ephemeral CPU and GPU workloads

### Multi-Endpoint S3 Support (No Vendor Lock-In)
	•	Supports multiple S3-compatible endpoints within a single workflow
	•	Buckets are routed to endpoints independently
	•	Enables mixing AWS S3, OCI Object Storage, MinIO, or other S3-compatible services
	•	Eliminates object-storage vendor lock-in without changing pipelines

### Object-Storage-First Execution Model
	•	Designed for object storage from day one
	•	No shared POSIX filesystem or NFS required
	•	Compatible with fully ephemeral cloud environments


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
Install locally and run any pipeline with the IAC executor:

```bash
make install
nextflow run hello \
  -plugins nf-iac@0.1.0 \
  -process.executor iac \
  -work-dir s3://my-bucket/nf-work/hello
```

## Configuration
Add the executor and IAC block to your `nextflow.config`:

```groovy
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

Review the template in `iac.config` and replace the placeholder SSH key, bucket endpoints, and OCI identifiers with your own values before running.
