# README #

This repo is a modern accelerator focused on implementing Karpenter with AWS EKS

This is not a perfects example of terraform as it contains no variable files or backend, but rather a quick and easy way to get your AWS resources up to play around with Karpenter!

# What is Karpenter? #

https://karpenter.sh/

Karpenter is an open source software and widely accepted as a replacement for your cluster autoscaler even by AWS! https://aws.amazon.com/blogs/aws/introducing-karpenter-an-open-source-high-performance-kubernetes-cluster-autoscaler/ 

Karpenter evaluates your EC2 compute needs in your Kubernetes cluster and provisions or decomissions EC2 instances to match your Kubernetes compute needs

You can think of it as a replacement for your Cluster Autoscaler provisioning "just in time" capacity for your cluster to optimize elasticity and cost while also minimizing operational overhead!

# How to use #

The steps below and how I learned and configured this repo can be found here: https://karpenter.sh/v0.19.3/getting-started/getting-started-with-terraform/

Authenticate to AWS using the CLI. Provide access key, secret key, us-east-1, and output format if wanted.

```
aws configure
```

Run terraform. At the top level of the repo and run the below commands to build the infrastructure and deploy the Karpenter helm chart

```
terraform init
terraform plan 
terraform apply -auto-approve
```

If you plan to use spot instances in the demo ensure you run this command from the CLI. You only need to run this once per account.

```
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com
# If the role has already been successfully created, you will see:
# An error occurred (InvalidInput) when calling the CreateServiceLinkedRole operation: Service role name AWSServiceRoleForEC2Spot has been taken in this account, please try a different suffix.
```

Connect to your new EKS cluster

```
aws eks update-kubeconfig --name karpenter-demo
```

Then you can run the deployment.yaml file to get a deployment up to test against

```
kubectl apply -f deployment.yaml
```

This deployment starts with 0 replicas so to trigger the Karpenter event of creating EC2 instances scale out the replicas of the deployment.

```
kubectl scale deployment inflate --replicas 5
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
```

You can then watch in the console EC2 instances spin up or run this command to check the nodes directly in Kubernetes

```
kubectl get nodes
```

# Cleanup #

Run these to commands to cleanup your account and avoid additional charges

```
kubectl delete deployment inflate
kubectl delete node -l karpenter.sh/provisioner-name=default
terraform destroy
aws ec2 describe-launch-templates \
    | jq -r ".LaunchTemplates[].LaunchTemplateName" \
    | grep -i "Karpenter-karpenter-demo" \
    | xargs -I{} aws ec2 delete-launch-template --launch-template-name {}
```