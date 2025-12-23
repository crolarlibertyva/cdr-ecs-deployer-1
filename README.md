# ECS Deployer - GitHub Actions Workflow

This repository contains a GitHub Actions workflow for deploying Docker images to AWS ECS with configurable parameters.

## Features

- Deploy any Docker image to AWS ECS
- Configure CPU and memory resources
- Set environment variables and secrets
- Configure volume mounts
- Uses AWS IAM authentication (OIDC or IAM user credentials)
- Automatic service deployment and stability checks

## Setup

### 1. AWS Authentication

#### Option A: OIDC (Recommended)

1. Create an IAM OIDC identity provider in AWS for GitHub Actions
2. Create an IAM role with trust policy for GitHub Actions
3. Add the role ARN as a GitHub secret: `AWS_ROLE_ARN`

Required IAM permissions:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "application-autoscaling:RegisterScalableTarget",
        "application-autoscaling:PutScalingPolicy",
        "application-autoscaling:DescribeScalableTargets",
        "application-autoscaling:DescribeScalingPolicies",
        "cloudwatch:PutMetricAlarm",
        "cloudwatch:DescribeAlarms",
        "cloudwatch:DeleteAlarms",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

#### Option B: IAM User Credentials

1. Create an IAM user with the required permissions (see above)
2. Add GitHub secrets:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
3. Update the workflow to use credentials instead of OIDC (see commented lines in workflow)

### 2. GitHub Secrets

Add the following secrets to your GitHub repository:
- `AWS_ROLE_ARN` (if using OIDC)
- OR `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` (if using IAM user)

## Usage

### Trigger the Workflow

Go to Actions → Deploy to AWS ECS → Run workflow

### Required Parameters

- **docker_image**: The Docker image to deploy (e.g., `123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0`)
- **ecs_cluster**: Name of your ECS cluster
- **ecs_service**: Name of your ECS service
- **task_definition_family**: Task definition family name
- **container_name**: Container name in the task definition

### Optional Parameters

- **cpu**: CPU units (default: `256`)
  - Options: 256 (.25 vCPU), 512 (.5 vCPU), 1024 (1 vCPU), 2048 (2 vCPU), 4096 (4 vCPU)

- **memory**: Memory in MB (default: `512`)
  - Must be compatible with CPU setting (see AWS documentation)

- **aws_region**: AWS region (default: `us-east-1`)

- **desired_count**: Number of ECS tasks to run (optional)
  - If not specified, keeps the current desired count
  - Example: `2` (runs 2 instances of the task)

- **enable_autoscaling**: Enable auto-scaling for the ECS service (default: `false`)
  - Set to `true` to configure auto-scaling policies

- **min_capacity**: Minimum number of tasks for auto-scaling (default: `1`)
  - Only used when `enable_autoscaling` is `true`

- **max_capacity**: Maximum number of tasks for auto-scaling (default: `10`)
  - Only used when `enable_autoscaling` is `true`

- **target_cpu_utilization**: Target CPU utilization percentage (default: `70`)
  - Auto-scales to maintain this CPU usage (0-100)
  - Only used when `enable_autoscaling` is `true`

- **target_memory_utilization**: Target memory utilization percentage (default: `80`)
  - Auto-scales to maintain this memory usage (0-100)
  - Only used when `enable_autoscaling` is `true`

- **scale_in_cooldown**: Cooldown period in seconds after scale-in (default: `300`)
  - Prevents rapid scale-down after scaling in
  - Only used when `enable_autoscaling` is `true`

- **scale_out_cooldown**: Cooldown period in seconds after scale-out (default: `60`)
  - Prevents rapid scale-up after scaling out
  - Only used when `enable_autoscaling` is `true`

- **environment_variables**: Environment variables in JSON array format
  ```json
  [
    {"name": "NODE_ENV", "value": "production"},
    {"name": "API_URL", "value": "https://api.example.com"}
  ]
  ```

- **secrets**: AWS Secrets Manager or Parameter Store secrets in JSON array format
  ```json
  [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:db-password"
    },
    {
      "name": "API_KEY",
      "valueFrom": "arn:aws:ssm:us-east-1:123456789:parameter/api-key"
    }
  ]
  ```

- **mount_points**: Volume mount points in JSON array format
  ```json
  [
    {
      "sourceVolume": "my-volume",
      "containerPath": "/data",
      "readOnly": false
    }
  ]
  ```

- **volumes**: Volume definitions in JSON array format
  ```json
  [
    {
      "name": "my-volume",
      "host": {
        "sourcePath": "/mnt/data"
      }
    }
  ]
  ```
  
  Or for EFS:
  ```json
  [
    {
      "name": "efs-volume",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-1234567",
        "transitEncryption": "ENABLED"
      }
    }
  ]
  ```

## Example Deployment

```yaml
# Minimal deployment
docker_image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
ecs_cluster: my-cluster
ecs_service: my-service
task_definition_family: my-app
container_name: my-app-container

# Deployment with fixed task count
docker_image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
ecs_cluster: production-cluster
ecs_service: api-service
task_definition_family: api-task
container_name: api-container
cpu: "1024"
memory: "2048"
desired_count: 3

# Deployment with auto-scaling
docker_image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
ecs_cluster: production-cluster
ecs_service: api-service
task_definition_family: api-task
container_name: api-container
cpu: "1024"
memory: "2048"
enable_autoscaling: true
min_capacity: 2
max_capacity: 20
target_cpu_utilization: 70
target_memory_utilization: 80
scale_in_cooldown: 300
scale_out_cooldown: 60

# Full deployment with all options
docker_image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
ecs_cluster: production-cluster
ecs_service: api-service
task_definition_family: api-task
container_name: api-container
cpu: "1024"
memory: "2048"
desired_count: 3
environment_variables: '[{"name":"NODE_ENV","value":"production"},{"name":"LOG_LEVEL","value":"info"}]'
secrets: '[{"name":"DB_PASSWORD","valueFrom":"arn:aws:secretsmanager:us-east-1:123456789:secret:prod-db-pass"}]'
mount_points: '[{"sourceVolume":"data-volume","containerPath":"/app/data","readOnly":false}]'
volumes: '[{"name":"data-volume","host":{"sourcePath":"/mnt/efs/data"}}]'
aws_region: us-east-1
```

## CPU and Memory Combinations

Valid combinations for Fargate:

| CPU | Memory Options (MB) |
|-----|-------------------|
| 256 | 512, 1024, 2048 |
| 512 | 1024, 2048, 3072, 4096 |
| 1024 | 2048, 3072, 4096, 5120, 6144, 7168, 8192 |
| 2048 | 4096 to 16384 (1GB increments) |
| 4096 | 8192 to 30720 (1GB increments) |

## Troubleshooting

- **Authentication errors**: Verify your AWS credentials and IAM permissions
- **Task definition errors**: Ensure CPU/memory combinations are valid
- **Service update fails**: Check that the service and cluster names are correct
- **JSON parsing errors**: Validate your JSON format for environment variables, mounts, and volumes

## Notes

- The workflow uses the existing task definition as a base and updates only the specified parameters
- A new task definition revision is created on each deployment
- The workflow waits for the service to reach a stable state before completing
- Failed deployments will not break the workflow; check the deployment status in the final step
- **Auto-scaling**: When enabled, creates both CPU and memory-based target tracking policies. The service will scale up/down to maintain the target utilization levels
- **Task count**: If both `desired_count` and `enable_autoscaling` are specified, the desired count is set first, then auto-scaling policies are applied
- **Cooldown periods**: Scale-out cooldown is typically shorter (60s) to respond quickly to increased load, while scale-in cooldown is longer (300s) to avoid rapid scale-down during temporary dips
