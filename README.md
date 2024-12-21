# Remove Dangling Docker Images

**Custom GitHub Action to remove dangling Docker images locally and from AWS Elastic Container Registry (ECR).**

This action provides a simple yet powerful way to clean up unused and untagged Docker images, both locally and on AWS ECR. It is particularly useful for maintaining optimal resource usage and avoiding unnecessary storage costs.

---

## Features

- **Remove Local Dangling Images:** Deletes unused and untagged Docker images from the local environment.
- **Clean AWS ECR:** Identifies and deletes untagged images (dangling images) from AWS ECR.
- **Flexible Configuration:**
  - Optional input for specifying the SSH host.
  - Optional inputs for AWS ECR repository name and region.
- **SSH Configuration Check:** Ensures that the `./.tmp/.ssh/config` file exists before running any commands.

---

## Inputs

| Input                | Description                                 | Required | Default          |
| -------------------- | ------------------------------------------- | -------- | ---------------- |
| `host`               | The SSH host for executing remote commands. | No       | `github-actions` |
| `aws-ecr-repository` | The name of the AWS ECR repository.         | No       |                  |
| `aws-ecr-region`     | The AWS region of the ECR repository.       | No       |                  |

---

## Usage

### Prerequisites

1. **SSH Configuration**
   Ensure you have a valid SSH configuration file at `./.tmp/.ssh/config`.

2. **AWS Credentials**
   Ensure your AWS CLI is configured with appropriate permissions to describe and delete ECR images.

### Example Workflow

```yml
name: "Deploy"

on:
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure SSH credentials
        uses: rajpal-se/configure-ssh-credentials@v1
        with:
          key: ${{ secrets.SSH_KEY }}
          hostname: ${{ secrets.HOSTNAME }}

      - name: Remove Dangling Docker Images
        uses: ./
        with:
          host: ${{ secrets.HOSTNAME }}
          aws-ecr-repository: ${{ secrets.AWS_ECR_REPOSITORY }}
          aws-ecr-region: ${{ secrets.AWS_ECR_REGION }}
```

---

## How It Works

### Step 1: Check for SSH Configuration File

The action ensures that `./.tmp/.ssh/config` exists. If the file is missing, the action will terminate with an error.

```bash
if [ ! -f "./.tmp/.ssh/config" ]; then
  echo "Error: SSH config file './.tmp/.ssh/config' does not exist."
  exit 1
fi
```

### Step 2: Remove Local Dangling Docker Images

Executes the following command over SSH to prune unused Docker images on the specified host:

```bash
ssh -F "./.tmp/.ssh/config" ${{ inputs.host }} << 'EOF'
  docker image prune -f
EOF
```

### Step 3: Clean AWS ECR

If `aws-ecr-repository` and `aws-ecr-region` are provided, the action deletes untagged images from the specified ECR repository:

```bash
ssh -F "./.tmp/.ssh/config" ${{ inputs.host }} << 'EOF'
  for digest in $(aws ecr describe-images --repository-name ${{ inputs.aws-ecr-repository }} --region ${{ inputs.aws-ecr-region }} --query 'imageDetails[?imageTags==`null`].imageDigest' --output text); do
    aws ecr batch-delete-image --repository-name ${{ inputs.aws-ecr-repository }} --region ${{ inputs.aws-ecr-region }} --image-ids imageDigest=$digest
  done
EOF
```

---

## Use Cases

### 1. Local Environment Cleanup

Remove unused images from a self-hosted runner or remote environment to free up space:

```yml
- name: Remove Local Dangling Docker Images
  uses: ./
  with:
    host: "self-hosted-runner"
```

### 2. AWS ECR Optimization

Optimize ECR storage by deleting untagged images:

```yml
- name: Remove Dangling Images from AWS ECR
  uses: ./
  with:
    aws-ecr-repository: "my-ecr-repo"
    aws-ecr-region: "us-east-1"
```

### 3. Combined Workflow

Run both local and AWS ECR cleanups in a single step:

```yml
- name: Complete Dangling Docker Image Cleanup
  uses: ./
  with:
    host: "self-hosted-runner"
    aws-ecr-repository: "my-ecr-repo"
    aws-ecr-region: "us-west-2"
```

---

## Branding

This action is branded with a red command icon to stand out in the GitHub Marketplace.

---

## License

This project is licensed under the MIT License.

<!-- - see the [LICENSE](LICENSE) file for details. -->

---

## Author

Developed by **Rajpal Singh**. Contributions are welcome!
