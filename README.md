
# cf-ecs-deploy
Deploy Codefresh to Amazon ECS Service

### Prerequisites
- Configure an ECS Cluster with at least one running instance.
- Configure an ECS Service and task definition with a deployed image.
  See http://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html

- Verify you have AWS Credentials (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY), with following privileges:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1479146904000",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:DescribeTasks",
        "ecs:ListClusters",
        "ecs:ListServices",
        "ecs:ListTasks",
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

### Deployment with Codefresh
The ```codefresh.yml``` file runs the ```codefresh/cf-deploy-ecs``` image with the ```cfecs-update``` command.

1. Add encrypted environment variables for AWS credentials.
     * AWS_ACCESS_KEY_ID
     * AWS_SECRET_ACCESS_KEY
2. Add the "deploy to ecs" step to the ```codefresh.yml``` file.
3. Specify the following parameters.
   - `aws region`
   - `ecs cluster`
   - `ecs-service-names`. 
See `cfecs-update -h` for parameter references.

```yaml
# codefresh.yml example with deploy to ecs step
version: '1.0'

steps:
  build-step:
    type: build
    image-name: repo/image:tag

  push to registry:
    type: push
    candidate: ${{build-step}}
    tag: ${{CF_BRANCH}}

  deploy to ecs:
    image: codefresh/cf-deploy-ecs
    commands:
      - cfecs-update <aws-region> <ecs-cluster-name> <ecs-service-name>
    environment:
      - AWS_ACCESS_KEY_ID=${{AWS_ACCESS_KEY_ID}}
      - AWS_SECRET_ACCESS_KEY=${{AWS_SECRET_ACCESS_KEY}}

    when:
      - name: "Execute for 'master' branch"
        condition: "'${{CF_BRANCH}}' == 'master'"
```


### Deployment Flow
1. Get the ECS service by specified `aws-region`, `ecs-cluster`, and `service-names`.
2. Create a new revision from the current task definition of the service. If `--image-name` and `--image-tag` are provided, replace the image tag.
3. Run the `update-service` command with the new task definition revision.
4. Wait for the deployment to complete. 
   By default, service deployment is no run with the `--no-wait` command.
    * Deployment is successfully completed if runningCount == desiredCount for PRIMARY deployment - see `aws ecs describe-service`
    * The `cfecs-update` command exits with a timeout error if --timeout (default = 900s) runningCount != desiredCount script exits with timeout
    * The `cfecs-update` exits with an error if --max-failed (default = 2) or more ECS tasks were stopped with error for the task definition that you are deploying.
      ECS continuously retries failed tasks.

### Usage with Docker

```bash
docker run --rm -it -e AWS_ACCESS_KEY_ID=**** -e AWS_SECRET_ACCESS_KEY=**** codefresh/cf-ecs-deploy cfecs-update [options] <aws-region> <ecs-cluster-name> <ecs-service-name>
```

### cfecs-update -h
```
usage: cfecs-update [-h] [-i IMAGE_NAME] [-t IMAGE_TAG] [--wait | --no-wait]
                    [--timeout TIMEOUT] [--max-failed MAX_FAILED] [--debug]
                    region_name cluster_name service_name

Codefresh ECS Deploy

positional arguments:
  region_name           AWS Region, ex. us-east-1
  cluster_name          ECS Cluster Name
  service_name          ECS Service Name

optional arguments:
  -h, --help            show this help message and exit
  --wait                Wait for deployment to complete (default)
  --no-wait             No Wait for deployment to complete
  --timeout TIMEOUT     deployment wait timeout (default 900s)
  --max-failed MAX_FAILED
                        max failed tasks to consider deployment as failed
                        (default 2)
  --debug               show debug messages

  -i IMAGE_NAME, --image-name IMAGE_NAME
                        Image Name in ECS Task Definition to set new tag
  -t IMAGE_TAG, --image-tag IMAGE_TAG
                        Tag for the image
```
