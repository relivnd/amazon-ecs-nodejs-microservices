# AWS Copilot
AWS Copilot Application is a group of services and environments. Think of it as a label for what you are building...

```shell
$ copilot env init
Environment name: api

  Which credentials would you like to use to create api?  [Use arrows to move, type to filter, ? for more help]
    Enter temporary credentials
  > [profile default]
```

Create Environment. Copilot provisions a secure VPS, ECS cluster, load balancer and all other required resources.
```shell
Environment name: api
Credential source: [profile default]

  Would you like to use the default configuration for a new environment?
    - A new VPC with 2 AZs, 2 public subnets and 2 private subnets
    - A new ECS Cluster
    - New IAM Roles to manage services and jobs in your environment
  [Use arrows to move, type to filter]
  > Yes, use default.
    Yes, but I'd like configure the default resources (CIDR ranges, AZs).
    No, I'd like to import existing resources (VPC, subnets).
```

AWS Copilot creates a **manifest.yml **that is used to configure the environment.
```shell
Environment name: api
Credential source: [profile default]
Default environment configuration? Yes, use default.
✔ Wrote the manifest for environment api at copilot/environments/api/manifest.yml
- Update regional resources with stack set "api-infrastructure"  [succeeded]  [0.0s]
- Update regional resources with stack set "api-infrastructure"  [succeeded]        [130.8s]
  - Update resources in region "us-east-1"                       [create complete]  [130.4s]
    - ECR container image repository for "monolith"              [create complete]  [2.5s]
    - KMS key to encrypt pipeline artifacts between stages       [create complete]  [124.5s]
    - S3 Bucket to store local artifacts                         [create complete]  [2.4s]
✔ Proposing infrastructure changes for the api-api environment.
- Creating the infrastructure for the api-api environment.  [create complete]  [56.0s]
  - An IAM Role for AWS CloudFormation to manage resources  [create complete]  [22.4s]
  - An IAM Role to describe resources in your environment   [create complete]  [25.0s]
✔ Provisioned bootstrap resources for environment api in region us-east-1 under application api.
```

deploy the application
```shell
$ copilot env deploy --name api

✔ Proposing infrastructure changes for the api-api environment.
- Creating the infrastructure for the api-api environment.                    [update complete]  [78.3s]
  - An ECS cluster to group your services                                     [create complete]  [7.5s]
  - A security group to allow your containers to talk to each other           [create complete]  [1.4s]
  - An Internet Gateway to connect to the public internet                     [create complete]  [16.0s]
  - Private subnet 1 for resources with no internet access                    [create complete]  [1.7s]
  - Private subnet 2 for resources with no internet access                    [create complete]  [1.7s]
  - A custom route table that directs network traffic for the public subnets  [create complete]  [10.9s]
  - Public subnet 1 for resources that can access the internet                [create complete]  [3.1s]
  - Public subnet 2 for resources that can access the internet                [create complete]  [6.0s]
  - A private DNS namespace for discovering services within the environment   [create complete]  [46.3s]
  - A Virtual Private Cloud to control networking of your AWS resources       [create complete]  [11.6s]
```


create service. service runs containers.

```shell
$ copilot svc init

Note: It's best to run this command in the root of your Git repository.
Welcome to the Copilot CLI! We're going to walk you through some questions
to help you get set up with a containerized application on AWS. An application is a collection of
containerized services that operate together.


  Which workload type best represents your architecture?  [Use arrows to move, type to filter, ? for more help]
    Request-Driven Web Service  (App Runner)
  > Load Balanced Web Service   (Internet to ECS on Fargate)
    Backend Service             (ECS on Fargate)
    Worker Service              (Events to SQS to ECS on Fargate)
    Scheduled Job               (Scheduled event to State Machine to Fargate)
```

look at manifest.yml
```yaml
# The manifest for the "monolith" service.
# Read the full specification for the "Load Balanced Web Service" type at:
#  https://aws.github.io/copilot-cli/docs/manifest/lb-web-service/

# Your service name will be used in naming your resources like log groups, ECS services, etc.
name: monolith
type: Load Balanced Web Service

# Distribute traffic to your service.
http:
  # Requests to this path will be forwarded to your service.
  # To match all requests you can use the "/" path.
  path: '/'
  # You can specify a custom health check path. The default is "/".
  # healthcheck: '/'

# Configuration for your containers and service.
image:
  # Docker build arguments. For additional overrides: https://aws.github.io/copilot-cli/docs/manifest/lb-web-service/#image-build
  build: 2-containerized/services/api/Dockerfile
  # Port exposed through your container to route traffic to it.
  port: 3000

cpu: 256       # Number of CPU units for the task.
memory: 512    # Amount of memory in MiB used by the task.
platform: linux/x86_64  # See https://aws.github.io/copilot-cli/docs/manifest/lb-web-service/#platform
count: 1       # Number of tasks that should be running in your service.
exec: true     # Enable running commands in your container.
network:
  connect: true # Enable Service Connect for intra-environment traffic between services.

# storage:
  # readonly_fs: true       # Limit to read-only access to mounted root filesystems.

# Optional fields for more advanced use-cases.
#
#variables:                    # Pass environment variables as key value pairs.
#  LOG_LEVEL: info

#secrets:                      # Pass secrets from AWS Systems Manager (SSM) Parameter Store.
#  GITHUB_TOKEN: GITHUB_TOKEN  # The key is the name of the environment variable, the value is the name of the SSM parameter.

# You can override any of the values defined above by environment.
#environments:
#  test:
#    count: 2               # Number of tasks to run for the "test" environment.
#    deployment:            # The deployment strategy for the "test" environment.
#       rolling: 'recreate' # Stops existing tasks before new ones are started for faster deployments.
```

Deploy service. container is built with docker and image pushed to elastic container registry.
```shell
$ copilot svc deploy --name monolith
Only found one service, defaulting to: monolith
Only found one environment, defaulting to: api
Building your container image: docker build -t 837028011264.dkr.ecr.us-east-1.amazonaws.com/api/monolith --platform linux/x86_64 
/Users/sparaaws/github/spara/amazon-ecs-nodejs-microservices/2-containerized/services/api -f 
/Users/sparaaws/github/spara/amazon-ecs-nodejs-microservices/2-containerized/services/api/Dockerfile
[+] Building 43.3s (10/10) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                                          
0.0s
 => => transferring dockerfile: 36B                                                                                                                                           
0.0s
 => [internal] load .dockerignore                                                                                                                                             
0.0s
 => => transferring context: 2B                                                                                                                                               
0.0s
 => [internal] load metadata for docker.io/mhart/alpine-node:7.10.1                                                                                                           
5.6s
 => [auth] mhart/alpine-node:pull token for registry-1.docker.io                                                                                                              
0.0s
 => [internal] load build context                                                                                                                                             
0.0s
 => => transferring context: 392B                                                                                                                                             
0.0s
 => [1/4] FROM docker.io/mhart/alpine-node:7.10.1@sha256:d334920c966d440676ce9d1e6162ab544349e4a4359c517300391c877bcffb8c                                                     
0.0s
 => => resolve docker.io/mhart/alpine-node:7.10.1@sha256:d334920c966d440676ce9d1e6162ab544349e4a4359c517300391c877bcffb8c                                                     
0.0s
 => CACHED [2/4] WORKDIR /srv                                                                                                                                                 
0.0s
 => [3/4] ADD . .                                                                                                                                                             
0.0s
 => [4/4] RUN npm install                                                                                                                                                    
37.1s
 => exporting to image                                                                                                                                                        
0.3s
 => => exporting layers                                                                                                                                                       
0.2s
 => => writing image sha256:26ea1872922a12bd3a297c2dd003d1fc71de93e0e5895d2264acca4db3963fbb                                                                                  
0.0s
 => => naming to 837028011264.dkr.ecr.us-east-1.amazonaws.com/api/monolith                                                                                                    
0.0s

Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them
Login Succeeded

Logging in with your password grants your terminal complete access to your account.
For better security, log in with a limited-privilege personal access token. Learn more at https://docs.docker.com/go/access-tokens/
Using default tag: latest
The push refers to repository [837028011264.dkr.ecr.us-east-1.amazonaws.com/api/monolith]
bc590299ddf7: Pushed
947736d9bac9: Pushed
5f70bf18a086: Pushed
3e893534526a: Pushed
040fd7841192: Pushed
latest: digest: sha256:721adc83096c12f21d61bb73d7ea6d296269cce607178676492aaa1f6cdad6bc size: 1365
```

## Break the monolith
create microservices

```shell
$ copilot svc init --app api --dockerfile ./3-microservices/services/posts/Dockerfile --name posts --svc-type "Load Balanced Web 
Service"

✔ Wrote the manifest for service posts at copilot/posts/manifest.yml
Your manifest contains configurations like your container size and port (:3000).

- Update regional resources with stack set "microservices-infrastructure"  [succeeded]  [0.0s]
Recommended follow-up actions:
  - Update your manifest copilot/posts/manifest.yml to change the defaults.
  - Run `copilot svc deploy --name posts --env test` to deploy your service to a test environment.

$ copilot svc init --app api --dockerfile ./3-microservices/services/threads/Dockerfile --name threads --svc-type "Load Balanced Web 
Service"

$ copilot svc init --app api --dockerfile ./3-microservices/services/users/Dockerfile --name users --svc-type "Load Balanced Web 
Service"

```

edit paths in manifest.yaml

```yaml
# Distribute traffic to your service.
http:
  # Requests to this path will be forwarded to your service.
  # To match all requests you can use the "/" path.
  path: 'api/posts'
```

deploy services

```shell
copilot svc deploy --name posts
Only found one environment, defaulting to: api
Building your container image: docker build -t 837028011264.dkr.ecr.us-east-1.amazonaws.com/api/posts --platform linux/x86_64 
/Users/sparaaws/github/spara/amazon-ecs-nodejs-microservices/3-microservices/services/posts -f 
/Users/sparaaws/github/spara/amazon-ecs-nodejs-microservices/3-microservices/services/posts/Dockerfile
[+] Building 1.6s (10/10) FINISHED
 => [internal] load build definition from Dockerfile                                                                                      
0.1s
 => => transferring dockerfile: 36B                                                                                                       
0.0s
 => [internal] load .dockerignore                                                                                                         
0.0s
 => => transferring context: 2B                                                                                                           
0.0s
 => [internal] load metadata for docker.io/mhart/alpine-node:7.10.1                                                                       
1.0s
 => [auth] mhart/alpine-node:pull token for registry-1.docker.io                                                                          
0.0s
 => [internal] load build context                                                                                                         
0.0s
 => => transferring context: 147B                                                                                                         
0.0s
 => [1/4] FROM docker.io/mhart/alpine-node:7.10.1@sha256:d334920c966d440676ce9d1e6162ab544349e4a4359c517300391c877bcffb8c                 
0.0s
 => => resolve docker.io/mhart/alpine-node:7.10.1@sha256:d334920c966d440676ce9d1e6162ab544349e4a4359c517300391c877bcffb8c                 
0.0s
 => CACHED [2/4] WORKDIR /srv                                                                                                             
0.0s
 => CACHED [3/4] ADD . .                                                                                                                  
0.0s
 => CACHED [4/4] RUN npm install                                                                                                          
0.0s
 => exporting to image                                                                                                                    
0.1s
 => => exporting layers                                                                                                                   
0.0s
 => => writing image sha256:b97d6e203a072a0ed3885414d5cc1f35baa9bbd46f74f122c4a3154a36e5e6a4                                              
0.0s
 => => naming to 837028011264.dkr.ecr.us-east-1.amazonaws.com/api/posts                                                                   
0.0s

Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them
Login Succeeded

Logging in with your password grants your terminal complete access to your account.
For better security, log in with a limited-privilege personal access token. Learn more at https://docs.docker.com/go/access-tokens/
Using default tag: latest
The push refers to repository [837028011264.dkr.ecr.us-east-1.amazonaws.com/api/posts]
8b3af4325a20: Pushed
9b446d573a92: Pushed
5f70bf18a086: Pushed
3e893534526a: Pushed
040fd7841192: Pushed
latest: digest: sha256:82749ddab70e1657feaa97134150e6d39abe32b3710538b37398996dc546e442 size: 1365
✔ Proposing infrastructure changes for stack api-api-posts
- Creating the infrastructure for stack api-api-posts                         [create complete]  [216.9s]
  - Service discovery for your services to communicate within the VPC         [create complete]  [3.6s]
  - Update your environment's shared resources                                [create complete]  [38.0s]
  - An IAM role to update your environment stack                              [create complete]  [19.5s]
  - An IAM Role for the Fargate agent to make AWS API calls on your behalf    [create complete]  [19.5s]
  - A HTTP listener rule for forwarding HTTP traffic                          [create complete]  [2.5s]
  - A custom resource assigning priority for HTTP listener rules              [create complete]  [3.8s]
  - A CloudWatch log group to hold your service logs                          [create complete]  [3.6s]
  - An IAM Role to describe load balancer rules for assigning a priority      [create complete]  [19.5s]
  - An ECS service to run and maintain your tasks in the environment cluster  [create complete]  [111.3s]
    Deployments
               Revision  Rollout      Desired  Running  Failed  Pending
      PRIMARY  2         [completed]  1        1        0       0
  - A target group to connect the load balancer to your service               [create complete]  [12.5s]
  - An ECS task definition to group your containers and run them on ECS       [create complete]  [0.0s]
  - An IAM role to control permissions for the containers in your tasks       [create complete]  [37.0s]
✔ Deployed service posts.
Recommended follow-up action:
  - You can access your service at http://api-a-Publi-7HVMVCJEP59-1269931118.us-east-1.elb.amazonaws.com/api/posts over the internet.
Deploy the threads and users microservices.


$ copilot svc deploy --name threads

$ copilot svc deploy --name users
```

delete monolith

```shell
copilot svc delete --name monolith
```

## Cleanup
```shell
copilot app delete --name api
```
