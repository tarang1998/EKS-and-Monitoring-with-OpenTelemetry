# EKS-and-Monitoring-with-OpenTelemetry

- Github Link : https://github.com/open-telemetry/opentelemetry-demoLinks
- Documentation link : https://opentelemetry.io/docs/demo/Links

 
## Phase 1: Environment and Initial Application Setup

### Docker Deployment
 
- Set up a dedicated EC2 instance (minimum large instance with 16GB storage) to install and configure Docker.
- Use the docker-compose.yml file available in the GitHub repository to bring up the service


    - User Data specification for EC2 instance to clone the Github Repo, install Docker and Docker Compose, and build the images   

        ```
        #! /bin/sh
        sudo yum update -y
        sudo yum install git -y
        sudo git clone https://github.com/open-telemetry/opentelemetry-demo.git
        sudo amazon-linux-extras install docker -y
        sudo service docker start
        sudo usermod -a -G docker ec2-user # Adds the ec2-user to the docker group to allow running Docker commands without sudo
        sudo chkconfig docker on # Configures Docker to start automatically when the system boots.
        sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
        sudo chmod +x /usr/local/bin/docker-compose
        cd opentelemetry-demo/
        docker-compose up --force-recreate --remove-orphans --detach

        ```
        
- Validate that all services defined in the docker-compose.yml are up and Ensure this by:
    - Running Docker commands like docker ps and docker-compose logs.
    - Accessing application endpoints (if applicable) and confirming service
- Once the deployment is validated, delete the EC2 instance to clean up


#### Deliverables

- Screenshot of the EC2 instance details (instance type, storage).

    ![EC2 Config](/screenshots/phase1/ec2-config.png)

    ![EC2 Security Config](/screenshots/phase1/ec2-security-config.png)

- Screenshots of the services running (docker ps).

    ![docker ps](/screenshots/phase1/docker-ps.png)

- Screenshots of Docker logs showing the application's startup

    - To view last n logs from every service : docker-compose log --tail n

    ![docker-compose log --tail 5 (1)](/screenshots/phase1/docker-compose-log-1.png)

    ![docker-compose log --tail 5 (2)](/screenshots/phase1/docker-compose-log-2.png)

    - View logs of an individual service : docker log *containerId* -n *n* (Eg. Last n logs from the email service)

    ![docker log <containerId> -n 50](/screenshots/phase1/docker-container-logs.png)


- Screenshot of accessible application

    - Web Store

    ![web store](/screenshots/phase1/webstore.png)

    - Grafana
    
    ![Grafana](/screenshots/phase1/grafana.png)

    - Load Generator UI
    
    ![Load Generator UI](/screenshots/phase1/loadgen.png)

    - Jaeger UI
    
    ![Jaeger UI](/screenshots/phase1/jaeger-ui.png)

    - Flagd configurator UI
    
    ![Flagd configurator UI](/screenshots/phase1/flagd-configurator-ui.png)

### Kubernetes Setup Tasks

- Set up an EKS Cluster in AWS with:
    - At least 2 worker
    - Instance type: large for worker nodes.
- Deploy the application to the EKS cluster using the provided opentelemetry-demo.yaml manifest file.
- Use a dedicated EC2 instance as the EKS client to:
    - Install and configure kubectl, AWS CLI, and eksctl for cluster
    - Run all Kubernetes commands from the EC2 instance (not from a local machine).

        - Create an IAM policy (say, EksAllAccess) to provide complete access to EKS Services

        ```
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": "eks:*",
                    "Resource": "*"
                },
                {
                    "Action": [
                        "ssm:GetParameter",
                        "ssm:GetParameters"
                    ],
                    "Resource": [
                        "arn:aws:ssm:*:<account_id>:parameter/aws/*",
                        "arn:aws:ssm:*::parameter/aws/*"
                    ],
                    "Effect": "Allow"
                },
                {
                    "Action": [
                    "kms:CreateGrant",
                    "kms:DescribeKey"
                    ],
                    "Resource": "*",
                    "Effect": "Allow"
                },
                {
                    "Action": [
                    "logs:PutRetentionPolicy"
                    ],
                    "Resource": "*",
                    "Effect": "Allow"
                }        
            ]
        }
        ```

        - Create an IAM policy (say, IamLimitedAccess) to provided limited access to AWS IAM

        ```
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:CreateInstanceProfile",
                        "iam:DeleteInstanceProfile",
                        "iam:GetInstanceProfile",
                        "iam:RemoveRoleFromInstanceProfile",
                        "iam:GetRole",
                        "iam:CreateRole",
                        "iam:DeleteRole",
                        "iam:AttachRolePolicy",
                        "iam:PutRolePolicy",
                        "iam:UpdateAssumeRolePolicy",
                        "iam:AddRoleToInstanceProfile",
                        "iam:ListInstanceProfilesForRole",
                        "iam:PassRole",
                        "iam:DetachRolePolicy",
                        "iam:DeleteRolePolicy",
                        "iam:GetRolePolicy",
                        "iam:GetOpenIDConnectProvider",
                        "iam:CreateOpenIDConnectProvider",
                        "iam:DeleteOpenIDConnectProvider",
                        "iam:TagOpenIDConnectProvider",
                        "iam:ListAttachedRolePolicies",
                        "iam:TagRole",
                        "iam:UntagRole",
                        "iam:GetPolicy",
                        "iam:CreatePolicy",
                        "iam:DeletePolicy",
                        "iam:ListPolicyVersions"
                    ],
                    "Resource": [
                        "arn:aws:iam::<account_id>:instance-profile/eksctl-*",
                        "arn:aws:iam::<account_id>:role/eksctl-*",
                        "arn:aws:iam::<account_id>:policy/eksctl-*",
                        "arn:aws:iam::<account_id>:oidc-provider/*",
                        "arn:aws:iam::<account_id>:role/aws-service-role/eks-nodegroup.amazonaws.com/AWSServiceRoleForAmazonEKSNodegroup",
                        "arn:aws:iam::<account_id>:role/eksctl-managed-*"
                    ]
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:GetRole",
                        "iam:GetUser"
                    ],
                    "Resource": [
                        "arn:aws:iam::<account_id>:role/*",
                        "arn:aws:iam::<account_id>:user/*"
                    ]
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:CreateServiceLinkedRole"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {
                            "iam:AWSServiceName": [
                                "eks.amazonaws.com",
                                "eks-nodegroup.amazonaws.com",
                                "eks-fargate.amazonaws.com"
                            ]
                        }
                    }
                }
            ]
        }
        ```

       
        - Create an IAM role (say, EKSClientRole) with the following policies, and attach it to the EC2 instance, serving as the EKS client 

            - IamLimitedAccess
            - EksAllAccess
            - AWSCloudFormationFullAccess (AWS Managed Policy)
            - AmazonEC2FullAccess (AWS Managed Policy)

            ![EKSClientRole](/screenshots/phase1/eksclientRole-IAMRole.png)


        - Create a security group for the EC2 instance allowing inbound SSH traffic

        - Launch the EC2 Instance (EKS Client), with the following configs

        ```
        aws ec2 run-instances --image-id ami-0166fe664262f664c --instance-type t2.medium --key-name SSH1 --security-group-ids sg-013069d9f0783aca5 --iam-instance-profile Name=EKSClientRole --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=EKSClient}]" --count 1 --region us-east-1 --user-data  file://./phase1/eksClientStartupScript.sh
        ```   

        - SSH into the EC2 instance 

        - Create the EKS Cluster using the following command 

        ```
        eksctl create cluster -f  /EKS-and-Monitoring-with-OpenTelemetry/phase1/eks-cluster-deployment.yaml
        ```

        ![EKS Cluster creation](/screenshots/phase1/eksctl-cluster-creation.png)

        - On occurrence of the warning while trying to access the cluster details from the AWS console as a root user 

            > Your current IAM principal doesn’t have access to Kubernetes objects on this cluster. This may be due to the current user or role not having Kubernetes RBAC permissions to describe cluster resources or not having an entry in the cluster’s auth config map. Learn more 

            - Check the following [article](https://stackoverflow.com/questions/70787520/your-current-user-or-role-does-not-have-access-to-kubernetes-objects-on-this-eks)

        
        - Deploy the application to the EKS Cluster

        ```
        kubectl apply --namespace otel-demo -f /EKS-and-Monitoring-with-OpenTelemetry/phase1/opentelemetry-demo.yaml
        ```

        ![Kubectl Config](/screenshots/phase1/kubectl-configuration.png)


- Validate the deployment:

    - Ensure all pods and services are running as expected in the otel-demo namespace.

        ```
        kubectl get all -n otel-demo
        ```

    - Access application endpoints through port-forwarding or service

        - Accessing the application 

            - Forward a local port on the EKS Client to a port on a service running within a Kubernetes cluster. This command allows the port-forwarding to be accessed from any IP address, not just localhost

            ```
            kubectl port-forward svc/opentelemetry-demo-frontendproxy 8080:8080 --namespace otel-demo --address 0.0.0.0
            ```

            - Access the application  
            ```
            http://<instance-public-ip>:8080    
            http://<instance-public-ip>:8080/grafana 
            http://<instance-public-ip>:8080/jaeger/ui/search
            http://<instance-public-ip>:8080/loadgen/   
            http://<instance-public-ip>:8080/feature

            ```



    - Collect the cluster details, including node and pod

        ```
        kubectl get nodes -o wide
        ```
            
        ![Kubectl get nodes](/screenshots/phase1/kubectl-get-nodes.png)


        ```
        kubectl get pods -n otel-demo
        ```

        ![Kubectl get pods](/screenshots/phase1/kubectl-get-pods-1.png)


        ```
        kubectl describe nodes
        ```


    - Do not delete the EKS cluster unless explicitly

    ```
    eksctl delete cluster -f  /EKS-and-Monitoring-with-OpenTelemetry/phase1/eks-cluster-deployment.yaml
    ```

#### Deliverables
 
- Screenshot of the EKS cluster configuration details (number of nodes, instance type, ).

    ![Cluster Config](/screenshots/phase1/cluster-config.png)

    ![Cluster Node Group Config](/screenshots/phase1/cluster-node-group-config.png)

- Screenshot of the EC2 instance used as the EKS client (instance type, storage, ).

    ![EKS Client](/screenshots/phase1/eks-client.png)

- Screenshot of kubectl get all -n otel-demo showing the status of pods, services, and deployments.

    ![Kubectl Get All Namespace otel-demo](/screenshots/phase1/kubectl-get-all-n-otel-1.png)


- Screenshot of logs from key application pods to confirm success
    
    ```
    kubectl logs <pod-name> -n otel-demo
    ```


    - Key Components 

        - Accounting Service Pod Logs

        ![Accounting Service Pod Logs](/screenshots/phase1/accounting-service-pod-logs.png)

        - Add Service Pod Logs 

        ![Ad Service Pod Logs](/screenshots/phase1/ad-service-pod-logs.png) 

        - Cart Service Pod Logs

        ![Cart Service Pod Logs](/screenshots/phase1/cart-service-pod-logs.png)

        - Checkout Service Pod Logs

        ![Checkout Service Pod Logs](/screenshots/phase1/checkout-service-pod-logs.png)

        - Email Service 

        ![Email Service Pod Logs](/screenshots/phase1/email-service-pod-logs.png)

        - Fraud Detection Service 

        ![Fraud detection Service Pod Logs](/screenshots/phase1/fraud-detection-service-pod-logs.png)

        - FrontEnd Proxy Pod Log

        ![Frontend Proxy Pod Log](/screenshots/phase1/kubectl-log-frontendproxy.png)

        - Image Service Pod Log

        ![Image Service Pod Log](/screenshots/phase1/image-service-pod-log.png)



    - Dependent Services 

        - Flagd pod logs

        ![FlagD pod log](/screenshots/phase1/flag-d-pod-log.png)



    - Telemetry Components 

        - Grafana Pod logs

        ```
        kubectl logs <grafana-pod-id> -n otel-demo --tail=50
        ``` 

        ![Grafana Pod Log](/screenshots/phase1/grafana-pod-log.png)


        - Jaeger Pod logs

        ![Jaeger Pod Log](/screenshots/phase1/jaeger-ui-pod-log.png)

    

- Exported Kubernetes manifest (opentelemetry-demo.yaml).

    - The yaml file containing the kubernetes configurations : [opentelemetry-demo.yaml](/phase1/opentelemetry-demo.yaml)

- Screenshot of accessible application follow the project documentation link.

    - Accessing Webstore 

        - Port forwarding on the EKS Client to the service running Kubernetes

        ![Kubectl port forwarding - Frontend Proxy](/screenshots/phase1/kubectl-port-forwarding-frontend-1.png)


        - Accessing the application Locally 

        ![Webstore local access](/screenshots/phase1/webstore-local-access.png)

        ![Grafana local access](/screenshots/phase1/grafana-local-access.png)

        ![LoadGen local access](/screenshots/phase1/loadgen-local-access.png)

        ![Jaeger UI Local access](/screenshots/phase1/jaeger-ui-local-access-kubectl.png)

        ![Flagd configurator UI](/screenshots/phase1/flag-d-ui.png)

    

## Phase 2: YAML Splitting and Modular Deployment

### Objective: 

Deploy the application by creating and organizing split YAML files, applying them either individually or recursively, and validating their functionality. Splitting the YAML file is crucial because if any service or pod is down, the corresponding YAML file can be reapplied to make it functional without affecting others. This approach simplifies debugging and deployment management.

### Tasks

- Create folders for resource types to organize the split YAML files by resource type (e.g., ConfigMaps, Secrets, Deployments, Services). Ensure the folder structure is logical and reflects the Kubernetes resources being deployed.

- Apply resources either individually or recursively:
    - Individually apply each file to ensure resources deploy successfully.
    - Alternatively, apply all files recursively from the root folder containing the organized files to deploy everything.

        - SSH into the instance and move to the following path : EKS-and-Monitoring-with-OpenTelemetry/phase2/deployment

        - Deploy the namespace.yaml file 

        ```
        kubectl apply -f /EKS-and-Monitoring-with-OpenTelemetry/phase2/deployment/namespace.yaml
        ```

        - Apply all the resources recursively

        ```
        kubectl apply -f /EKS-and-Monitoring-with-OpenTelemetry/phase2/deployment/open-telemetry --recursive --namespace otel-demo
        ```
- Validate resource deployment by checking the status of pods, services, and Debug any issues by reviewing pod logs or describing problematic resources.

    ```
    kubectl get all -n otel-demo
    ```

- Compress the organized folder of split YAML files into a ZIP file
 
### Deliverables
- Screenshots of the created folder structure containing the split YAML

    ![Folder Structure 1 ](/screenshots/phase2/folderStructure1.png)

    ![Folder Structure 2 ](/screenshots/phase2/folderStructure2.png)

    ![Folder Structure 3 ](/screenshots/phase2/folderStructure3.png)

    - Folder Structure containing the split yaml file : [link](/phase2/treeStructure.txt)


- Screenshots showing successful deployment of each resource (individually or recursively).

    ![Namespace deployment](/screenshots/phase2/kubectl-namespace-deployment.png)

    ![Recursive deployment](/screenshots/phase2/kubectl-recursive-deployment.png)

- Screenshots showing all resources running successfully, including pods, services.

    ![Kubectl get all](/screenshots/phase2/kubectl-get-all-n-otel-demo.png)

- Logs or screenshots verifying the initialization and proper functioning of application

    ![WebStore Access](/screenshots/phase2/web-store-access.png)

    ![Grafana](/screenshots/phase2/grafana.png)

    ![LoadGen](/screenshots/phase2/loadgen.png)

    ![Jaeger UI](/screenshots/phase2/jaeger-ui.png)

    ![Flag d](/screenshots/phase2/flagd.png)

    - Frontend-proxy pod logs 

    ![Frontend-proxy pod logs](/screenshots/phase2/frontend-proxy-logs.png)

    - Grafana pod log

    ![Grafana pod logs](/screenshots/phase2/grafana-pod-log.png)


- A ZIP file containing the organized and split YAML

    - Link to zip file : [zip file](/phase2/deployment.zip)

- A short report explaining the purpose of each resource, steps followed during deployment, and resolutions to any challenges Note : Manage the namespaces properly while deploying the yaml files

    - Reasoning Behind Splitting YAML Files by Application Level

        The decision to split YAML files by application level reflects an organizational strategy to manage each individual microservice in an isolated manner.
        
        This approach offers several key benefits:

        - In a microservices-based architecture, each service operates independently and has its own deployment lifecycle. By splitting YAML files for each microservice, we ensure that each service’s Kubernetes configuration is managed separately. This approach allows for the microservice's to be deployed, scaled, or updated independently, reducing the complexity of dealing with monolithic configurations.

        - This selective deployment reduces downtime and the risk of impacting other services in the system. It also improves the overall deployment speed since only the necessary resources are updated.

        - When a problem occurs within a specific microservice, it is easier to isolate and debug because the deployment artifacts are specific to that service

        - With smaller, service-specific YAML files, the chances of misconfiguring a microservice by accidentally affecting others are minimized.

        - Parallel Development and Deployment: Teams working on different applications can operate independently without conflict.

        - Enhanced Scalability: Facilitates scaling individual microservices or applications based on demand.


    - Purpose of Each Resource

        - Namespace YAML:
        Defines logical segregation for resources, enabling streamlined management and isolation between applications or teams.

        - OpenTelemetry:

            - OpenSearch: Manages data storage and retrieval for telemetry data using StatefulSets for persistence.

            - Jaeger: Provides tracing capabilities to debug distributed applications.

            - OpenTelemetry Collector: Gathers metrics and traces from services for analysis.

            - Prometheus: A metrics-based monitoring tool.

            - Grafana: Visualization of telemetry data and metrics.

            - WebApplication:

                - Core: Contains Kafka and validation services.

                - Backend: Includes microservices like accounting, ad, cart, etc., for functional requirements.

                - Frontend: Handles user-facing components with a proxy for load balancing.
                
                
    - Steps Followed During Deployment
        
        - Organizing Resources:

            - Resources are grouped by their function or application to ensure logical separation. The kubectl configuration files are arranged based on the different microservices

        - Applying YAML Files:

            - Recursive Deployment: A batch deployment is achieved by applying all YAML files recursively from the root directory. This is accomplished by the following command : kubectl apply -f /EKS-and-Monitoring-with-OpenTelemetry/phase2/deployment/open-telemetry --recursive --namespace otel-demo
        
        - Validation:

            - Checked the status of resources (kubectl get pods, kubectl get services) to ensure successful deployment.
            - Used kubectl logs and kubectl describe commands for debugging failed deployments.
        
    - Challenges and Resolutions

        - Dependency Issues: Service Deployment failing due to dependencies between the services 

        - Resolution: Ensuring certain microservices are deployed first before the others. The order of deployment was important as certain services were depended on the other

        - ClusterRole Binding Errors: Figuring out permission management for IAM Roles and Users, thereby enabling the EKS Client and github action workflow to interact with EKS.

        
    - Conclusion

    This structured approach ensures clear separation of concerns, facilitates smoother deployments, and enables efficient management of microservices. The folder hierarchy mirrors the application architecture, making it intuitive for developers and DevOps engineers to manage and troubleshoot resources.
        

# Phase 6: CI/CD Integration
 
## Objective 

Set up a CI/CD pipeline to automate the build, test, and deployment processes for the application, ensuring a streamlined and efficient development workflow

## Tasks
 
### Set Up CI/CD Pipeline:
 
- Configure a CI/CD pipeline using tools like GitHub Actions or GitLab CI/CD to automate the application deployment
- Automate the following processes:
    - Building container images
    - Running automated tests to validate application
    - Deploying the application to the Kubernetes cluster
 

### Enable Rollback Mechanism:
 
- Integrate a rollback mechanism within the pipeline to revert to the last stable deployment in case of failure.
- Validate the rollback process to ensure minimal downtime and application.

### Deliverables

- A fully functional CI/CD pipeline automating build, test, and deployment

    - Created 2 CI/CD pipelines:

        - deployment.yaml : This workflow watches for any changes in the following folders when they are pushed into the main branch
            - 'open-telemetry-demo/src/**' : The subfolders within this contains the source code for the individual microservices 
            - 'phase2/deployment/open-telemetry/**' : The subfolders within this contains the kubectl configurations for all the services 

            The workflow currently checks for changes in the following microservices : accounting-service, ad-service, cart-service and frontend.
            However it could be extended to include all the microservices.

            If any changes are made to the source code of the microservices the following operations are performed:
            - The docker images for those specific microservices whose source codes have been updated are build.
            - The build images are pushed to ECR.
            - The pushed images are retrieved by EKS and the application pods are updated.
            - The status of the rollout is checked.
            - If the deployment fails due to any error, the deployment is rolled back to the previous stable state.

            If any changes are made to the kubectl configuration of the services the following operations are performed:
            - The kubectl configuration of only the updated microservice is reconfigured instead of the whole application.
            - The status of the rollout is checked.
            - In case of an error, the deployment is rolled back to the previous stable state.

        - testing.yaml : This worflow performs integration test on the entire application whenever a Pull request is pushed and is approved validating that the  application is working as intended after changes are made.


- Clear pipeline configuration files with integration into a container

    - The configuration file for deployment.yaml : [deployment.yaml](.github/workflows/deployment.yaml) 

    - The configuration file for testing.yaml : [testing.yaml](.github/workflows/testing.yaml) 

    - ECR Repositories created 

        ![ECR](/screenshots/phase3/ECR.png)



- Logs or screenshots showcasing successful builds, tests, deployments

    - Deployement of the accounting service source code :

        - Changes made to the accounting service source code

        ![Accounting Service Source Code](/screenshots/phase3/accountingServiceUpdate.png)

        - Successfull deployment of the accounting service

        ![Accounting Service Deployment](/screenshots/phase3/ci-cd-pipeline-accounting-service.png)

        - Changes reflected in the pod log

        ![Accounting Service Pod Log changes](/screenshots/phase3/accountingservicepodlog.png)

        - Github Action Worflow : https://github.com/tarang1998/EKS-and-Monitoring-with-OpenTelemetry/actions/runs/12255175216/job/34187702749
        
    - Deployement of the ad service source code :

        - Changes made to the ad service source code

        ![Ad Service Source Code](/screenshots/phase3/adServiceUpdate.png)

        - Successfull deployment of the ad service

        ![Ad Service Deployment](/screenshots/phase3/ci-cd-pipeline-add-service.png)

        - Changes reflected in the pod log

        ![Ad Service Pod Log changes](/screenshots/phase3/adServicePodLog.png)

        - Github Action Worflow : https://github.com/tarang1998/EKS-and-Monitoring-with-OpenTelemetry/actions/runs/12265787202/job/34222655321
        

    - Deployement of the cart service source code :

        - Changes made to the cart service source code

        ![Cart Service Source Code](/screenshots/phase3/cart-service-update.png)

        - Successfull deployment of the cart service

        ![Cart Service Deployment](/screenshots/phase3/cart-service-ci-cd-pipeline.png)

        - Changes reflected in the pod log

        ![Cart Service Pod Log changes](/screenshots/phase3/cart-service-pod-update.png)

        - Github Action Worflow : https://github.com/tarang1998/EKS-and-Monitoring-with-OpenTelemetry/actions/runs/12266214488

    - Deployement of the frontend service source code  :

        - Changes made to the frontend service source code

        ![Frontend Service Source Code](/screenshots/phase3/front-end-code-change.png)

        - Successfull deployment of the frontend service

        ![Frontend Service Deployment](/screenshots/phase3/ci-cd-frontend.png)

        - Changes reflected in the application

        ![Front end application change](/screenshots/phase3/front-end-change.png)

        - Github Action Worflow : https://github.com/tarang1998/EKS-and-Monitoring-with-OpenTelemetry/actions/runs/12266654614/job/34225271565

    - Deployement of the accounting service kubectl configuration:

        - Changes made to the accounting service kubectl configuration

        ![Accounting service kubectl configuration](/screenshots/phase3/kubectl-account-service-update.png)

        - Successfull deployment of the updated kubectl configuration

        ![Kubectl configuration Change](/screenshots/phase3/ci-cd-kubectl-change-accounting-service.png)

        - Kubectl services after deployment 

        ![After deployment](/screenshots/phase3/kubectl-accounting-service-pods.png)


        - Github Action Worflow : https://github.com/tarang1998/EKS-and-Monitoring-with-OpenTelemetry/actions/runs/12266787927
        

    - Deployment of the frontend service kubectl configuration:

        - Changes made to the front service kubectl configuration

        ![Frontend service kubectl configuration](/screenshots/phase3/frontend-deployment-update.png)

        - Successfull deployment of the updated kubectl configuration

        ![Kubectl configuration Change](/screenshots/phase3/ci-cd-frontend.png)

        - Kubectl services after deployment 

        ![After deployment](/screenshots/phase3/frontend-pod-replication.png)


        - Github Action Worflow :https://github.com/tarang1998/EKS-and-Monitoring-with-OpenTelemetry/actions/runs/12266904381

    - CI/CD pipeline for integration testing 

        - The workflow starts when a PR request is approved 

        ![PR Integration testing](/screenshots/phase3/PR-integration-testing.png)

        - The worflow performs test on the microservices 

        ![Integration testing](/screenshots/phase3/integrationTesting.png)

        - The Github Workflow : https://github.com/tarang1998/EKS-and-Monitoring-with-OpenTelemetry/actions/runs/12271442097/job/34238330866 



- Validation of rollback functionality ensuring recovery from deployment

    - We similulated an error by modifying the image URI in frontend deployment file to an incorrect address

    ![Simulated error](/screenshots/phase3/simulatingImagePullBackOffError.png)

    - EKS tries to deploy the configuration, but faces an error

    ![Image Error](/screenshots/phase3/errorWhileDeployment.png)

    - Successful rollback of the kubectl configuration via github actions 

    ![RollBack](/screenshots/phase3/ci-cd-rollout.png)

    - Pod Status after Rollback

    ![Post RollBack Status](/screenshots/phase3/podStatusAfterRollout.png)

    - Github Action Worflow :https://github.com/tarang1998/EKS-and-Monitoring-with-OpenTelemetry/actions/runs/12267909186

- Confirmation of secure management of sensitive data

    - We used github secrets to manage sensitive data instead of exposing the secrets in git 

    ![Git Secrets](/screenshots/phase3/gitSecrets.png)

 



