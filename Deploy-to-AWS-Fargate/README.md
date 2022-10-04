# Deploy a Stand-alone Watson NLP Runtime to Amazon ECS on AWS Fargate
[Amazon Web Services (AWS)](https://aws.amazon.com/) is a comprehensive cloud computing platform that includes infrastructure as a service (IaaS) and platform as a service (PaaS) offerings. [AWS Fargate](https://aws.amazon.com/fargate/) is a serverless, pay-as-you-go compute engine that lets you focus on building applications without managing servers. AWS Fargate is compatible with both [Amazon Elastic Container Service (ECS)](https://aws.amazon.com/ecs/) and [Amazon Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/).

This tutorial will walk you through the steps to deploy a stand-alone Watson NLP Runtime to Amazon ECS on AWS Fargate.


## Prerequisites
- Ensure you have an AWS account
- [Install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) AWS CLI
- [Configure](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html) a default profile with a proper default [region name](https://docs.aws.amazon.com/general/latest/gr/rande-manage.html)
- Download and install the latest version of [Docker Desktop](https://docs.docker.com/desktop/)
  - [Download for Mac](https://docs.docker.com/desktop/install/mac-install/)
  - [Download for Windows](https://docs.docker.com/desktop/install/windows-install/)
  - Alternatively, install the [Docker Compose CLI for Linux](https://docs.docker.com/cloud/ecs-integration/#install-the-docker-compose-cli-on-linux)

**Tip**:
- If you don't have an AWS account, you may want to consider [AWS Free Tier](https://aws.amazon.com/free/free-tier/).
- Follow the [security best practices](https://docs.aws.amazon.com/accounts/latest/reference/best-practices-root-user.html) for the root user of your AWS account, and [create an admin user](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html) for daily use.
- Make sure you have [required permissions](https://docs.docker.com/cloud/ecs-integration/#requirements) on AWS account to run applications on ECS.


## Create the runtime container image
The IBM Entitled Registry contains various container images for Watson Runtime. Once you've obtained the entitlement key from the [container software library](https://myibm.ibm.com/products-services/containerlibrary), you can login to the registry with the key, and pull the runtime images to your local machine. The Watson Runtime on its own doesn't have any models included. However, you can easily build a runtime container image to include one or more pretrained models, which are also stored as container images in the IBM Entitled Registry.

### Step 1: Login to the IBM Entitled Registry

<span style="font-size:x-small">

```
docker login cp.icr.io --username cp --password <your_entitlement_key>
```
</span>

### Step 2: Download a couple of models to a local directory

<span style="font-size:x-small">

```
mkdir models
REGISTRY=cp.icr.io/cp/ai
MODELS="watson-nlp_syntax_izumo_lang_en_stock:0.0.4 watson-nlp_syntax_izumo_lang_fr_stock:0.0.4"
for i in $MODELS
do
  image=$REGISTRY/$i
  docker run -it --rm -e ACCEPT_LICENSE=true -v `pwd`/models:/app/models $image
done
```
</span>

### Step 3: Create a ```Dockerfile``` using a text editor of your choice

<span style="font-size:x-small">

```
ARG TAG=0.0.1
FROM cp.icr.io/cp/ai/watson-nlp-runtime:${TAG}
COPY models /app/models
```
</span>

### Step 4: Build the image

<span style="font-size:x-small">

```
docker build -t my-watson-nlp-runtime:latest .
```
</span>


## Upload your runtime image to Amazon ECR
Now the runtime image is created, let's put it on [Amazon ECR](https://aws.amazon.com/ecr/), so that it can be used for deployment. Each AWS account is provided with a [default private registry](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Registries.html) (*aws_account_id*.dkr.ecr.*region*.amazonaws.com).

### Step 5: Login to the default registry

<span style="font-size:x-small">

```
aws ecr get-login-password | docker login --username AWS --password-stdin ${DEFAULT_REGISTRY}
```
</span>

### Step 6: Create a repository in the default registry

<span style="font-size:x-small">

```
aws ecr create-repository --repository-name my-watson-nlp-runtime
```
</span>

### Step 7: Upload the image to Amazon ECR

<span style="font-size:x-small">

```
# Tag the image
docker tag my-watson-nlp-runtime:latest ${DEFAULT_REGISTRY}/my-watson-nlp-runtime:latest

# Push the image
docker push ${DEFAULT_REGISTRY}/my-watson-nlp-runtime:latest
```
</span>


## Deploy the Runtime to Amazon ECS on AWS Fargate
[Docker ECS integration](https://docs.docker.com/cloud/ecs-integration/) transforms the Compose application standard into a collection of AWS resources, defined as a [AWS CloudFormation template](https://aws.amazon.com/cloudformation/resources/templates/). You can use `docker compose convert` to generate a CloudFormation stack file from your Compose file and inspect resources it defines. Once you have identified the changes required to the CloudFormation template, you can include overlays in your Compose file that will be automatically applied on `docker compose up`. An overlay is a YAML object that uses the same CloudFormation template data structure as the one generated by ECS integration, but only contains attributes to be updated or added. It will be merged with the generated template before being applied on the AWS infrastructure.

### Step 8: Create a AWS ECS context in Docker
Run the ```docker context create ecs myecscontext``` command to create an Amazon ECS Docker context named `myecscontext`.

<span style="font-size:x-small">

```
$ docker context create ecs myecscontext
? Create a Docker context using: An existing AWS profile
? Select AWS Profile default
Successfully created ecs context "myecscontext"
```
</span>

### Step 9: Create a `compose.yaml` file
The Docker Compose CLI relies on Amazon CloudFormation to manage the application deployment. You can use `docker compose convert` to generate a CloudFormation stack file from your Compose file and inspect resources it defines. Once you have identified the changes required to the CloudFormation template, you can include overlays in your Compose file that will be automatically applied on `compose up`. An overlay is a YAML object that uses the same CloudFormation template data structure as the one generated by ECS integration, but only contains attributes to be updated or added. It will be merged with the generated template before being applied on the AWS infrastructure.

<span style="font-size:x-small">

```
version: "3.8"

services:
  runtime:
    image: "${DEFAULT_REGISTRY}/my-watson-nlp-runtime:latest"
    deploy:
      x-aws-autoscaling: 
        min: 1
        max: 2 #required
        cpu: 75
        #mem: - mutualy exlusive with cpu
      resources:
        limits:
          cpus: '2'
          memory: 4096M
    ports:
      - target: 8080
        x-aws-protocol: http

networks:
  default:

x-aws-cloudformation:
  Resources:
    Runtime8080TargetGroup:
      Properties:
        HealthCheckPath: /swagger
        Matcher:
          HttpCode: 200-499
```
</span>

Notice the YAML block under `x-aws-cloudformation:`, which is an overlay that customizes the properties of the `Runtime8080TargetGroup` resource, for which no specific `x-aws-*` custom extension is available. 

### Step 10: Validate the `compose.yaml` file
Not only can you generate a CloudFormation stack file from the Compose file with the `docker compose convert` command, it can also help check the Compose file for syntax errors.

<span style="font-size:x-small">

```
docker --context myecscontext compose --project-name sample-project convert
```
</span>

### Step 11: Deploy it to Amazone ECS on AWS Fargate
If no errors are found by `docker compose convert`, and the generated CloudFormation template looks good to you, run the `docker compose up` command to deploy it.

<span style="font-size:x-small">

```
docker --context myecscontext compose --project-name sample-project up
```
</span>

### Step 12: Check your deployment
It may take a few minutes to complete the deployment. If successful, you can now list the deployed service with the `docker compose ps` command, which shows the `hostname:port` you need for accessing the service endpoint of the Watson NLP Runtime. Append `/swagger` to the URL and open it in a browser to access the Swagger UI, if you want to interact with the REST API resources provided by the Watson NLP Runtime.

<span style="font-size:x-small">

```
docker --context myecscontext compose --project-name sample-project ps
```
</span>

You can also check the application logs with the `docker compose logs` command.

<span style="font-size:x-small">

```
docker --context myecscontext compose --project-name sample-project logs
```
</span>

If you get a response like the following, the Watson NLP Runtime is working properly.

<span style="font-size:x-small">

```
{
  "text": "This is a test.",
  "producerId": {
    "name": "Izumo Text Processing",
    "version": "0.0.1"
  },
  "tokens": [
    {
      "span": {
        "begin": 0,
        "end": 4,
        "text": "This"
      },
      "lemma": "",
      "partOfSpeech": "POS_UNSET",
      "dependency": null,
      "features": []
    },
    {
      "span": {
        "begin": 5,
        "end": 7,
        "text": "is"
      },
      "lemma": "",
      "partOfSpeech": "POS_UNSET",
      "dependency": null,
      "features": []
    },
    {
      "span": {
        "begin": 8,
        "end": 9,
        "text": "a"
      },
      "lemma": "",
      "partOfSpeech": "POS_UNSET",
      "dependency": null,
      "features": []
    },
    {
      "span": {
        "begin": 10,
        "end": 14,
        "text": "test"
      },
      "lemma": "",
      "partOfSpeech": "POS_UNSET",
      "dependency": null,
      "features": []
    },
    {
      "span": {
        "begin": 14,
        "end": 15,
        "text": "."
      },
      "lemma": "",
      "partOfSpeech": "POS_UNSET",
      "dependency": null,
      "features": []
    }
  ],
  "sentences": [
    {
      "span": {
        "begin": 0,
        "end": 15,
        "text": "This is a test."
      }
    }
  ],
  "paragraphs": [
    {
      "span": {
        "begin": 0,
        "end": 15,
        "text": "This is a test."
      }
    }
  ]
}
```
</span>


## Clean up
Don't forget to clean up afterwards with the `docker compose down` command, to avoid paying for the cloud resources you no longer need.

<span style="font-size:x-small">

```
docker --context myecscontext compose --project-name sample-project down
```
</span>
