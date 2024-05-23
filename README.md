# Step-by-step-guide-to-create-EKS-cluster-deploy-a-sample-application

Amazon EKS is AWS managed Highly Available, Scalable Elastic Kubernetes Service; is used to create, manage Kubernetes cluster.
Creating an Amazon EKS (Elastic Kubernetes Service) cluster involves several steps. 
Below is a detailed step-by-step guide to create an EKS cluster.

### Step 1: Install and Configure AWS CLI

1. **Install AWS CLI:**
   Follow the instructions to install the AWS CLI from the [official documentation](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).

2. **Configure AWS CLI:**
   Configure the AWS CLI with your credentials.

   ```sh
   aws configure
   ```

   You will be prompted to enter your AWS Access Key ID, Secret Access Key, default region, and default output format (e.g., json).

### Step 2: Install eksctl

`eksctl` is a command-line tool for creating and managing EKS clusters.

1. **Install eksctl:**
   Follow the instructions to install `eksctl` from the [official documentation](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html).

   For example, on macOS:

   ```sh
   brew tap weaveworks/tap
   brew install weaveworks/tap/eksctl
   ```

   Or on Linux:

   ```sh
   curl --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
   sudo mv /tmp/eksctl /usr/local/bin
   ```

### Step 3: Create an EKS Cluster

1. **Create a Cluster:**
   Use `eksctl` to create a cluster. The following command creates a cluster named `my-cluster` with the default settings:

   ```sh
   eksctl create cluster --name my-cluster --region us-west-2 --nodegroup-name my-nodes --node-type t3.medium --nodes 3
   ```

   - `--name`: Name of the cluster
   - `--region`: AWS region where the cluster will be created
   - `--nodegroup-name`: Name of the node group
   - `--node-type`: EC2 instance type for the nodes
   - `--nodes`: Number of nodes to create in the node group

2. **Wait for Cluster Creation:**
   The creation process can take several minutes. `eksctl` will handle the creation of the EKS control plane, worker nodes, and necessary IAM roles and policies.

### Step 4: Configure kubectl

`kubectl` is the command-line tool for interacting with Kubernetes clusters.

1. **Install kubectl:**
   Follow the instructions to install `kubectl` from the [official documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

2. **Update kubeconfig:**
   Use `eksctl` to update your kubeconfig file with the new cluster's information.

   ```sh
   eksctl utils write-kubeconfig --cluster my-cluster --region us-west-2
   ```

   This command ensures that `kubectl` can communicate with your EKS cluster.

3. **Verify kubectl Configuration:**

   ```sh
   kubectl get nodes
   ```

   You should see a list of the nodes in your cluster.

### Step 5: Install the AWS Load Balancer Controller (Optional)

The AWS Load Balancer Controller manages AWS Elastic Load Balancers for a Kubernetes cluster. Follow these steps to install it.

1. **Install cert-manager:**
   The AWS Load Balancer Controller requires cert-manager for certificate management.

   ```sh
   kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.8.0/cert-manager.yaml
   ```

2. **Install the AWS Load Balancer Controller:**

   ```sh
   eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

   curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

   aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam-policy.json

   eksctl create iamserviceaccount \
     --cluster=my-cluster \
     --namespace=kube-system \
     --name=aws-load-balancer-controller \
     --attach-policy-arn=arn:aws:iam::<your-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
     --approve

   kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=main"

   helm repo add eks https://aws.github.io/eks-charts

   helm repo update

   helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
     -n kube-system \
     --set clusterName=my-cluster \
     --set serviceAccount.create=false \
     --set region=us-west-2 \
     --set vpcId=<your-vpc-id> \
     --set serviceAccount.name=aws-load-balancer-controller
   ```

   Replace `<your-account-id>` with your AWS account ID and `<your-vpc-id>` with your VPC ID.

### Step 6: Deploy a Sample Application

1. **Create a Deployment:**

   ```yaml
   # nginx-deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx:latest
           ports:
           - containerPort: 80
   ```

   Apply the deployment:

   ```sh
   kubectl apply -f nginx-deployment.yaml
   ```

2. **Create a Service:**

   ```yaml
   # nginx-service.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-service
   spec:
     selector:
       app: nginx
     ports:
     - protocol: TCP
       port: 80
       targetPort: 80
     type: LoadBalancer
   ```

   Apply the service:

   ```sh
   kubectl apply -f nginx-service.yaml
   ```

3. **Verify the Deployment:**

   ```sh
   kubectl get deployments
   kubectl get services
   ```

   The `nginx-service` should have an external IP assigned to it by the AWS Load Balancer Controller.

### Conclusion

This step-by-step guide covers the creation of an Amazon EKS cluster using `eksctl`, configuring `kubectl`, and deploying a sample application. For more advanced configurations and management, refer to the official AWS EKS documentation and the `eksctl` documentation.
