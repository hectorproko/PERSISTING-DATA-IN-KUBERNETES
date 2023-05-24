# PERSISTING-DATA-IN-KUBERNETES
PROJECT 23

### PERSISTING DATA IN KUBERNETES
<!--
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
![logo](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/myeksstack.png)  

Create Cluster: (subnet IDs will change)(So I have to go to the manual pat for them to apear the optiosn and copy from there)(variable)  
``` bash
aws eks create-cluster --profile kube --region us-east-1 --name Project23 --kubernetes-version 1.22 \
   --role-arn arn:aws:iam::199055125796:role/myAmazonEKSClusterRole \
   --resources-vpc-config subnetIds=subnet-095191232e77ae8a2,subnet-0ea7c60ec185bcadf,subnet-0459cc122c4f496bd,subnet-0b34de0afb38812ec
```

![logo](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/nodegroup.png)  

``` bash
hector@hector-Laptop:~$ aws eks create-cluster --profile kube --region us-east-1 --name Project23 --kubernetes-version 1.22 \
>    --role-arn arn:aws:iam::199055125796:role/myAmazonEKSClusterRole \
>    --resources-vpc-config subnetIds=subnet-095191232e77ae8a2,subnet-0ea7c60ec185bcadf,subnet-0459cc122c4f496bd,subnet-0b34de0afb38812ec

CLUSTER arn:aws:eks:us-east-1:199055125796:cluster/Project23    2022-08-11T11:36:02.405000-04:00        Project23       eks.5   
arn:aws:iam::199055125796:role/myAmazonEKSClusterRole   CREATING        1.22
KUBERNETESNETWORKCONFIG ipv4    10.100.0.0/16
CLUSTERLOGGING  False
TYPES   api
TYPES   audit
TYPES   authenticator
TYPES   controllerManager
TYPES   scheduler
RESOURCESVPCCONFIG      False   True    vpc-0e0784e8cc327ad9b
PUBLICACCESSCIDRS       0.0.0.0/0
SUBNETIDS       subnet-095191232e77ae8a2
SUBNETIDS       subnet-0ea7c60ec185bcadf
SUBNETIDS       subnet-0459cc122c4f496bd
SUBNETIDS       subnet-0b34de0afb38812ec
hector@hector-Laptop:~$
```

``` bash
hector@hector-Laptop:~$ aws eks update-kubeconfig --profile kube --region us-east-1 --name Project23

Cluster status is CREATING
hector@hector-Laptop:~$ aws eks update-kubeconfig --profile kube --region us-east-1 --name Project23
Added new context arn:aws:eks:us-east-1:199055125796:cluster/Project23 to /home/hector/.kube/config
hector@hector-Laptop:~$
```
-->
First we need to set up [Kubernetes on AWS (EKS)](https://github.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/blob/main/Project22_Steps.md#kubernetes-on-aws-eks), we will follow the same steps as described in [Project 22](https://github.com/hectorproko/DEPLOYING-APPLICATIONS-INTO-KUBERNETES-CLUSTER/blob/main/Project22_Steps.md)  

To confirm that the Kubernetes cluster is up and running  
``` bash
hector@hector-Laptop:~$ kubectl cluster-info
Kubernetes control plane is running at https://D236EDB340398719073CDF558090376C.gr7.us-east-1.eks.amazonaws.com
CoreDNS is running at https://D236EDB340398719073CDF558090376C.gr7.us-east-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
hector@hector-Laptop:~$
```

Environment Ready To Follow Steps  

Lets see what it looks like for our Nginx pod to persist data using `awsElasticBlockStore` volume  

Before you create a volume, lets run the nginx deployment into kubernetes without a volume.  

``` bash
hector@hector-Laptop:~/Project23$ kubectl get pods
No resources found in default namespace.

hector@hector-Laptop:~/Project23$ kubectl apply -f nginx-pod.yaml
deployment.apps/nginx-deployment created

hector@hector-Laptop:~/Project23$ cat nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

hector@hector-Laptop:~/Project23$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6fdcffd8fc-j2jtt   1/1     Running   0          22s
nginx-deployment-6fdcffd8fc-l75gk   1/1     Running   0          22s
nginx-deployment-6fdcffd8fc-zxk9p   1/1     Running   0          22s
hector@hector-Laptop:~/Project23$
```

Tasks  
	• Verify that the pod is running  
	• Check the logs of the pod  

``` bash
hector@hector-Laptop:~/Project23$ kubectl logs nginx-deployment-6fdcffd8fc-j2jtt
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2022/08/11 17:36:08 [notice] 1#1: using the "epoll" event method
2022/08/11 17:36:08 [notice] 1#1: nginx/1.23.1
2022/08/11 17:36:08 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6)
2022/08/11 17:36:08 [notice] 1#1: OS: Linux 5.4.204-113.362.amzn2.x86_64
2022/08/11 17:36:08 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/08/11 17:36:08 [notice] 1#1: start worker processes
2022/08/11 17:36:08 [notice] 1#1: start worker process 31
2022/08/11 17:36:08 [notice] 1#1: start worker process 32
```

Exec into the pod and navigate to the nginx configuration file `/etc/nginx/conf.d`  
Open the config files to see the default configuration.  

``` bash
hector@hector-Laptop:~/Project23$ kubectl exec -it nginx-deployment-6fdcffd8fc-j2jtt -- bash

root@nginx-deployment-6fdcffd8fc-j2jtt:/# cat /etc/nginx/conf.d
cat: /etc/nginx/conf.d: Is a directory

root@nginx-deployment-6fdcffd8fc-j2jtt:/# ls /etc/nginx/conf.d
default.conf

root@nginx-deployment-6fdcffd8fc-j2jtt:/# cat /etc/nginx/conf.d/default.conf
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}

root@nginx-deployment-6fdcffd8fc-j2jtt:/#
```

 **Create Volume** first we have to check the AZ of the worker node where the pod is running

``` bash
hector@hector-Laptop:~/Project23$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE                              NOMINATED NODE   READINESS GATES
nginx-deployment-6fdcffd8fc-j2jtt   1/1     Running   0          18m   192.168.154.43   ip-192-168-138-68.ec2.internal    <none>           <none>
nginx-deployment-6fdcffd8fc-l75gk   1/1     Running   0          18m   192.168.246.1    ip-192-168-229-214.ec2.internal   <none>           <none>
nginx-deployment-6fdcffd8fc-zxk9p   1/1     Running   0          18m   192.168.190.59   ip-192-168-138-68.ec2.internal    <none>           <none>

hector@hector-Laptop:~/Project23$ kubectl describe nodes ip-192-168-138-68.ec2.internal
Name:               ip-192-168-138-68.ec2.internal
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=t3.medium
                    beta.kubernetes.io/os=linux
                    eks.amazonaws.com/capacityType=ON_DEMAND
                    eks.amazonaws.com/nodegroup=Project23
                    eks.amazonaws.com/nodegroup-image=ami-066d220fc7b27642c
                    failure-domain.beta.kubernetes.io/region=us-east-1
                    failure-domain.beta.kubernetes.io/zone=us-east-1a
                    k8s.io/cloud-provider-aws=502d7fd2fa5d65b376d4116fc6c22958
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-192-168-138-68.ec2.internal
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=t3.medium
                    topology.kubernetes.io/region=us-east-1
                    topology.kubernetes.io/zone=us-east-1a
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 11 Aug 2022 12:51:49 -0400
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  ip-192-168-138-68.ec2.internal
  AcquireTime:     <unset>
  RenewTime:       Thu, 11 Aug 2022 13:55:10 -0400
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Thu, 11 Aug 2022 13:51:31 -0400   Thu, 11 Aug 2022 12:51:47 -0400   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Thu, 11 Aug 2022 13:51:31 -0400   Thu, 11 Aug 2022 12:51:47 -0400   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Thu, 11 Aug 2022 13:51:31 -0400   Thu, 11 Aug 2022 12:51:47 -0400   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Thu, 11 Aug 2022 13:51:31 -0400   Thu, 11 Aug 2022 12:52:19 -0400   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:   192.168.138.68
  Hostname:     ip-192-168-138-68.ec2.internal
  InternalDNS:  ip-192-168-138-68.ec2.internal
Capacity:
  attachable-volumes-aws-ebs:  25
  cpu:                         2
  ephemeral-storage:           20959212Ki
  hugepages-1Gi:               0
  hugepages-2Mi:               0
  memory:                      3967476Ki
  pods:                        17
Allocatable:
  attachable-volumes-aws-ebs:  25
  cpu:                         1930m
  ephemeral-storage:           18242267924
  hugepages-1Gi:               0
  hugepages-2Mi:               0
  memory:                      3412468Ki
  pods:                        17
System Info:
  Machine ID:                 ec2ae5d8820f8f02bdbcd3090f0ebcea
  System UUID:                ec2ae5d8-820f-8f02-bdbc-d3090f0ebcea
  Boot ID:                    8f05467e-2bb8-4130-84e8-8b5d837a1c36
  Kernel Version:             5.4.204-113.362.amzn2.x86_64
  OS Image:                   Amazon Linux 2
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://20.10.13
  Kubelet Version:            v1.22.9-eks-810597c
  Kube-Proxy Version:         v1.22.9-eks-810597c
ProviderID:                   aws:///us-east-1a/i-0405e3476ad9160bf
Non-terminated Pods:          (4 in total)
  Namespace                   Name                                 CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                 ------------  ----------  ---------------  -------------  ---
  default                     nginx-deployment-6fdcffd8fc-j2jtt    0 (0%)        0 (0%)      0 (0%)           0 (0%)         19m
  default                     nginx-deployment-6fdcffd8fc-zxk9p    0 (0%)        0 (0%)      0 (0%)           0 (0%)         19m
  kube-system                 aws-node-55j85                       25m (1%)      0 (0%)      0 (0%)           0 (0%)         63m
  kube-system                 kube-proxy-xfxt5                     100m (5%)     0 (0%)      0 (0%)           0 (0%)         63m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource                    Requests   Limits
  --------                    --------   ------
  cpu                         125m (6%)  0 (0%)
  memory                      0 (0%)     0 (0%)
  ephemeral-storage           0 (0%)     0 (0%)
  hugepages-1Gi               0 (0%)     0 (0%)
  hugepages-2Mi               0 (0%)     0 (0%)
  attachable-volumes-aws-ebs  0          0
Events:                       <none>

hector@hector-Laptop:~/Project23$ kubectl describe nodes ip-192-168-229-214.ec2.internal

Name:               ip-192-168-229-214.ec2.internal
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=t3.medium
                    beta.kubernetes.io/os=linux
                    eks.amazonaws.com/capacityType=ON_DEMAND
                    eks.amazonaws.com/nodegroup=Project23
                    eks.amazonaws.com/nodegroup-image=ami-066d220fc7b27642c
                    failure-domain.beta.kubernetes.io/region=us-east-1
                    failure-domain.beta.kubernetes.io/zone=us-east-1b
                    k8s.io/cloud-provider-aws=502d7fd2fa5d65b376d4116fc6c22958
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-192-168-229-214.ec2.internal
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=t3.medium
                    topology.kubernetes.io/region=us-east-1
                    topology.kubernetes.io/zone=us-east-1b
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 11 Aug 2022 12:51:40 -0400
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  ip-192-168-229-214.ec2.internal
  AcquireTime:     <unset>
  RenewTime:       Thu, 11 Aug 2022 13:59:36 -0400
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Thu, 11 Aug 2022 13:58:13 -0400   Thu, 11 Aug 2022 12:51:39 -0400   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Thu, 11 Aug 2022 13:58:13 -0400   Thu, 11 Aug 2022 12:51:39 -0400   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Thu, 11 Aug 2022 13:58:13 -0400   Thu, 11 Aug 2022 12:51:39 -0400   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Thu, 11 Aug 2022 13:58:13 -0400   Thu, 11 Aug 2022 12:52:00 -0400   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:   192.168.229.214
  Hostname:     ip-192-168-229-214.ec2.internal
  InternalDNS:  ip-192-168-229-214.ec2.internal
Capacity:
  attachable-volumes-aws-ebs:  25
  cpu:                         2
  ephemeral-storage:           20959212Ki
  hugepages-1Gi:               0
  hugepages-2Mi:               0
  memory:                      3967476Ki
  pods:                        17
Allocatable:
  attachable-volumes-aws-ebs:  25
  cpu:                         1930m
  ephemeral-storage:           18242267924
  hugepages-1Gi:               0
  hugepages-2Mi:               0
  memory:                      3412468Ki
  pods:                        17
System Info:
  Machine ID:                 ec28587c627ce365417414935ae4fb9a
  System UUID:                ec28587c-627c-e365-4174-14935ae4fb9a
  Boot ID:                    58110d2a-2a9d-4f33-9d71-acfedd49bd4c
  Kernel Version:             5.4.204-113.362.amzn2.x86_64
  OS Image:                   Amazon Linux 2
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://20.10.13
  Kubelet Version:            v1.22.9-eks-810597c
  Kube-Proxy Version:         v1.22.9-eks-810597c
ProviderID:                   aws:///us-east-1b/i-00b40db6d5c47e304
Non-terminated Pods:          (5 in total)
  Namespace                   Name                                 CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                 ------------  ----------  ---------------  -------------  ---
  default                     nginx-deployment-6fdcffd8fc-l75gk    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23m
  kube-system                 aws-node-mqxms                       25m (1%)      0 (0%)      0 (0%)           0 (0%)         67m
  kube-system                 coredns-7f5998f4c-8fxj9              100m (5%)     0 (0%)      70Mi (2%)        170Mi (5%)     136m
  kube-system                 coredns-7f5998f4c-h8sp2              100m (5%)     0 (0%)      70Mi (2%)        170Mi (5%)     136m
  kube-system                 kube-proxy-kd4v9                     100m (5%)     0 (0%)      0 (0%)           0 (0%)         67m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource                    Requests    Limits
  --------                    --------    ------
  cpu                         325m (16%)  0 (0%)
  memory                      140Mi (4%)  340Mi (10%)
  ephemeral-storage           0 (0%)      0 (0%)
  hugepages-1Gi               0 (0%)      0 (0%)
  hugepages-2Mi               0 (0%)      0 (0%)
  attachable-volumes-aws-ebs  0           0
Events:                       <none>
hector@hector-Laptop:~/Project23$
``` 

So we have **3 PODS** running on **2 NODES** with 2 AZ  
`zone=us-east-1a`  
`zone=us-east-1b`  

Picked  `zone=us-east-1a`

![logo](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/createvolume.png)   

``` bash
hector@hector-Laptop:~/Project23$ cat nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "vol-0b62e3d1f63ef4698"
          fsType: ext4

hector@hector-Laptop:~/Project23$ kubectl apply -f nginx-pod.yaml
deployment.apps/nginx-deployment configured

hector@hector-Laptop:~/Project23$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5844d76665-gq748   1/1     Running   0          7s
```

So we checked the running pod, we described the NODE it was running on, and the node was on east-a, the one we picked, seem like before



Go ahead and explore the running pod. Run `describe` on both the **pod** and **deployment**

``` bash
hector@hector-Laptop:~/Project23$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5844d76665-gq748   1/1     Running   0          19m

hector@hector-Laptop:~/Project23$ kubectl describe pod nginx-deployment-5844d76665-gq748
Name:         nginx-deployment-5844d76665-gq748
Namespace:    default
Priority:     0
Node:         ip-192-168-138-68.ec2.internal/192.168.138.68
Start Time:   Thu, 11 Aug 2022 15:20:00 -0400
Labels:       pod-template-hash=5844d76665
              tier=frontend
Annotations:  kubernetes.io/psp: eks.privileged
Status:       Running
IP:           192.168.150.93
IPs:
  IP:           192.168.150.93
Controlled By:  ReplicaSet/nginx-deployment-5844d76665
Containers:
  nginx:
    Container ID:   docker://284c38ada44ee7a5cef34b3fd7369442f99092d745551cde05171e8f8cfb1693
    Image:          nginx:latest
    Image ID:       docker-pullable://nginx@sha256:790711e34858c9b0741edffef6ed3d8199d8faa33f2870dea5db70f16384df79
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 11 Aug 2022 15:20:01 -0400
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fgckn (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  nginx-volume:
    Type:       AWSElasticBlockStore (a Persistent Disk resource in AWS)
    VolumeID:   vol-0b62e3d1f63ef4698
    FSType:     ext4
    Partition:  0
    ReadOnly:   false
  kube-api-access-fgckn:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason                  Age   From                     Message
  ----    ------                  ----  ----                     -------
  Normal  Scheduled               19m   default-scheduler        Successfully assigned default/nginx-deployment-5844d76665-gq748 to ip-192-168-138-68.ec2.internal
  Normal  Pulling                 19m   kubelet                  Pulling image "nginx:latest"
  Normal  Pulled                  19m   kubelet                  Successfully pulled image "nginx:latest" in 132.32775ms
  Normal  Created                 19m   kubelet                  Created container nginx
  Normal  Started                 19m   kubelet                  Started container nginx
  Normal  SuccessfulAttachVolume  19m   attachdetach-controller  AttachVolume.Attach succeeded for volume "nginx-volume"

hector@hector-Laptop:~/Project23$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           123m

hector@hector-Laptop:~/Project23$ kubectl describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Thu, 11 Aug 2022 13:36:01 -0400
Labels:                 tier=frontend
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               tier=frontend
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  tier=frontend
  Containers:
   nginx:
    Image:        nginx:latest
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:
   nginx-volume:
    Type:       AWSElasticBlockStore (a Persistent Disk resource in AWS)
    VolumeID:   vol-0b62e3d1f63ef4698
    FSType:     ext4
    Partition:  0
    ReadOnly:   false
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-5844d76665 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  20m   deployment-controller  Scaled down replica set nginx-deployment-6fdcffd8fc to 1
  Normal  ScalingReplicaSet  20m   deployment-controller  Scaled up replica set nginx-deployment-5844d76665 to 1
  Normal  ScalingReplicaSet  20m   deployment-controller  Scaled down replica set nginx-deployment-6fdcffd8fc to 0
hector@hector-Laptop:~/Project23$
```

To complete the configuration, we will need to add another section to the deployment yaml manifest. The **volumeMounts** which basically answers the question "Where should this Volume be mounted inside the container?" Mounting a volume to a directory means that all data written to the directory will be stored on that volume.

``` bash
hector@hector-Laptop:~/Project23$ cat nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-volume
          mountPath: /usr/share/nginx/
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "  vol-07b537651bbe68be0"
          fsType: ext4
hector@hector-Laptop:~/Project23$ kubectl apply -f nginx-pod.yaml
deployment.apps/nginx-deployment configured
```



### PMANAGING VOLUMES DYNAMICALLY WITH PVS AND PVCS

By default, in EKS, there is a default **storageClass** configured as part of EKS installation.  

Now lets create some persistence for our nginx deployment. We will use 2 different approaches.  
Approach 1  

Create a manifest file for a PVC  

``` bash
hector@hector-Laptop:~$ kubectl get storageclass
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  21m

hector@hector-Laptop:~$ nano nginx-volume-claim.yml

hector@hector-Laptop:~$ kubectl apply -f nginx-volume-claim.yml
persistentvolumeclaim/nginx-volume-claim created

hector@hector-Laptop:~$ kubectl get pvc
NAME                 STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nginx-volume-claim   Pending                                      gp2            23s
hector@hector-Laptop:~$
```

Moved the manifest to **Project23** folder

``` bash
hector@hector-Laptop:~/Project23$ kubectl describe pvc nginx-volume-claim
Name:          nginx-volume-claim
Namespace:     default
StorageClass:  gp2
Status:        Pending
Volume:
Labels:        <none>
Annotations:   <none>
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:
Access Modes:
VolumeMode:    Filesystem
Used By:       <none>
Events:
  Type    Reason                Age                   From                         Message
  ----    ------                ----                  ----                         -------
  Normal  WaitForFirstConsumer  99s (x26 over 7m52s)  persistentvolume-controller  waiting for first consumer to be created before binding
hector@hector-Laptop:~/Project23$
```

``` bash
hector@hector-Laptop:~/Project23$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           29m


hector@hector-Laptop:~/Project23$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   REASON   AGE
pvc-aa96611c-aba1-42c4-b079-243af9ae7212   2Gi        RWO            Delete           Bound    default/nginx-volume-claim   gp2                     4m54s
hector@hector-Laptop:~/Project23$
```

To proceed, simply apply the new deployment configuration below.

2. Then configure the Pod spec to use the PVC

``` bash
hector@hector-Laptop:~/Project23$ cat nginxdeployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-volume-claim
          mountPath: "/tmp/dare"
      volumes:
      - name: nginx-volume-claim
        persistentVolumeClaim:
          claimName: nginx-volume-claim

hector@hector-Laptop:~/Project23$ kubectl apply -f nginxdeployment.yml
deployment.apps/nginx-deployment unchanged

hector@hector-Laptop:~/Project23$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   REASON   AGE
pvc-aa96611c-aba1-42c4-b079-243af9ae7212   2Gi        RWO            Delete           Bound    default/nginx-volume-claim   gp2                     11m
hector@hector-Laptop:~/Project23$
``` 

EC2 > Elastic Block Store > Volumes  
![logo](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/createvolume2.png)  

Copied the service .yaml from Project 22  

``` bash
hector@hector-Laptop:~/Project23$ cat nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    tier: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

hector@hector-Laptop:~/Project23$ kubectl apply -f nginx-service.yaml
service/nginx-service created

hector@hector-Laptop:~/Project23$ kubectl get svc
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)        AGE
kubernetes      ClusterIP      10.100.0.1      <none>                                                                    443/TCP        4h47m
nginx-service   LoadBalancer   10.100.54.132   a69047daa62674e7392375d178cb0a9b-1787659259.us-east-1.elb.amazonaws.com   80:31491/TCP   14s
hector@hector-Laptop:~/Project23$
```

``` bash
hector@hector-Laptop:~/Project23$ lynx a69047daa62674e7392375d178cb0a9b-1787659259.us-east-1.elb.amazonaws.com   
```

![logo](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/welcomenginx.png)  

``` bash
hector@hector-Laptop:~/Project23$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-64bfb7c874-2jrkh   1/1     Running   0          70s
hector@hector-Laptop:~/Project23$

hector@hector-Laptop:~/Project23$ kubectl apply -f nginx-service.yaml
service/nginx-service created

hector@hector-Laptop:~/Project23$ kubectl get svc
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE
kubernetes      ClusterIP      10.100.0.1       <none>                                                                   443/TCP        5h22m
nginx-service   LoadBalancer   10.100.182.155   aab8c1f0d166c4dfba6efab2c8126f6f-785035398.us-east-1.elb.amazonaws.com   80:30460/TCP   16s
hector@hector-Laptop:~/Project23$ kubectl apply -f nginx-service.yaml
service/nginx-service unchanged

hector@hector-Laptop:~/Project23$ lynx aab8c1f0d166c4dfba6efab2c8126f6f-785035398.us-east-1.elb.amazonaws.com
```


![logo](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/404notfound.png)  


![logo](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/404notfound2.png)  


### PCONFIGMAP

Lets go through the below process so that you can see an example of a `configMap` use case.
1. Remove the **volumeMounts** and **PVC** sections of the manifest and use kubectl to apply the configuration

``` bash
		hector@hector-Laptop:~/Project23$ cat nginxdeployment.yml
		apiVersion: apps/v1
		kind: Deployment
		metadata:
		  name: nginx-deployment
		  labels:
		    tier: frontend
		spec:
		  replicas: 1
		  selector:
		    matchLabels:
		      tier: frontend
		  template:
		    metadata:
		      labels:
		        tier: frontend
		    spec:
		      containers:
		      - name: nginx
		        image: nginx:latest
		        ports:
		        - containerPort: 80
		      volumes:
		      - name: nginx-volume-claim
		        persistentVolumeClaim:
		          claimName: nginx-volume-claim
		
		hector@hector-Laptop:~/Project23$ kubectl apply -f nginxdeployment.yml
deployment.apps/nginx-deployment configured
```


2. port forward the service and ensure that you are able to see the "Welcome to nginx" page

``` bash
	hector@hector-Laptop:~/Project23$ kubectl get service
	NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE
	kubernetes      ClusterIP      10.100.0.1       <none>                                                                   443/TCP        5h31m
	nginx-service   LoadBalancer   10.100.182.155   aab8c1f0d166c4dfba6efab2c8126f6f-785035398.us-east-1.elb.amazonaws.com   80:30460/TCP   8m50s
	hector@hector-Laptop:~/Project23$

```


![logo](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/welcomenginx3.png) 

1. exec into the running container and keep a copy of the **index.html** file somewhere. For example  

``` bash
hector@hector-Laptop:~/Project23$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5b98d885c7-xdn7h   1/1     Running   0          4m43s

hector@hector-Laptop:~/Project23$ kubectl exec -it nginx-deployment-5b98d885c7-xdn7h -- bash
root@nginx-deployment-5b98d885c7-xdn7h:/# cat /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
root@nginx-deployment-5b98d885c7-xdn7h:/#
```

In our own use case here, We will use configMap to create a file in a volume.  

``` bash
hector@hector-Laptop:~/Project23$ cat nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: website-index-file
data:
  # file to be mounted inside a volume
  index-file: |
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>

hector@hector-Laptop:~/Project23$ kubectl apply -f nginx-configmap.yaml
configmap/website-index-file created
```


Update the deployment file to use the configmap in the volumeMounts section  

``` bash
hector@hector-Laptop:~/Project23$ kubectl apply -f nginx-pod
nginx-pod-with-cm.yaml  nginx-pod.yaml          nginx-pod.yamlBAK

hector@hector-Laptop:~/Project23$ kubectl apply -f nginx-pod-with-cm.yaml
deployment.apps/nginx-deployment created

hector@hector-Laptop:~/Project23$ cat nginx-pod-with-cm.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
          - name: config
            mountPath: /usr/share/nginx/html
            readOnly: true
      volumes:
      - name: config
        configMap:
          name: website-index-file
          items:
          - key: index-file
            path: index.html
hector@hector-Laptop:~/Project23$
```

``` bash
hector@hector-Laptop:~/Project23$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7dcdfbd66f-m527b   1/1     Running   0          2m3s

hector@hector-Laptop:~/Project23$ kubectl exec -it nginx-deployment-7dcdfbd66f-m527b -- bash
root@nginx-deployment-7dcdfbd66f-m527b:/# ls -ltr  /usr/share/nginx/html
total 0
lrwxrwxrwx 1 root root 17 Aug 11 21:29 index.html -> ..data/index.html
root@nginx-deployment-7dcdfbd66f-m527b:/#

hector@hector-Laptop:~/Project23$ kubectl get configmap
NAME                 DATA   AGE
kube-root-ca.crt     1      5h50m
website-index-file   1      9m16s
hector@hector-Laptop:~/Project23$

hector@hector-Laptop:~/Project23$ kubectl get configmap
NAME                 DATA   AGE
kube-root-ca.crt     1      5h50m
website-index-file   1      9m16s

hector@hector-Laptop:~/Project23$ kubectl edit cm website-index-file
configmap/website-index-file edited
``` 

![logo](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/configmap.png) 


``` bash
hector@hector-Laptop:~/Project23$ kubectl get service
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE
kubernetes      ClusterIP      10.100.0.1       <none>                                                                   443/TCP        5h54m
nginx-service   LoadBalancer   10.100.182.155   aab8c1f0d166c4dfba6efab2c8126f6f-785035398.us-east-1.elb.amazonaws.com   80:30460/TCP   32m
hector@hector-Laptop:~/Project23$
```

![logo](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/usingconfigmap.png)  

