# nf-iac — Infrastructure-as-Code Executor for Nextflow

nf-iac is a Nextflow executor plugin that treats infrastructure and storage as first-class parts of the workflow lifecycle.

Instead of assuming a pre-existing cluster or a fixed cloud backend, nf-iac dynamically provisions compute resources, executes workflows, and tears everything down — while remaining cloud-agnostic and object-storage-first.

---

## Core Concepts

- **Executor = Infrastructure**
- **Workflow run = Ephemeral compute environment**
- **Execution state = Single execution bucket**
- **Inputs = Multi-cloud, multi-vendor**

---

## Features

### Infrastructure-as-Code Execution

- Compute resources are provisioned dynamically using Terraform
- No pre-existing cluster required
- Infrastructure lifecycle is bound to the workflow run
- Automatic cleanup after completion or failure
- Supports both CPU and GPU workloads

This turns Nextflow into a control plane, not just a scheduler.

---

### Cloud & Vendor Agnostic

- Works with any Terraform-supported cloud
- No dependency on a specific cloud provider
- Designed for hybrid and multi-cloud environments
- Avoids compute vendor lock-in by design

---

### Multi-Endpoint S3 Support (No Storage Vendor Lock-In)

nf-iac supports multiple S3-compatible endpoints, routed by bucket.

- Different buckets may point to different S3 providers
- Inputs can be read from any S3-compatible service
- Execution and results remain within a single execution bucket

This enables workflows to consume data across clouds without forcing data migration.

---

### Clear Storage Boundary

To preserve correctness and reproducibility:

- `workDir` and `outputDir` must reside in the same bucket
- That bucket defines the execution boundary of a workflow run
- Inputs may come from any bucket or endpoint

This ensures deterministic execution, safe teardown, and reliable resume behavior.

---

### Object-Storage-First Design

- No shared POSIX filesystem required
- No NFS dependency
- Designed for ephemeral cloud environments
- Optimized for object storage semantics

---

## Usage

Run any Nextflow pipeline with nf-iac:

```bash
nextflow run <pipeline> \
  -plugins nf-iac@0.1.0 \
  -process.executor iac \
  -work-dir s3://nf-data/nf-work/run1
```

---

## Configuration (iac.config)

All executor configuration is defined in `iac.config`.  
Users should include this file rather than redefining executor settings inline.

```groovy
iac {
  pollInterval = '60 sec'   // how often the executor polls for task updates
  dumpInterval = '30 m'     // how often task logs and metadata are flushed
  namePrefix   = 'somelonglongworker' // prefix applied to generated resource names

  /*
   * Storage configuration
   *
   * - workDir and outputDir must reside in the same bucket
   * - inputs may come from any bucket listed below
   * - each bucket may use a different S3-compatible endpoint
   */
  storage = [
    [
      bucket: 'nf-data',                          // execution bucket (workDir + outputs)
      region: System.getenv('AWS_REGION'),
      accessKey: System.getenv('AWS_ACCESS_KEY_ID'),
      secretKey: System.getenv('AWS_SECRET_ACCESS_KEY'),
      endpoint: System.getenv('AWS_ENDPOINT'),
    ],
    [
      bucket: 'ngi-igenomes',                     // optional secondary / input bucket
      region: System.getenv('AWS_REGION'),
      accessKey: System.getenv('AWS_ACCESS_KEY_ID'),
      secretKey: System.getenv('AWS_SECRET_ACCESS_KEY'),
      endpoint: System.getenv('AWS_ENDPOINT'),
    ]
  ]

  /*
   * SSH access
   *
   * Public key injected into provisioned nodes
   */
  sshAuthorizedKey = 'ssh-ed25519 AAAA...REDACTED'

  /*
   * OCI infrastructure configuration
   */
  oci {
    compartment  = System.getenv('COMPARTMENT_OCID')
    subnet       = System.getenv('SUBNET_OCID')
    image        = System.getenv('IMAG_OCID')
    profile      = 'MIRXES'
    defaultShape = '1 VM.Standard.E4.Flex'
    defaultDisk  = '1024 GB'
  }
}
```

Include the configuration in your Nextflow setup:

```groovy
includeConfig 'iac.config'
```

---

## Storage Model Summary

| Path | Requirement |
|------|-------------|
| `workDir` | Single bucket |
| `outputDir` | Same bucket as `workDir` |
| `inputs` | Any S3 bucket, any endpoint |
| `providers` | Mixed, S3-compatible |

---

## Why nf-iac?

Traditional Nextflow execution assumes:

- static clusters
- single-cloud storage
- shared POSIX filesystems

nf-iac removes these assumptions and enables:

- ephemeral, per-run infrastructure
- cloud-neutral execution
- portable data access
- cost-efficient burst workloads

---

## Project Status

- Published on the Nextflow Plugin Registry
- Early-stage but functional
- Actively developed
- Feedback and contributions welcome

---

## Links

- **GitHub**: https://github.com/leoustc/nf-iac-plugin
- **Registry**: https://registry.nextflow.io/plugins/nf-iac@0.1.0

---

## License

Apache 2.0 (unless otherwise stated)
