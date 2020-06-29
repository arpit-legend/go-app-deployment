# Setup Go app with Redis database on AWS kubernetes



# Special features are listed below!
  - goapp application is scalable based on both memory and space.
  - Pods of goapp application and pods of redis server are running  with sync.
  - goapp app is having system's highest priority ever possible.
  - Classical ingress Load Balancer is over service of goapp app.
  - redis service is not exposed outside cluster to for security reasons.
  - Every credentials used in this application even by deployments are protected as secrets.
  - redis pods share persistant volume which is on AWS in form of EBS for data recovery in case of pod crash.
  - Used ECR as container registry for goapp app image.
  - Load testing over application pod is done using python code from Locust.
  
Following steps will help us in:
  - Setup kubernetes cluster on AWS using KOPS.
  - Understand every concepts of Depolyment, Secerts, PriorityClass, Autoscaling, Services and volumes.
  - Create goapp CURD application setup with database connectivity with full security.
  - Understanding more about namespace and it's uses.



> Let's start creating...... 


## Setting up kubernetes cluster using Windows machine

Following steps will help in setting cluster easily:
1. Install KOPS using exe file while linux has bundle.
```sh
         Invoke-WebRequest https://github.com/kubernetes/kops/releases/download/1.12.2/kops-windows-amd64 -OutFile "C:\Kubernetes\kops.exe"
```
2. Setup AWS CLI for shell, enter AWS access key Id and secret access key.
```sh
    aws configure
```

3.  Create S3 bucket for etcd database for kubernetes
```sh
    aws s3api create-bucket --bucket kopsbuckettest --region us-east-1  --acl private
```
3. Set enviroment variables used to configore cluster in future
```sh
    setx AWS_ACCESS_KEY_ID AKIAIOSFODNN7EXAMPLE
    setx AWS_SECRET_ACCESS_KEY wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    setx AWS_DEFAULT_REGION us-east-1
    setx NAME=kopstest1.k8s.local
    setx KOPS_STATE_STORE s3://kopsbuckettest
```
4. Finally create cluster
```sh
    kops create cluster \
    --name kopstest1.k8s.local \
    --zones us-east-1a,us-east-1b \
    --node-count 3 \
    --state s3://kopsbuckettest \
    --yes
```
5. Create a public private key pair and push key in cluster so local access to cluster node is possible.
```sh
    ssh-keygen
    kops create secret --name %NAME% sshpublickey admin -i c:\Users\ja\.ssh\id_rsa.pub
```
6. Finally update cluster.
```sh
    kops update cluster %NAME% --yes
```
7. Run few basic command to check all setup is done.
```sh
    kops validate cluster
    kops get ig nodes --name %NAME%
    kubectl config use-context %NAME%
```


 
## Now set up ECR(Elastic Container repository) for images.
1. Create ECR repository named "goapp". 
```sh
    aws ecr create-repository \
    --repository-name goapp \
    --image-scanning-configuration scanOnPush=true
```
2. Get ECR credentials and login as docker login in repository to push images.
```-sh
aws ecr --region=us-east-1 get-authorization-token --output text --query authorizationData[].authorizationToken | base64 -d | cut -d: -f2
```
   (Note: It renews in 12 hours)
    For this I created a shell script to create docker login secret. So we have to run shell only Enter username as AWS and password from -->"aws ecr --region=us-east-1 get-authorization-token --output text --query authorizationData[].authorizationToken | base64 -d | cut -d: -f2"
```sh
    ./ecr/ecr-secret.sh
    docker login 720472024431.dkr.ecr.us-east-1.amazonaws.com/goapp
```
3. Get into our goapp app and look at config for database. Hostname is service endpoint as it in other namespace not in application pod because of security. 

4. Build docker image of goapp app and push it into ECR

```sh
    
    docker built -t goapp .
    docker tag goapp 720472024431.dkr.ecr.us-east-1.amazonaws.com/goapp
    docker push 720472024431.dkr.ecr.us-east-1.amazonaws.com/goapp

```
## Now comes our setup for Kubernetes.

1. Setup namespace and resouce quota:
```sh
cd goapp-redis-kubernetes/namespace-quota
 kubectl create -f namespace-quota.yml
 ```

2. Setup database credentials for security purpose, not to expose our password in manifest file of redis-controller (if needed)

```sh
cd goapp-redis-kubernetes/secret
kubectl create -f secret.yml
```
3. Create replication controller for redis database having persistant volume, so that all pods share same volume on AWS EBS. So that if any pod crashes our data persists.
--- Create Persistant volume and persistant volume claim
```sh
    cd goapp-redis-kubernetes/voulme
    kubectl create -f storage-class.yml
    kubectl create -f pv-claim.yml
```
  --- Create redis controller and its serivce, Serivce i made in such way that it is accesible only within cluster, not from outside for security of database.
```sh
    cd goapp-redis-kubernetes/Database
    kubectl create -f redis-controller.yml
    kubectl create -f redis-service.yml
```
4. Now our redis service is running on default namespace. I created goapp app with highest system priority with priorityClassName: system-cluster-critical having namespace as: kube-system using this pods of application gets highest priority like master node services. Based on image pull from ecr protected with credentials.

```sh
    cd goapp-redis-kubernetes/web-app
    kubectl create -f web-deployment.yml
    kubectl create -f web-service.yml
```
This app service expose pods with classical ELB ingress.

5. Now comes our autoscaling pods part. I mapped web-deployment with HorizontalPodAutoscaler so that it scales pod when 50% of CPU is used along with 60% of memory. It maintaines minimum 7 replicas of application pod.
Beofore jumping to autoscale, I would like to suggest to apply heapster over admin level so that autoscaling works in actual because it is responsible for meteric collection
--sh
    cd goapp-redis-kubernetes/metrics
    kubectl create -f .
```


```sh
    cd goapp-redis-kubernetes/autoscale
    kubectl create -f auto-scale.yml
```

Results in following:
```sh
NAME         REFERENCE                   TARGETS                     MINPODS   MAXPODS   REPLICAS   AGE
hpa-goapp   Deployment/web-controller   33742262857m/30Mi, 0%/50%   7         20        7          6h1m
```
6. Now all done. We are ready to see goapp  app running
```sh
C:\Users\SG0306821>kubectl get service -n kube-system
    NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)         AGE
    kube-dns               ClusterIP      100.64.0.10      <none>                                                                   53/UDP,53/TCP   3d2h
    kubernetes-dashboard   ClusterIP      100.67.33.43     <none>                                                                   443/TCP         2d5h
    metrics-server         ClusterIP      100.68.249.123   <none>                                                                   443/TCP         2d2h
    web                    LoadBalancer   100.70.167.56    a465e443f853411eaa0fc0ef1357c39a-383398374.us-east-1.elb.amazonaws.com
```
7.  Kubernetes on KOPS doesn't provide its own dashboard, So i created for more graphical visibility. Get get into dashboard and everything is ready for you.
```sh
    cd goapp-redis-kubernetes/dashboard
    kubectl create -f sample-user.yml
```

It needs admin credentials to view whole. Will show demo while presenting.

  