#Install kubernetes on centos7
Step1: Master node

sudo -i
#Disbale Selinux
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
#Enable required firewall port
firewall-cmd  --permanent --add-port=6443/tcp
firewall-cmd  --permanent --add-port=2379-2380/tcp
firewall-cmd  --permanent --add-port=10250/tcp
firewall-cmd  --permanent --add-port=10251/tcp
firewall-cmd  --permanent --add-port=10252/tcp
firewall-cmd  --permanent --add-port=10255/tcp
firewall-cmd --reload
#Load kernerl module
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
#Add all hosts in host file
vi /etc/hosts
10.160.0.2  master
10.160.0.3  worker1

#Configure repository
root@master ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
> [kubernetes]
> name=kubernetes
> baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
> enabled=1
> gpgcheck=1
> repo_gpgcheck=1
> gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
> EOF
# Install Kubeadm and docker package
yum install kubeadm docker -y
#Start and enable kubectl and docker service
systemctl restart docker && systemctl enable docker
systemctl restart kubelet && systemctl enable kubelet
kubeadm init
kubeadm join 10.160.0.2:6443 --token n5sc17.h8ymsymlcyoegfii \
        --discovery-token-ca-cert-hash sha256:c5c185da7b2354c0b56267655300c0b10216f0bb310e35b4cf3fc095e7270467 
#Use below command to use the cluster as root user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config 
#Deploy pod network to the cluster try to run below commands to get status of cluster and pods
kubectl get nodes
kubectl get pods --all-namespaces

#To make the cluster status ready and kube-dns status running, deploy the pod network so that containers of different host  communicated each other. Pod network is the overlay network between the worker nodes. run below command to deploy network
xport kubever=$(kubectl version | base64 | tr -d '\n')
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$kubever"
#Run the following commands to verify the status
kubectl get nodes
kubectl get pods --all-namesapces
#check the cluster info
kubectl cluster-info
*************************************************************
#Now add worker nodes to kubernetes master nodes. perform the following steps on each worker node
1. Disable SELINUX and configure firewall rules on worker nodes. Before disabling Selinux set the hostname on the worker node. /etc/hosts
sudo -i
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
firewall-cmd  --permanent --add-port=10250/tcp
firewall-cmd  --permanent --add-port=10255/tcp
firewall-cmd  --permanent --add-port=30000-32767/tcp
firewall-cmd  --permanent --add-port=30000-6783/tcp
firewall-cmd --reload
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
2. Configure kubernetes repository
[root@worker1 ~]# cat /etc/yum.repos.d/kubernetes.repo 
[kubernetes]
name=kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
3. Install kubeadm and docker
yum install kubeadm docker -y
4. Start and enable docker service
systemctl restart docker && systemctl enable docker
5. Join Worker node to Master
kubeadm join 10.160.0.2:6443 --token n5sc17.h8ymsymlcyoegfii \
        --discovery-token-ca-cert-hash sha256:c5c185da7b2354c0b56267655300c0b10216f0bb310e35b4cf3fc095e7270467 

*************************************************************
Now verify nodes status from master node using kubectl command
kubectl get nodes
-----------------------------------------------------------
#Single container pod
kubectl run tomcat --image=tomcat:8.0
#Switch to pod
kubectl exec -it tomcat -- /bin/bash
#Delete pod
kubectl delete pod tomcat
#Create deployment file tomcat.yml
apiVersion: v1
kind: Pod
metadata:
  name: tomcat
spec:
  containers:
  - name: tomcat-container
    image: tomcat:8.0
kubectl create -f tomcat.yml
kubectl get pod
kubectl get pod -o wide
kubectl get pod tomcat -o yaml
kubectl describe pod tomcat
kubectl exec -it tomcat -- /bin/bash
hostname
kubectl get pods --all-namespaces
exit
kubectl delete pod tomcat
------------------------
#nginx-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    tier: dev
spec:
  containers:
  - name: nginx-container
    image: nginx

kubectl create -f nginx-pod.yml
kubectl get pod
kubectl get pod -o wide
kubectl get pod nginx-pod -o yaml
kubectl describe pod nginx-pod
kubectl exec -it nginx-pod -- /bin/bash
hostname
exit
kubectl delete pod nginx-pod
**********************************************
Replication controller configuration
**********************************************
[root@master deployment]# cat nginx-rc.yml 
#nginx-rc.yml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
spec:
  replicas: 3
  selector:
    app: nginx-app
  template:
    metadata:
      name: nginx-pod
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx-container
        image: nginx
        ports:
        - containerPort: 80
kubectl get pod
kubectl get pod -l app=nginx-app  (filter pods label tag)
Note: Replication controller ensures that specified number of pods are running at any time. RC and Pods are associated with 'lables'
kubectl describe rc nginx-rc
kubectl get po -o wide
Note: Down one node where pod is running after sometime pod again start on another node as RC is managing it
kubectl get po -o wide
---------
Scaling up
---------
kubectl scale rc nginx-rc --replicas=5
kubectl get rc nginx-rc
kubectl get po -o wide
-----------
Scaling down
-----------
kubectl scale rc nginx-rc --replicas=3
kubectl get rc nginx-rc
kubectl get po -o wide

----------
Delete Replication controller
---------
kubectl delete -f nginx-rc.yml
kubectl get rc
*************************
Replicaset:
************************
Ensures that a specified number of pods are running as any time:
Replication controller Vs. Replicaset
Replicaset-> set-based selectors
Replication Controller: equity-based-selectors

equity-based:                            set-based:
operators: = == !=                 operators: In NotIn Exists
Example 			   Example
environment  = production	   environment In (Production,Qa)
tier != frontend		   tier NotIn (frontend,backend)
#Command line
kubectl get pods -l environment=production
kubectl get pods -l 'environment In (production)'

In manifest:                                                          
...
selector:
  environment: production
  tier: fronted
...
Supports: services, replication controller

In menifest:
...
selector:
  matchExpressions:
  - {key: environment,operator:In,values:[prod,qa]}
  - {key: tier,operator:NotIn,values:[frontend,backend]}
Support: Job, Deployment, Replicaset, Damonset

------
Matchlabels
-----
...
selector:
  app: nginx
  tier: frontend
...

Support: Replication controller, Services

...
selector:
  matchLables:
    app: nginx
    tier: frontend
...

Support on new resources, such as: Replicaset,Deployment,Jobs,Damonset
#########
Replicaset manifest file
########
#replicaset.yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-app
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      name: nginx-pod
      labels:
        app: nginx-app
        tier: frontend
    spec:
      containers:
      - name: nginx-container
        image: nginx
        ports:
        - containerPort: 80
kubectl create -f replicaset.yml
kubectl get pods
kubectl get po -l tier=frontend
kubectl get rs nginx-rs -o wide
kubectl describe rs nginx-rs
-------------
Scheduling
------------
If one node down where pods are running replicaset will schedule the pods on another node.

kubectl get nodes
kubectl get po -o wide
-----------
Scaling up
----------
kubectl scale rs nginx-rs --replicas=5
kubectl get rs nginx-rs
kubectl get po -o wide
---------
Scale down
---------
kubectl scale rs nginx-rs --replicas=3
kubectl get rs nginx-rs
kubectl get po -o wide
-------------
Delete Replicaset
-------------
kubectl delete -f replicaset.yml
kubectl get rs
kubectl get po -l app=nginx-app
***********************************************
Deployment: Provide features of updates and rollback but replicaset doesn't.
***********************************************
Featues: multiple replicas,upgrade,rollback,scale-up or down, pause and resume
Deployment type:
recreate: terminate the old version and release the new one
RollingUpdate: release a new version on a rolling update fashion, one after the other
blue/green: release a new version alongside the old version then switch traffic
canary: release a new version to a subset of users, then proceed to a full rollout
https://blog.container-solutions.com/kubernetes-deployment-strategies
https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/recreate 
https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/ramped
https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/blue-green
https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/canary

#nginx-deploy.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.7.9
        ports:
        - containerPort: 80
kubectl get deploy -l app=nginx-app
kubectl get rs -l app=nginx-app
kubectl get po -l app=nginx-app
kubectl describe deploy nginx-deployment
-------------
Update
-------------
kubectl set image deploy nginx-deployment nginx-container=nginx:1.9.1 (update nginx 1.7.9 to 1.9.1)
kubectl edit deploy nginx-deployment (change image version)
kubectl rollout status deploy  nginx-deployment
kubectl get deploy

------------
Rollback
------------
nginx:1.7.9 --> nginx:1.91 (correct 1.9.1)
kubectl set image deploy nginx-deployment nginx-container=nginx:1.91 --record
kubectl rollout status deploy nginx-deployment (check the status)
kubectl rollout history deploy nginx-deployment (print command history)
kubectl rollout undo deploy nginx-deployment (undo the deployment)
kubectl rollout status deploy  nginx-deployment

-----------
Scale up
-----------
kubectl scale deployment nginx-deployment --replicas=5
kubectl get deploy
kubectl get po
------------
Scale down
------------
kubectl scale deployment nginx-deployment --replicas=1
kubectl get deploy
kubectl get po
------------
Delete deployment
-----------
kubectl delete -f nginx-deployment.yml
kubectl get po -l app=nginx-app
*****************************************
Services
*****************************************
Services is a way of grouping of pods that are running on the cluster. services provides important features such as 1. load balancing 2. service discovery betwen Apps 3. support zero downtime app deployment
why we need services:
Is it possible to have permanent IP address?
How do various components connect & communicate?
How do applications are exposed to outside world?

Types of services:
ClusterIP: Exposes the Service on a cluster-internal IP. Choosing this value makes the Service only reachable from within the cluster. This is the default ServiceType.
NodePort: Exposes the Service on each Node's IP at a static port (the NodePort). A ClusterIP Service, to which the NodePort Service routes, is automatically created. You'll be able to contact the NodePort Service, from outside the cluster, by requesting <NodeIP>:<NodePort>.
LoadBalancer: Exposes the Service externally using a cloud provider's load balancer. NodePort and ClusterIP Services, to which the external load balancer routes, are automatically created.
ExternalName: Maps the Service to the contents of the externalName field (e.g. foo.bar.example.com), by returning a CNAME record with its value. No proxying of any kind is set up.

#nodeport.yml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  labels:
    app: nginx-app
spec:
  selector:
    app: nginx-app
  type: NodePort
  ports:
  -nodePort: 31000
   port: 80
   targetPort: 80
kubectl create -f nodeport.yml
kubectl create -f nginx-deployment.yml
kubectl get service -l app=nginx-app
kubectl get po -o wide
kubectl get service
kubectl describe svc my-service
Note: Access app by node ip and node port http://nodeip:nodeport
------------------
Delete
----------------
kubectl delete svc my-service
kubectl get pods
*************************
LoadBalancer service type
*************************
#service-lb.yml
apiVersion: v1
kind: Service
metadata:
  name: my-service2
  labels:
    app: nginx-service
spec:
  selector:
    app: nginx-app
  type: LoadBalancer
  ports:
  - nodePort: 31000
    port: 80
    targetPort: 80
kubectl create -f service-lb.yml
kubectl get svc
kubectl get service -l app=nginx-app
kubectl describe service my-service2
kubectl describe service my-service2 | grep Load



 
    








