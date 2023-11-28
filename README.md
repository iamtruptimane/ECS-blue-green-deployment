# Using Amazon ECS blue-green deployment

A blue/green deployment strategy is a method of updating an existing application by deploying a new version and switching over to it once the new deployment is ready. This approach allows you to perform functional testing of the new deployment to ensure the new version is working correctly. It also allows for easy rollbacks to the previous version.


In this Project we build docker images and deploy using Elastic Container service.

## Environment Before Deployment:

![Environment Before](https://github.com/iamtruptimane/ECS-blue-green-deployment/blob/main/img/env_before.png)

## Environment After Deployment
![Environment After](https://github.com/iamtruptimane/ECS-blue-green-deployment/blob/main/img/env_after.png)

## Pre-configured resources for this project:

## Networking configuration:
 1. VPC
 2. two public subnets
 3. internet gateway 
 4. route table
 5. NACL


 ## compute configuration:
 1. security group
 2. launch template:
    * advanced configuration: add user data
    ```
    #!/usr/bin/env bash
    echo ECS_CLUSTER=ecs-cluster >> /etc/ecs/ecs.config
    ```
    Thsi user data script tells that,Auto Scaling group has been configured in such a way that it will launch container instances that will join an ECS cluster with this name(ecs-cluster)
    
 3. Auto scalling group:
    * Use the latest ECS-optimized Amazon Linux 2 image for 64-bit x86 for the operating system
    * Will attempt to join an Amazon ECS cluster named _ecs-cluster_ 
    * Have an IAM instance profile attached with an IAM role appropriate for ECS container instances
    * Use a pre-configured security group that allows inbound traffic on port 80 and port 8081
    * Are tagged with the name ecs-instance
 4. Application load balencer

 5. Target group

 ## IAM configuration:
    1.CodeBuildServiceRole

    2.ecsTaskExecutionRole



 ## Storage configuration:
 1. Amazon S3 bucket :  add object in s3 bucket

    1. [ecs-blue](https://github.com/iamtruptimane/ECS-blue-green-deployment/tree/main/ecs-blue)
    
    2. [ecs-green](https://github.com/iamtruptimane/ECS-blue-green-deployment/tree/main/ecs-green)

    _NOte : compress these files to zip and upload in Amazon s3 bucket._
## Step 1: Logging In to the Amazon Web Services Console
login to your AWS account using your credentials.

## Step 2: Creating an Elastic Container Repository
Amazon Elastic Container Registry (ECR) hosts your images in a highly available and scalable architecture, it also integrates with the Docker CLI and ECS to allow for easy and convenient push, pull, and deployment operations.

In this step, you will use Amazon ECR to create your own fully-managed Docker container registry within Amazon ECS.

1. In the AWS Management Console search bar, enter ECR, and click the Elastic Container Registry result under Services:

2. Click on the orange Get Started button under Create repository section:

3. In the Repository configuration section, enter _ca-container-registry_ as the Repository name and then click Create repository:

_Warning: Ensure that the container repository name matches ca-container-registry exactly. The build code for the application you will deploy expects this repository to exist._

In this step, you created a repository for Docker images within AWS. This allows you to integrate your Docker images more closely with ECS and other Amazon services, such as CodeBuild.

## Step 3: Building Docker Images with AWS CodeBuild

AWS CodeBuild is a fully-managed build service. CodeBuild can compile code, run tests, and produce packages that you can deploy on any compatible resource. Using CodeBuild in this project eliminates the need for you to create a build server or install any software on your local machine.

In this step, you will use CodeBuild to build the Docker images containing the two applications you need to complete the project.

1. In the search bar at the top, enter CodeBuild, and under Services, click the CodeBuild result:

The Build projects page of AWS Codebuild will build a project and that project builds a docker image from a zip file([ecs-blue](https://github.com/iamtruptimane/ECS-blue-green-deployment/tree/main/ecs-blue), [ecs-green](https://github.com/iamtruptimane/ECS-blue-green-deployment/tree/main/ecs-green)) stored in an Amazon S3 bucket.   

You will create a two project named _ecs-blue-project_ and _ecs-green-project_   that is identical, except that they both uses a different zip file. Both projects will produce Docker images, tagged testblue and testgreen respectively. 

### Build ecs-bule-project:

2. To begin creating the blue CodeBuild project, click Create build project:

3. In the Create build project form, configure the following details, leaving the defaults for options not specified:
* Project configuration:
    * Project name: Enter _ecs-blue-project_
* Source:
    * Source provider: Select Amazon S3
    * Bucket: <your_s3_bucket_name>
    * S3 object key: Enter _ecs-blue.zip_

* Environment:
    * Environment image: Ensure Managed image is selected
    * Operating system: Select Ubuntu
    * Runtime: Select Standard
    * Image: Select aws/codebuild/standard:6.0
    * Image version: Select Always use the latest image for this runtime version
    * Privileged: Checked
    * Service role: Select Existing service role
    * Role ARN: Select the CodeBuildServiceRole role 
    * Allow AWS CodeBuild to modify this service role...: Unchecked

* Buildspec:
    * Build specifications: Ensure Use a buildspec file is selected
    * Buildspec name: Enter buildspec.yml
* Artifacts:
     * Type: Ensure No artifacts is selected

_Warning: Ensure you have checked the Privileged checkbox before proceeding. If this is unchecked, the docker client will be unable to access the host's docker instance, and building the image will fail._

The project uses Ubuntu Linux with Docker installed as the build environment to create the image. The build spec is a collection of build commands and settings that provide instructions on how to build the Docker image and where to push it once complete.

4. To create your code build project, click Create build project:

5. To modify your project's environment, click on Edit > Environment:

6. Uncheck Allows AWS CodeBuild to modify this service role... if it has been selected:

7. Expand Additional configuration, scroll down to Environment variables, and enter the following:
* Name: AWS_ACCOUNT_ID
* Value: <your_AWS_Account_ID>
* Type: Ensure Plaintext is selected

The Account ID is used to construct the ECR repository URI during the build process. The AWS region is also required to construct the URI. However, CodeBuild includes an AWS_REGION environment variable in build environment by default.

8. At the bottom of the page, click Update environment:

9. To start building the blue project, click Start build:

### Build ecs-green-project:
 You will create a second project named _ecs-green-project_ that is identical to the blue project you just created, except that it uses a different zip file([ecs-green](https://github.com/iamtruptimane/ECS-blue-green-deployment/tree/main/ecs-green))

 repeat all the step above except:

 * project name: _ecs-green-project_ 

* zip file name:_ecs-green.zip_

10. Return to the Build projects page and observe that the Last build status for both blue and green projects reports Succeeded:

You have built two Docker images, a blue and green version of the same application.

11. Navigate to the Amazon Elastic Container Registry, and click on the repository named ca-container-registry:

You will see two images listed, named testgreen and testblue:

The application buildspec.yml files([buildspec.yml](https://github.com/iamtruptimane/ECS-blue-green-deployment/blob/main/ecs-blue/buildspec.yml)) tag the container images with testgreen or testblue, depending on the version of the application they contain.

In this step, you used CodeBuild to create two Docker container images with two different applications.

## Step 4: Creating an Amazon ECS Cluster
An Amazon Elastic Container Service (ECS) cluster is a logical grouping of container instances where you can define task definitions to execute tasks.

In this lab step, you will create an ECS cluster using a pre-configured Amazon EC2 Auto Scaling group (ASG), and a pre-configured Amazon Virtual Private Cloud (VPC) network. 

1. In the search bar at the top of the AWS Management Console, enter ECS, and under Services, click the Elastic Container Service result:

2. To navigate to the clusters page, in the left-hand menu, click Clusters:

3. To begin creating a new ECS cluster, in the top-right, click Create cluster:

4. To name your cluster, in the Cluster configuration section, in the Cluster name textbox, enter _ecs-cluster_:

_Warning: Ensure that your cluster name matches ecs-cluster (all lowercase) exactly. The pre-configured Amazon EC2 Auto Scaling group has been configured to launch container instances that will join an ECS cluster with this name. If the name does not match, later lab steps will not work correctly._ 

5. In the rest of the form, configure the following:
* Infrastructure:
    * Deselect AWS Fargate

Note: Ensure all non-greyed-out options in the Infrastructure section are unchecked, you will add infrastructure to the cluster later in this step.

6. To finish creating your ECS cluster, at the bottom of the page, click Create:

The ECS cluster creation process is a fairly complex operation that utilizes the following Amazon services:

* Elastic Compute Cloud (EC2): The cluster is powered by EC2 instances and networking features
* Identity and Access Management (IAM): The EC2 instances require roles with defined policies to securely interact with other services
* AWS CloudFormation:  CloudFormation is the mechanism that deploys and manages your cluster as configured

7. To view details of your cluster, in the clusters list, click ecs-cluster:

### Creating infrastructure for our ecs-cluster:
8. To see infrastructure information about this cluster, in the row of tabs under the Cluster overview, click Infrastructure:

9. To create an Amazon ECS capacity provider for an Amazon EC2 Autoscaling Group, on the right-hand side of the Capacity providers section, click Create:

10. To configure a capacity provider, enter and select the following:
* Basic details:
     Capacity provider name: Enter _ecs-capacity-provider_
* Auto Scaling group:
    * Select the existing ASG whose name begins with <Your_ASG_name>

* Scaling policies (Expand this)
    * Uncheck Turn on managed scaling

You have specified that the cluster will use Amazon EC2 instances in an existing Auto Scaling group to run containers on. 

Auto Scaling group is standard and configured similarly to an ASG that could be used to manage autoscaling for an application running on Amazon EC2 instances without using ECS. 

11. To finish creating the capacity provider, at the bottom of the page, click Create:

12. Scroll down and observe the Container instances table:

You will see one container instance listed.

This Amazon EC2 instance was launched by the pre-configured Amazon EC2 Auto Scaling group that was created earlier. Now that you have created a cluster named ecs-cluster, the instance has joined your Amazon Elastic Container Service cluster.

In this step, you created a new Amazon ECS cluster, and you configured it to use a pre-existing VPC and ASG.

## Step 5: Creating Amazon ECS Task Definitions
Task definitions are required to run Docker containers in Amazon ECS. They tell the services which Docker images to use for the container instances, what kind of resources to allocate, network specifics, and other details.

In this step, you will create two task definitions, one for your blue application and one for your green application.

1. In the Amazon ECS console, in the left-hand menu, click Task definitions:

2. To begin creating a new task definition, in the top-right, click Create new task definition, and click Create new task definition in the dropdown that appears:

### Create ecs-blue task definition:

3. To configure your task definition, enter and select the following, leaving all other options at their default:

* Task definition configuration:
    * Task definition family: Enter _ecs-blue-taskdef_
* Infrastructure requirements:
    * Launch type: Deselect AWS Fargate (Serverless), and select Amazon EC2 instances
    * Network mode: default
    * CPU: Enter 0.125 vCPU
    * Memory: Enter 0.25 GB
    * Task role: ecsTaskExecutionRole
    * Task execution role: ecsTaskExecutionRole
* Container - 1:
    * Name: Enter _ecs-blue-container_
    * Image URI: Enter the URI of the testblue Docker image URI you made a note of previously
    * Container port: Change to 8081

* Logging:
    * Uncheck Use log collection

8081 is the port that the blue and green applications listen on when they are run.

Amazon ECS supports creating task definitions comprised of multiple containers. You have specified CPU and memory limits at the task level. You can also do so for each container in the task definition. In this lab, as there is only one container in the definition, you do not need to specify container-level limits.

4. To finish creating the task definition for the blue container, scroll to the bottom and click Create:

5. Return to the task definitions page by clicking Task definitions in the breadcrumb navigation at the top of the page:

6. Repeat the instructions for creating a task definition, with the following changes:

### Create ecs-green task definition:
* Enter ecs-green-taskdef for the Task definition family
* Enter _ecs-green-container_ for the container name
* Use the URI of the testgreen image from your Amazon ECR repository for the Image URI
* For all other options, configure the same values as you did for the blue container.

In this step, you created two task definitions for your application. You specified information about the Docker container images, resources, and networking details to use.

## Step 6: Creating Amazon ECS Services
An ECS service is a mechanism that allows ECS to run and maintain a specified number of instances of a task definition. If any tasks or container instances should fail or stop, the ECS service scheduler launches another instance to replace it. This is similar to Auto Scaling in that it maintains a desired number of instances, but it does not scale instances up or down based on CloudWatch alarms or other Auto Scaling mechanisms. Services behind a load balancer provide a relatively seamless way to maintain a certain amount of resources while keeping a single application reference point.

In this step, you will create two services, one for your blue application and one for the green application.

1. In the Amazon ECS console, in the left-hand menu, click Clusters, and in the table, click ecs-cluster:

2. To begin creating a new Amazon ECS service, in the Services tab at the bottom of the page, click Create:

## create a service for Blue app:
3. Enter and select the following to configure a service for your blue application:

* Environment:
    * Compute Options: Launch type
    * Launch Type: EC2
* Deployment configuration:
    * Application type: Ensure Service is selected
    * Family: Select ecs-blue-taskdef
    * Revision: Ensure the selected Revision is LATEST
    * Service name: Enter _ecs-blue-service_
    * Desired tasks: Replace 1 with 2

4. Scroll down and click on Load balancing to expand the load balancing section:

An application load balancer and appropriate related resources we have been pre-configured for this project.

5. Enter and select the following to configure a public-facing load balancer for your Amazon ECS service:

* Load balancer type: Select Application Load Balancer
* Application Load Balancer: Select Use an existing load balancer
* Load balancer: Select <Your_ALB_name>
* Listener: Select Use an existing listener
* Listener: Select 80:HTTP
* Target group: Select Use an existing target group
* Target group name: Select <Your_target_group_name>

Accept the defaults for all other options on this page.

6. When ready, at the bottom of the page, click Create to finish creating your blue Amazon ECS service:

7. To view the tasks in your service, in the row of tabs, click Tasks:

### create a service for Green app:
8. Repeat the instructions for creating a service, with the following changes:

* Deployment configuration:
    * Family: Select _ecs-green-taskdef_
    * Service name: Enter _ecs-green-service_
    * Desired tasks: Replace 1 with 0

* For all other options, configure the same values as you did for the blue service

Configure the Load balancing options to be the same as those for the blue service.

Notice that you have set Desired tasks to zero. A service with zero tasks will not launch any container instances upon creation. This allows you to prepare for future operations before your deployment is fully ready.

Notice that the blue service has two running tasks and the green service has zero.

In this step, you created two services for your blue and green applications. You learned how these services control the desired capacity. You started two tasks for the blue application upon service creation. These tasks launched and registered two container instances.

## Step 7: Viewing Instances and the Application's Message
Now that you have put all of the pieces in place to build and store Docker images, dynamically register container instances, and maintain a target capacity, it is time to learn more about the interdependent service actions and see some results.

In this step, you will view the resources launched by Amazon ECS and access the blue application launched when you created the blue service.

### observe the Host EC2 instance:
1. In the search bar at the top of the AWS Management Console, enter EC2, and under Services, click the EC2 result:

2. To view Amazon EC2 instances, in the left-hand menu, under Instances, click Instances:

You will see one running instance named _ecs-instance_:
This instance was launched by the Amazon EC2 Auto Scaling group that we configured before as part of the setup of this project.

### observe the container instances deployed by the ECS service:
3. To list load-balancing target groups, in the left-hand menu, under Load Balancing, click Target Groups:

You will see one target group listed named <Your_target_group_name>.

4. To see details of the target group, under Name, click <Your_target_group_name> :

Observe the Registered targets table on the Targets tab:

Notice that you have two healthy targets despite having only one running EC2 instance. ECS deployed two container instances on the EC2 host instance. Each container instance is dynamically registered with the target group on an assigned port in the ephemeral port range.

### Test the Blue application:
5. To navigate to load balancers, in the left-hand menu, under Load Balancing, click Load Balancers:

6. To view details of the load balancer, under Name, click <Your_ALB_name>:

7. Under DNS name, click the copy icon to copy the DNS name of the load balancer to your clipboard:

8. In a new browser tab, paste the DNS name, append /api/ to the end of it, and press enter:

In response, your browser will display:
```
{"message": "Hello - I'm BLUE"}
```

The format or appearance may vary slightly depending on the browser you use. This is a simple JavaScript Object Notation (JSON) message delivered by the application running on the container instances.

In this step, you looked at the resources created by your ECS services and tasks. You also looked at the end result of your application, a JSON message accessible through HTTP.

## Step 8: Updating the Services and Application Deployments
Now that you know how the services interact and how to launch tasks into your Amazon ECS cluster, it is time to switch from one application to the next. A blue-green deployment is a technique that uses two identical production environments to reduce downtime.

In this step, you will launch both applications with all of the container instances you need, including a brief overlap period, to demonstrate the load balancer's round-robin distribution and container resource efficiency.

### Update the desired task of green app:
1. Navigate to the Clusters page of the Amazon ECS console.

2. To access the cluster overview, click ecs-cluster:

3. To select the green service, in the Services tab, click the checkbox next to ecs-green-service:

4. To modify the green service, with it selected, click Update:

5. Change Desired tasks from 0 to 2:

6. To make this change take effect, at the bottom of the page, click Update:

7. Return to the Target Groups section of the Amazon EC2 console and click <Your_target_group_nmme> to view details.

You will now see four instances under Registered targets:

The reduced resource requirements for a Docker container allow you to run multiple container instances on one EC2 instance. This works well for applications that require little resources, such as this simple message application.

### Test the green application:

8. Refresh your browser tab with the DNS name of the load balancer in the address bar.

It may take several refreshes, but you will see the message change:

```
{"message": "Hello - I'm GREEN"}
```
Right now you have both versions of the application running in their own pair of container instances. 

9. Return to the cluster overview page for your cluster in the Amazon ECS console.

### Update the desired task of blue app:
10. Select the ecs-blue-service and click Update:

11. Reduce the Desired tasks field from 2 to 0, and click Update at the bottom of the page:

Optional: Feel free to return to the target group page of the Amazon EC2 console and observe two of the registered targets being deregistered.

12. Refresh your browser tab with the DNS name of the load balancer in the address bar several times.

This time you will only see the message from the green application. It may take a couple of refreshes to see the change take effect.

By swapping the desired tasks of each service, you have manually replicated a blue/green deployment.

## Services used:
All of the services used in this project (most prominently ECS, EC2, CodeBuild, and ECR) are well supported by the AWS command-line interface (CLI), AWS HTTP application programming interface (API), and AWS software development kits (SDK). Using these methods to create, configure, and operate the Amazon ECS and related services can result in fully automated deployments that are customized to your needs and workflow. 

## Summary
In this project, you used AWS CodeBuild along with Amazon Elastic Container Registry to build and store docker images. You then created a new Amazon Elastic Container Service cluster, and the ECS task definitions and ECS services necessary to perform a blue/green deployment. Finally, you verified that the deployments were working, and manually switched from blue to green versions of the application.













































    





















     






    































