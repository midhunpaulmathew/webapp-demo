# webapp-demo
a demo webapp deployed in kubernetes


## install dependencies

follow the below steps to configure your local machine with dependencies

### 1. awscli

1. ```sudo apt  install awscli```
2. ```aws --version```

### 2. kubectl

1. ```curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"```
2. ```chmod +x kubectl```
3. ```sudo mv kubectl /usr/local/bin/```
4. ```kubectl version --client```

### 3. aws iam authenticator

1. ```curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.15.10/2020-02-22/bin/linux/amd64/aws-iam-authenticator```
2. ```chmod +x ./aws-iam-authenticator```
3. ```sudo mv ./aws-iam-authenticator /usr/local/bin```

### 4. terraform

1. ```sudo apt-get install unzip```
2. ```wget https://releases.hashicorp.com/terraform/1.10.5/terraform_1.10.5_linux_amd64.zip```
3. ```unzip terraform_1.10.5_linux_amd64.zip```
4. ```sudo mv terraform /usr/local/bin/```
5. ```terraform --version```

## configure awscli

follow the below steps to configure awscli on your local machine

1. create an access key and secret key for your iam user in your aws account
2. ```aws configure```
3. ```aws sts get-caller-identity```

## clone the repository

clone this repository to a convinient path in your local machine

1. ```git clone https://github.com/midhunpaulmathew/webapp-demo.git```
2. ```cd webapp-demo```

## review terraform configurations

check and modify the terraform files for your setup

1. ```cd ./eks/```
2. *variables.tf* contains the *aws region*, *vpc CIDR* and *kubernetes version*
3. *eks.tf* contains *node group size*, *instance type*, *public access CIDRs* etc..
4. *vpc.tf* contains *cluster name*, *subnet CIDRs*
5. *sg.tf* contains security group rules specific to worker nodes

## provisioning the infrastructure

follow below steps to provision the infrastructure in your aws account

1. terraform init
2. terraform plan
3. terraform apply

>wait for the resources to be get allocated and active

## accessing the kubernetes api

once the resources are up and active, follow the below steps to access the cluster from your local machine

1. ```aws eks update-kubeconfig --region <cluster-region> --name <cluster-name>```
2. ```kubectl cluster-info```
3. ```kubectl get nodes```

## setup aws loadbalancer controller

to access the services to a public frontend, install a cluster add-on. follow the below steps.
> 1. make sure to replace placeholders inside <> with your account specific values \
> 2. policy files and lbc yamls are already included in the repo, you may not want to download them again.

1. ```cd ../lbc/```
2. ```curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json```

3. ```aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json```
	
4. ```oidc_id=$(aws eks describe-cluster --name webapp-eks-s2pNkB5F --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)```

5. ```aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4```

6. `cat >load-balancer-role-trust-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<your-account-id>:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/<oidc-id>>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.region-code.amazonaws.com/id/<oidc-id>:aud": "sts.amazonaws.com",
                    "oidc.eks.region-code.amazonaws.com/id/<oidc-id>:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
                }
            }
        }
    ]
}
EOF`
7. `aws iam create-role \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --assume-role-policy-document file://"load-balancer-role-trust-policy.json"`

8. `aws iam attach-role-policy \
  --policy-arn arn:aws:iam::<your-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --role-name AmazonEKSLoadBalancerControllerRole`

9. `cat >aws-load-balancer-controller-service-account.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: aws-load-balancer-controller
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<your-account-id>:role/AmazonEKSLoadBalancerControllerRole
EOF`

10. ```kubectl apply -f aws-load-balancer-controller-service-account.yaml```
11. `kubectl apply \
    --validate=false \
    -f https://github.com/jetstack/cert-manager/releases/download/v1.13.5/cert-manager.yaml`
12. ```curl -Lo v2_11_0_full.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.11.0/v2_11_0_full.yaml```
13. ```sed -i.bak -e '690,698d' ./v2_11_0_full.yaml```
14. ```sed -i.bak -e 's|your-cluster-name|<your-cluster-name>|' ./v2_11_0_full.yaml```
15. ```kubectl apply -f v2_11_0_full.yaml```
16. ```curl -Lo v2_11_0_ingclass.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.11.0/v2_11_0_ingclass.yaml```
17. ```kubectl apply -f v2_11_0_ingclass.yaml```
18. ```kubectl get deployment -n kube-system aws-load-balancer-controller```

> verify your load balancer controller deployment.

## setup web application


we use an nginx deployment to deploy a static webpage in the kubernetes and access it using the loadbalancer service

> 1. make sure to check and modify the deployment and service yamls for additional customization. \
> 2. also modify the configmap yaml for customizing your webapp code. \
> 3. *webapp-lb.yaml* requires your public subnet ids, replace the placeholders in the yaml with actual values.

1. ```cd ../webapp/```
2. use ```kubectl apply -f <file-name>``` to create the *namespace*, *configmap*, *deployment* and *services*
3. ```kubectl get all -n <namespace>``` to verify the *deployment* and *services*
4. get the *loadbalancer* DNS from the console to access your static webpage.

## Optional: Using a Custom Dockerfile

in the above deployment, we used the official nginx image. if we want to use a custom docker image, we can build and push one to our registry and make use of it in the deployment templates.

to build and push a custom docker image, follow the below steps.

1. ```cd ~/webapp-demo/docker/```
2. ```docker build -t <your-dockerhub-username>/<image-name>:<tag> .```
3. ```docker login```
4. ```docker push <your-dockerhub-username>/<image-name>:<tag>```
5. edit the deployment yaml in *webapp-demo/webapp/* to make use of the above image
6. apply the new deployment template

> you may need to remove the configmap object and its references from the deployment template also, as the custom image contains the application code embedded.