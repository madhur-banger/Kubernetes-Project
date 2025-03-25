<<<<<<< HEAD


# Cloud-Native Web Voting Application with Kubernetes

This cloud-native web application is built using a mix of technologies. It's designed to be accessible to users via the internet, allowing them to vote for their preferred programming language out of six choices: C#, Python, JavaScript, Go, Java, and NodeJS.

## Technical Stack

- **Frontend**: The frontend of this application is built using React and JavaScript. It provides a responsive and user-friendly interface for casting votes.

- **Backend and API**: The backend of this application is powered by Go (Golang). It serves as the API handling user voting requests. MongoDB is used as the database backend, configured with a replica set for data redundancy and high availability.

## Kubernetes Resources

To deploy and manage this application effectively, we leverage Kubernetes and a variety of its resources:

- **Namespace**: Kubernetes namespaces are utilized to create isolated environments for different components of the application, ensuring separation and organization.

- **Secret**: Kubernetes secrets store sensitive information, such as API keys or credentials, required by the application securely.

- **Deployment**: Kubernetes deployments define how many instances of the application should run and provide instructions for updates and scaling.

- **Service**: Kubernetes services ensure that users can access the application by directing incoming traffic to the appropriate instances.

- **StatefulSet**: For components requiring statefulness, such as the MongoDB replica set, Kubernetes StatefulSets are employed to maintain order and unique identities.

- **PersistentVolume and PersistentVolumeClaim**: These Kubernetes resources manage the storage required for the application, ensuring data persistence and scalability.

## Learning Opportunities

Creating and deploying this cloud-native web voting application with Kubernetes offers a valuable learning experience. Here are some key takeaways:

1. **Containerization**: Gain hands-on experience with containerization technologies like Docker for packaging applications and their dependencies.

2. **Kubernetes Orchestration**: Learn how to leverage Kubernetes to efficiently manage, deploy, and scale containerized applications in a production environment.

3. **Microservices Architecture**: Explore the benefits and challenges of a microservices architecture, where the frontend and backend are decoupled and independently scalable.

4. **Database Replication**: Understand how to set up and manage a MongoDB replica set for data redundancy and high availability.

5. **Security and Secrets Management**: Learn best practices for securing sensitive information using Kubernetes secrets.

6. **Stateful Applications**: Gain insights into the nuances of deploying stateful applications within a container orchestration environment.

7. **Persistent Storage**: Understand how Kubernetes manages and provisions persistent storage for applications with state.

By working through this project, you'll develop a deeper understanding of cloud-native application development, containerization, Kubernetes, and the various technologies involved in building and deploying modern web applications.


### **************************Steps to Deploy**************************


Create EKS cluster with NodeGroup (2 nodes of t2.medium instance type)
Create EC2 Instance t2.micro (Optional)

##IAM role for ec2	
```
{
	"Version": "2012-10-17",
	"Statement": [{
		"Effect": "Allow",
		"Action": [
			"eks:DescribeCluster",
			"eks:ListClusters",
			"eks:DescribeNodegroup",
			"eks:ListNodegroups",
			"eks:ListUpdates",
			"eks:AccessKubernetesApi"
		],
		"Resource": "*"
	}]
}
```

Install Kubectl:
```
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.11/2023-03-17/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo cp ./kubectl /usr/local/bin
export PATH=/usr/local/bin:$PATH
```

Install AWScli:
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Once the Cluster is ready run the command to set context:
```
aws eks update-kubeconfig --name EKS_CLUSTER_NAME --region us-west-2
```

To check the nodes in your cluster run
```
kubectl get nodes
```

If using EC2 and getting the "You must be logged in to the server (Unauthorized)" error, refer this: https://repost.aws/knowledge-center/eks-api-server-unauthorized-error

Clone the github repo
```
git clone https://github.com/N4si/K8s-voting-app.git
```

**Create CloudChamp Namespace**
```
kubectl create ns cloudchamp

kubectl config set-context --current --namespace cloudchamp
```

**MONGO Database Setup**


To create Mongo statefulset with Persistent volumes, run the command in manifests folder:
```
kubectl apply -f mongo-statefulset.yaml
```

Mongo Service
```
kubectl apply -f mongo-service.yaml
```

Create a temporary network utils pod. Enter into a bash session within it. In the terminal run the following command:
```
kubectl run --rm utils -it --image praqma/network-multitool -- bash
```
Within the new utils pod shell, execute the following DNS queries:
```
for i in {0..2}; do nslookup mongo-$i.mongo; done
```
Note: This confirms that the DNS records have been created successfully and can be resolved within the cluster, 1 per MongoDB pod that exists behind the Headless Service - earlier created. 

Exit the utils container
```
exit
```

On the `mongo-0` pod, initialise the Mongo database Replica set. In the terminal run the following command:
```
cat << EOF | kubectl exec -it mongo-0 -- mongo
rs.initiate();
sleep(2000);
rs.add("mongo-1.mongo:27017");
sleep(2000);
rs.add("mongo-2.mongo:27017");
sleep(2000);
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
EOF
```

Note: Wait until this command completes successfully, it typically takes 10-15 seconds to finish, and completes with the message: bye


To confirm run this in the terminal:
```
kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"
```

Load the Data in the database by running this command:
## Note: use langdb not langdb() as shown in the video
```
cat << EOF | kubectl exec -it mongo-0 -- mongo
use langdb;
db.languages.insert({"name" : "csharp", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 5, "compiled" : false, "homepage" : "https://dotnet.microsoft.com/learn/csharp", "download" : "https://dotnet.microsoft.com/download/", "votes" : 0}});
db.languages.insert({"name" : "python", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 3, "script" : false, "homepage" : "https://www.python.org/", "download" : "https://www.python.org/downloads/", "votes" : 0}});
db.languages.insert({"name" : "javascript", "codedetail" : { "usecase" : "web, client-side", "rank" : 7, "script" : false, "homepage" : "https://en.wikipedia.org/wiki/JavaScript", "download" : "n/a", "votes" : 0}});
db.languages.insert({"name" : "go", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 12, "compiled" : true, "homepage" : "https://golang.org", "download" : "https://golang.org/dl/", "votes" : 0}});
db.languages.insert({"name" : "java", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 1, "compiled" : true, "homepage" : "https://www.java.com/en/", "download" : "https://www.java.com/en/download/", "votes" : 0}});
db.languages.insert({"name" : "nodejs", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 20, "script" : false, "homepage" : "https://nodejs.org/en/", "download" : "https://nodejs.org/en/download/", "votes" : 0}});

db.languages.find().pretty();
EOF
```

Create Mongo secret:
```
kubectl apply -f mongo-secret.yaml
```

**API Setup**

Create GO API deployment by running the following command:
```
kubectl apply -f api-deployment.yaml
```

Expose API deployment through service using the following command:
```
kubectl expose deploy api \
 --name=api \
 --type=LoadBalancer \
 --port=80 \
 --target-port=8080
```

Next set the environment variable:

```
{
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $API_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl $API_ELB_PUBLIC_FQDN/ok
echo
}
```

Test and confirm that the API route URL /languages, and /languages/{name} endpoints can be called successfully. In the terminal run any of the following commands:
```
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/go | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/java | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/nodejs | jq .
```

If everything works fine, go ahead with Frontend setup.
```
{
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
echo API_ELB_PUBLIC_FQDN=$API_ELB_PUBLIC_FQDN
}
```

**Frontend setup**

Create the Frontend Deployment resource. In the terminal run the following command:
```
kubectl apply -f frontend-deployment.yaml
```

Create a new Service resource of LoadBalancer type. In the terminal run the following command:
```
kubectl expose deploy frontend \
 --name=frontend \
 --type=LoadBalancer \
 --port=80 \
 --target-port=8080
```

Confirm that the Frontend ELB is ready to recieve HTTP traffic. In the terminal run the following command:
```
{
FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $FRONTEND_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl -I $FRONTEND_ELB_PUBLIC_FQDN
}
```

Generate the Frontend URL for browsing. In the terminal run the following command:
```
echo http://$FRONTEND_ELB_PUBLIC_FQDN
```

Test the full end-to-end cloud native application

 Using your local workstation's browser - browse to the URL created in the previous output.

After the voting application has loaded successfully, vote by clicking on several of the **+1** buttons, this will generate AJAX traffic which will be sent back to the API via the API's assigned ELB.


Query the MongoDB database directly to observe the updated vote data. In the terminal execute the following command:
```
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```

## **Summary**

In this Project, you learnt how to deploy a cloud native application into EKS. Once deployed and up and running, you used your local workstation's browser to test out the application. You later confirmed that your activity within the application generated data which was captured and recorded successfully within the MongoDB ReplicaSet back end within the cluster.
=======
# #TWSThreeTierAppChallenge

## Overview
This repository hosts the `#TWSThreeTierAppChallenge` for the TWS community. 
The challenge involves deploying a Three-Tier Web Application using ReactJS, NodeJS, and MongoDB, with deployment on AWS EKS. Participants are encouraged to deploy the application, add creative enhancements, and submit a Pull Request (PR). Merged PRs will earn exciting prizes!

**Get The Challenge here**

[![YouTube Video](https://img.youtube.com/vi/tvWQRTbMS1g/maxresdefault.jpg)](https://youtu.be/tvWQRTbMS1g?si=eki-boMemxr4PU7-)

## Prerequisites
- Basic knowledge of Docker, and AWS services.
- An AWS account with necessary permissions.

## Challenge Steps
- [Application Code](#application-code)
- [Jenkins Pipeline Code](#jenkins-pipeline-code)
- [Jenkins Server Terraform](#jenkins-server-terraform)
- [Kubernetes Manifests Files](#kubernetes-manifests-files)
- [Project Details](#project-details)

## Application Code
The `Application-Code` directory contains the source code for the Three-Tier Web Application. Dive into this directory to explore the frontend and backend implementations.

## Jenkins Pipeline Code
In the `Jenkins-Pipeline-Code` directory, you'll find Jenkins pipeline scripts. These scripts automate the CI/CD process, ensuring smooth integration and deployment of your application.

## Jenkins Server Terraform
Explore the `Jenkins-Server-TF` directory to find Terraform scripts for setting up the Jenkins Server on AWS. These scripts simplify the infrastructure provisioning process.

## Kubernetes Manifests Files
The `Kubernetes-Manifests-Files` directory holds Kubernetes manifests for deploying your application on AWS EKS. Understand and customize these files to suit your project needs.

## Project Details
🛠️ **Tools Explored:**
- Terraform & AWS CLI for AWS infrastructure
- Jenkins, Sonarqube, Terraform, Kubectl, and more for CI/CD setup
- Helm, Prometheus, and Grafana for Monitoring
- ArgoCD for GitOps practices

🚢 **High-Level Overview:**
- IAM User setup & Terraform magic on AWS
- Jenkins deployment with AWS integration
- EKS Cluster creation & Load Balancer configuration
- Private ECR repositories for secure image management
- Helm charts for efficient monitoring setup
- GitOps with ArgoCD - the cherry on top!

📈 **The journey covered everything from setting up tools to deploying a Three-Tier app, ensuring data persistence, and implementing CI/CD pipelines.**

## Getting Started
To get started with this project, refer to our [comprehensive guide](https://amanpathakdevops.medium.com/advanced-end-to-end-devsecops-kubernetes-three-tier-project-using-aws-eks-argocd-prometheus-fbbfdb956d1a) that walks you through IAM user setup, infrastructure provisioning, CI/CD pipeline configuration, EKS cluster creation, and more.

### Step 1: IAM Configuration
- Create a user `eks-admin` with `AdministratorAccess`.
- Generate Security Credentials: Access Key and Secret Access Key.

### Step 2: EC2 Setup
- Launch an Ubuntu instance in your favourite region (eg. region `us-west-2`).
- SSH into the instance from your local machine.

### Step 3: Install AWS CLI v2
``` shell
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
aws configure
```

### Step 4: Install Docker
``` shell
sudo apt-get update
sudo apt install docker.io
docker ps
sudo chown $USER /var/run/docker.sock
```

### Step 5: Install kubectl
``` shell
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

### Step 6: Install eksctl
``` shell
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### Step 7: Setup EKS Cluster
``` shell
eksctl create cluster --name three-tier-cluster --region us-west-2 --node-type t2.medium --nodes-min 2 --nodes-max 2
aws eks update-kubeconfig --region us-west-2 --name three-tier-cluster
kubectl get nodes
```

### Step 8: Run Manifests
``` shell
kubectl create namespace workshop
kubectl apply -f .
kubectl delete -f .
```

### Step 9: Install AWS Load Balancer
``` shell
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
eksctl utils associate-iam-oidc-provider --region=us-west-2 --cluster=three-tier-cluster --approve
eksctl create iamserviceaccount --cluster=three-tier-cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::626072240565:policy/AWSLoadBalancerControllerIAMPolicy --approve --region=us-west-2
```

### Step 10: Deploy AWS Load Balancer Controller
``` shell
sudo snap install helm --classic
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl apply -f full_stack_lb.yaml
```

### Cleanup
- To delete the EKS cluster:
``` shell
eksctl delete cluster --name three-tier-cluster --region us-west-2
```
- To clean up rest of the stuff and not incure any cost
```
Stop or Terminate the EC2 instance created in step 2.
Delete the Load Balancer created in step 9 and 10.
Go to EC2 console, access security group section and delete security groups created in previous steps
```

## Contribution Guidelines
- Fork the repository and create your feature branch.
- Deploy the application, adding your creative enhancements.
- Ensure your code adheres to the project's style and contribution guidelines.
- Submit a Pull Request with a detailed description of your changes.

## Rewards
- Successful PR merges will be eligible for exciting prizes!

## Support
For any queries or issues, please open an issue in the repository.

---
Happy Learning! 🚀👨‍💻👩‍💻
>>>>>>> 6561c91 (Updated K8s Manifest File)
