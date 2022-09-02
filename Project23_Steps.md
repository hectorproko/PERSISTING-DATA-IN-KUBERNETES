# PERSISTING-DATA-IN-KUBERNETES
PROJECT 23

### PERSISTING DATA IN KUBERNETES

Could be a reusable file
Redoing Using EKS
Starting from Kubernetes on AWS (EKS)


``` bash
aws cloudformation create-stack \
--region us-east-1 \
--stack-name my-eks-vpc-stack \
--template-url https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
```
Output:  
``` bash
{
    "StackId": "arn:aws:cloudformation:us-east-1:199055125796:stack/my-eks-vpc-stack/4cd47ee0-1986-11ed-9200-0ad99519b727"
}
```




### PMANAGING VOLUMES DYNAMICALLY WITH PVS AND PVCS
### PCONFIGMAP
