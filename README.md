# Jenkins Pipeline: Push Docker Image to AWS ECR & Build Docker Container on AWS ECS


## This is a Jenkins tutorial that explains how to automate a pipeline to push a docker image to AWS ECR and build it in AWS ECS 

![Architecture](/Jenkins-Github.png)

## Overview of the project
In this project, I used my [HomeEstimates web app](https://github.com/obreo/HomeEstimates) as a docker file stored in a GitHub repository. Then I used a Jenkins container with Docker in docker image (DinD) that would be triggered with a push event to the repository.
 
Once Triggered, a Jenkins pipeline will run a Jenkinsfile that would run a set of stages, starting from source code checkout, then build a docker image of the web app from the dockerfile and push it to an AWS ECR repository. 

Finally, the pipeline will run a stage to start a container on AWS ECS using the web app image.


## Prerequisites:
1. Instance to run Jenkins container with Docker; this could be a host server or your local machine.
2. Ngrok: If the local machine is used for Jenkins. This would expose the local network to a public URL on NAT Gateway securely.
3. GitHub Repository to host your web app files
4. AWS account.

## Discussion

We'll start by configuring each part alone and then connect them together:

### Setting AWS Account

#### Things to do: 
1. Create a Security Group.
2. Create ECR Repository.
3. Create ECS Cluster and Task definition.
4. Create a User with restricted authentication to manage ECR & ECS

#### Creating security group:
**Step. 1:** From the AWS Dashboard, search for EC2, then click on security groups from the left navigation bar.

**Step. 2:** Create a new security group and choose edit inbound rules with:

1. HTTP
2. HTTPS
3. SSH
4. Any port required for your application.

#### Creating ECR Repository:
From the AWS Dashboard, browse into ECR, click start, assign the repo name & choose private.

NOTE: It's better to make the ECR repo, and other resources with the same name so it can be modified easily.

#### Creating ECS Cluster & Task Definition:
We'll create a Fragate cluster, which is a serverless service to run the containers:

**Cluster:**
1. From the cluster's tab, create a new cluster.
2. Name it and choose Fargate.

**Task Definition**
A task is used to define the container and the spec capacity to run it.

1. From the Task definitions tab, create a new task definition.
2. Choose Name it, choose Fargate, Linux and decide the container's capacity in the Task Size.
3. For the role, choose TaskExecutionRole. If not available, you may create it from IAM roles.
4. In the next section, add a new container and set it to *Primary*, the name would be your container's name. The repository URI is your ECR's container repository URI.
5. In the Container's port, set your web application's port.
6. Leave the rest as default and create.


#### Creating restricted User account:
This user will have two primary roles:

* To access Jenkins to push Docker image to ECR repo, and pass an IAM role to do so.
* To give access to run ECS service.

**Step. 1**: From AWS Dashboard, Open IAM, create a new policy, and attach:

1. *ECR policy* with all actions, specify the resource with the ARN of the ECR repo you've created.
2. *ECS policy*.
3. *IAM:PassRole* policy only.

**Step. 2:** Create a new user and attach the policy you've created.

**Step. 3:** Create an Access Key for the new User account and download the credentials because we'll need to use them later.


### Setting up Ngrok Server
1. [Download](https://ngrok.com/) the software.
2. Run the PowerShell from the path directory.
3. Run the command `ngrok http 8080`.
4. Follow the server's instructions to get longer sessions.

After configuration, keep the server running in the PowerShell and don't exit.

**Alternative:** you may use the EC2 instance and use its Public Ip as an end point.

### Setting up GitHub and App Repository

**Step. 1: Setting The Repository:** Create a new repository that would be private or public, then push all your app files to it including the **jenkinsfile** that has the pipeline workflow and the **dockerfile** that would build your application.


**Step. 2: Setting Github WebHooks:** Github webhooks will help us trigger Jenkins when a certain event happens. 

1. From the repository settings, scroll to GitHub hooks, then add a webhook.
2. For the Payload URL, add the Ngrok endpoint given to you with the suffix `/github-webhook/`:
```
https://NGROK-ENDPOINT.com/github-webhook/
```
3. Keep the content type as `application/jason`
4. Keep the rest the same.
5. For events, use `Just the push event.`

**Step. 3: Setting GitHub Credentials to Authenticate Jenkins:**

To let Jenkins connect to your GitHub account it needs to have access. To do so:
1. Browse to account *settings*, then scroll to *Developer Settings* > *Personal Access Token* > *Tokens*.
2. Create a new Token (Classic), Name it, and choose Repo from *select scopes*.

You'll get a token, **copy it somewhere in your local machine as we'll use it later.**

### Setting Up Jenkins container with DinD
Running the Jenkins container with a docker container is a bit tricky and complicated. But it's easy and straight to the point if you use the Jenkins [Documentation](https://www.jenkins.io/doc/book/installing/docker/). Highly recommended for this process.

Once installed, we need to configure the following:
1. Browse to the Public IP of your Jenkins machine with an 8080 port.
2. Jenkins will give you an Admin password that you'll figure out by the following:

```
# Check the container's ID:
Docker ps

# Get the stored Jenkins password: 
docker exec -it <DOCKER-ID> cat /var/jenkins_home/secrets/initialAdminPassword
```
Copy the password & insert it in the Jenkins field.
3. Choose let Jenkins install default plugins. Then set your new password and username.
4. From the *Manage Jenkins* tab, click on plugins and install the following:

* Docker
* Docker-API
* AWS Credentials
* AWS Steps

5. From *Manage Jenkins*, click on *Credintials* and add a new jenkins credintial. Choose **Type: Username & Password**.

Where *Username is the GitHub account username*. *Password is the Token* we've saved previously from GitHub Tokens. 

The ID is the label of these credentials.

6. Again from Credentials, create a new credential with **Type: AWS credentials**.

Insert the *Access Key and Secret Keys* that were downloaded from AWS IAM's user account.

7. We need to install AWS CLI in the Jenkins container. This would let us manage our AWS services using the credentials we created:


From AWS CLI [Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#cliv2-linux-install):


This can be installed by the following:
```
# Login as root in Jenkins Container:
Docker exec -it -u root <DOCKER-ID> /bin/bash

# Download the AWSCLI package:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Unzip the package:
unzip awscliv2.zip

# Install:
./aws/install
```

### Running Jenkins Pipeline:

1. From Jenkins Dashboard, click on `New Item > Enter Item Name & Choose Pipeline > Create`.

2. Now we'll configure the following:

**General:**

* Check **Github Project** & add your repository's URL:
```
https://github.com/USER/REPOSITORY_NAME/
```

**Build Triggers:**
* Check **GitHub hook trigger for GITScm polling**.

**Pipeline > Definition**

* Check Pipeline Script from SCM
* SCM type: Git
* add the Git URL of your GitHub repository:
```
In your repo: Code > HTTPS > Copy URL
```
* Choose the GitHub credentials that we stored in Jenkins.

*Note: If your repo is public you won't need to add the credentials step.*

**Script Path**

The name of your Jenkinsfile in Github.

*Note: Your script path must be exactly named as what you've named your file in the GitHub repo, otherwise it'll fail.* 

### Configuring the Jenkins Pipeline:

The Pipeline workflow will run as the following:

`Checkout the source code >> Build the Docker Image based on the Dockerfile stored in the repo >> Push the Docker Image to AWS ECR >> Run the Image as a container in AWS ECS based on the AWS ECR repository.`

Here is a sample:
```
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                // Check the source-code, e.g. GitHub.
                checkout scm
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    // Build Docker image
                    sh 'docker build -t IMAGE:latest .'

                    // Authenticate and push to ECR (check it in your AWS ECR repo by "View Push Credentials")
                    withAWS(credentials: 'AWS_CREDITNIALS_ID') {
                        sh '''
                        aws ecr get-login-password --region <AWS_REGION> | docker login --username AWS --password-stdin <ECR_LOGIN_ID>
                        docker tag IMAGE:latest <ECR_LOGIN_ID>/IMAGE:latest
                        docker push <ECR_LOGIN_ID>/IMAGE:latest
                        '''
                    }
                }
            }
        }

        stage('Deploy / Update to ECS') {
            steps {
                script {
                    // Define the ECS cluster, service name, and task definition name
                    def ecsCluster = ''//'your-ecs-cluster-name'
                    def ecsService = ''//'your-ecs-service-name'
                    def taskDefinition = ''//'your-task-definition-name'

                    // Create a new ECS service using the existing task definition
                    withAWS(region: 'us-east-1', credentials: 'AWS_CREDITNIALS_ID') {
                        // Create a new Service based on the Task and Cluster
                        //sh "aws ecs create-service --region us-east-1 --cluster ${ecsCluster} --service-name ${ecsService} --platform-version LATEST --task-definition ${taskDefinition} --launch-type FARGATE --desired-count 1 --network-configuration 'awsvpcConfiguration={subnets=[subnet-0cdba1186d],securityGroups=[sg-079fa4bef],assignPublicIp=ENABLED}'"
                        
                        // Update Service based on the Task and Cluster
                        sh "aws ecs update-service --region us-east-1 --service ${ecsService} --cluster ${ecsCluster} --task-definition ${taskDefinition}"
                    }
                }
            }
        }
    }
}

```

Once done, click on Run or push an event to your repo and Jenkins shall start.
