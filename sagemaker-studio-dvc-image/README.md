## Conda Environments as Kernels

This tutorial explains how to create a custom image for Amazon SageMaker Studio that has DVC already installed.
The advantage of creating an image and make it available to all SageMaker Studio users is that it creates a consistent environment for the SageMake Studio users, which they could also run locally.

This tutorial is heavily inspired by [this example](https://github.com/aws-samples/sagemaker-studio-custom-image-samples/tree/main/examples/conda-env-kernel-image).
Further information about custom images for SageMaker Studio can be found [here](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-byoi.html)

### Prerequisite

* An AWS account
* an IAM user with enough permissions (`AmazonEC2ContainerRegistryFullAccess`, and `SageMakerFullAccess`)
* a SageMaker Domain already created

### Overview

This custom image sample demonstrates how to create a custom Conda environment in a Docker image and use it as a custom kernel in SageMaker Studio.

The Conda environment must have the appropriate kernel package installed, for e.g., `ipykernel` for a Python kernel. This example creates a Conda environment called `myenv` with a few Python packages (see [environment.yml](environment.yml)) and the `ipykernel`. SageMaker Studio will automatically recognize this Conda environment as a kernel named `conda-env-myenv-py` (See  [app-image-config-input.json](app-image-config-input.json))

### Building the image

## Resize Cloud9

```bash
cd ~/sagemaker-dvc-catboost-demo/sagemaker-studio-dvc-image
./resize-cloud9.sh 20
```
Build the Docker image and push to Amazon ECR. 
```bash
sudo yum install jq -y
```

```bash
# Modify these as required. The Docker registry endpoint can be tuned based on your current region from https://docs.aws.amazon.com/general/latest/gr/ecr.html#ecr-docker-endpoints
export REGION=$(aws configure list | grep region | awk '{print $2}') # if AWS_DEFAULT_REGION is set takes precedence over the aws config
export ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')
```

```bash
# Build and push the image
export IMAGE_NAME=conda-env-dvc-kernel
aws --region ${REGION} ecr get-login-password | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/smstudio-custom

docker build . -t ${IMAGE_NAME} -t ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/smstudio-custom:${IMAGE_NAME}
docker push ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/smstudio-custom:${IMAGE_NAME}
```

### Using with SageMaker Studio

Create a SageMaker Image (SMI) with the image in ECR. 

```bash
# Role in your account to be used for SMI. Modify as required.
export ROLE_ARN=arn:aws:iam::${ACCOUNT_ID}:role/RoleName
```

```bash
aws --region ${REGION} sagemaker create-image \
    --image-name ${IMAGE_NAME} \
    --role-arn ${ROLE_ARN}

aws --region ${REGION} sagemaker create-image-version \
    --image-name ${IMAGE_NAME} \
    --base-image "${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/smstudio-custom:${IMAGE_NAME}"

# Verify the image-version is created successfully. Do NOT proceed if image-version is in CREATE_FAILED state or in any other state apart from CREATED.
aws --region ${REGION} sagemaker describe-image-version --image-name ${IMAGE_NAME}
```

Create a AppImageConfig for this image

```bash
aws --region ${REGION} sagemaker create-app-image-config --cli-input-json file://app-image-config-input.json

```

Create a Domain, providing the SageMaker Image and AppImageConfig in the Domain input. Replace the placeholders for VPC ID, Subnet IDs, and Execution Role in `create-domain-input.json`

```bash
aws --region ${REGION} sagemaker create-domain --cli-input-json file://create-domain-input.json
```

If you have an existing Domain, you can also use the `update-domain`

```bash
aws --region ${REGION} sagemaker update-domain --cli-input-json file://update-domain-input.json
```

Create a User Profile, and start a Notebook using the SageMaker Studio launcher.