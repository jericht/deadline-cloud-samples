# Maya container for AWS Deadline Cloud

This example builds a Docker image that packages Maya with the
[deadline-cloud-for-maya](https://github.com/aws-deadline/deadline-cloud-for-maya)
adaptor and GPU support for rendering on AWS Deadline Cloud.

## Use cases

- Run Maya Arnold GPU renders on Deadline Cloud service-managed fleets.
- Bundle third-party renderer plugins (V-Ray, Redshift) into the image at build time.

## Prerequisites

- **Docker** installed locally ([Get Docker](https://docs.docker.com/get-docker/))
- **AWS CLI** configured with credentials ([Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
- **An ECR repository** to store the built image ([Creating an ECR repository](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-create.html))
- **An S3 bucket** for job attachments ([Job attachments storage](https://docs.aws.amazon.com/deadline-cloud/latest/userguide/storage-job-attachments.html))
- **A Deadline Cloud farm** ([Getting started with Deadline Cloud](https://docs.aws.amazon.com/deadline-cloud/latest/userguide/getting-started.html))
- **IAM roles** for the queue and fleet ([Deadline Cloud IAM roles](https://docs.aws.amazon.com/deadline-cloud/latest/userguide/security-iam.html))
- **Maya Linux installer** (.tgz) downloaded from [Autodesk](https://www.autodesk.com/products/maya/overview)

## What's included

| Component | Description |
|-----------|-------------|
| Base image | `aswf/ci-base:2026` — Rocky Linux 8, CUDA 12.9, VFX Platform 2026 (industry standard) |
| Maya | Installed from user-supplied .tgz (default version 2025) |
| Adaptor | `deadline-cloud-for-maya` — the OpenJD adaptor that Deadline Cloud invokes to drive renders |
| Plugins | Optional renderer `.run` installers placed in `plugins/` |
| CloudFormation | `cloudformation.yaml` — deploys the queue, fleet, and queue environment in one stack |

## Project structure

```
maya-nvidia-cuda/
├── Dockerfile
├── cloudformation.yaml       # One-click deploy (queue + fleet + queue env)
├── installer/                # Place Maya .tgz here before building
└── plugins/                  # Place renderer .run installers here (optional)
```

## Building the image

1. Download the Maya Linux installer from Autodesk and place it in `installer/`:
   ```
   installer/Autodesk_Maya_2025_Linux_64bit.tgz
   ```

2. Optionally place renderer `.run` installers in `plugins/`.

3. Build:
   ```bash
   docker build -t maya-aswf:2025 .

   # Custom Maya version
   docker build --build-arg MAYA_VERSION=2024 -t maya-aswf:2024 .

   # Custom VFX Platform year
   docker build --build-arg VFX_PLATFORM_YEAR=2025 -t maya-aswf:2025 .
   ```

### Push to ECR

```bash
ECR_REPO=<your-account-id>.dkr.ecr.<region>.amazonaws.com/<your-repo-name>
ECR_REGISTRY=$(echo $ECR_REPO | cut -d/ -f1)
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin $ECR_REGISTRY
docker tag maya-aswf:2025 $ECR_REPO:2025
docker push $ECR_REPO:2025
```

## Deploying to Deadline Cloud

Use the provided CloudFormation template to deploy everything in one command:

```bash
aws cloudformation deploy \
    --template-file cloudformation.yaml \
    --stack-name maya-container-stack \
    --parameter-overrides \
        FarmId=farm-... \
        ECRImageURI=$ECR_REPO:2025 \
        FleetRoleArn=arn:aws:iam::...:role/FleetRole \
        QueueRoleArn=arn:aws:iam::...:role/QueueRole \
        JobAttachmentsBucket=my-deadline-bucket
```

This creates:
- A **queue** with job attachment settings and the container queue environment attached
- A **fleet** with GPU instances, Docker host configuration, and NVIDIA Container Toolkit
- A **queue-fleet association** connecting the two

### Updating the container image

After pushing a new image tag to ECR, update the stack so the queue environment's
default `ContainerImage` parameter points to the new tag. This way users submitting
jobs don't have to manually change the image URI in the submitter dialog.

```bash
aws cloudformation deploy \
    --template-file cloudformation.yaml \
    --stack-name maya-container-stack \
    --parameter-overrides \
        FarmId=farm-... \
        ECRImageURI=$ECR_REPO:2025.1 \
        FleetRoleArn=arn:aws:iam::...:role/FleetRole \
        QueueRoleArn=arn:aws:iam::...:role/QueueRole \
        JobAttachmentsBucket=my-deadline-bucket
```

### Tearing down

```bash
aws cloudformation delete-stack --stack-name maya-container-stack
```

## How it works

1. **Build** — The Dockerfile extracts Maya from the RPM installer, installs the adaptor, and bakes in any renderer plugins.
2. **Host config** — When a fleet instance launches, the host configuration script installs Docker and the NVIDIA Container Toolkit.
3. **Queue environment** — On each session, the enter script pulls the image, starts the container, and installs `maya-openjd`/`MayaAdaptor` wrappers that forward adaptor calls into the container via `docker exec`.
4. **Render** — The Deadline Cloud worker invokes `MayaAdaptor` as usual; the wrapper transparently runs it inside the container with GPU access.

## Licensing

Maya requires a valid license. The container uses `--network host` so it can reach
your FlexLM license server. Set the `ADSKFLEX_LICENSE_FILE` environment variable
in your job template or queue environment to point at your license server
(e.g. `2080@license-server.studio.local`).

## GPU support

GPU rendering (Arnold GPU) is automatic when the fleet has GPU instances. The queue
environment conditionally adds `--gpus all --runtime=nvidia` based on whether the
host has an NVIDIA GPU. CPU-only instances fall back to Arnold CPU rendering.
