---
layout: post
title:  "Building your own private cloud with Raspberry Pi 4 and Kubernetes"
author: sandro
categories: [ Raspberry Pi ]
image: assets/images/raspberry4-cluster.jpg
tags: [sticky, featured]
---
If you want to learn something about **Raspberry Pi, Ubuntu, Docker, Kubernetes** and get a taste of **ARM64**, then building your own Raspberry Pi Kubernetes Cluster is o good starting point. Or at least, I learned a lot about these technologies when building this. However I was sitting in a lock-down and had plenty of time to spend. So if you want to speed up a little bit and skip the hours debugging and fiddling around Github issue pages, then I recommend you to follow my guide here. At the time writing this article it is probabbly the most condensed knowledge to have a propper mini clud. Or at least I would have considered myself lucky to find an article like that. But maybe I was not looking hard enough ;-).

### What we will build
Our goal is to build a proper mini cloud. So that means is, you should be able to run various applications (servers) running behind a single or multiple URLs having proper load balancing (and autoscaling) and all of this cramped up in 10x8x6cm of space in your living room. 

More percislely we will build a Kubernetes cluster with at least two (actually you would need three to have proper load balancing) Raspberry Pi 4s and a external hard drive for persistance storage.

We are using the following components:

- **Docker** for running our containers
- **Kubernetes** as our container orchestrator
- **MetalLB** as load balancer to claim IP addresses from our router
- **Nginx** as ingress controller to have multiple PODs sitting behind the same IP for convenience
- An **NFS** server for persistance storage
- **Grafana** to get some health metrics about our cluster

### What hardware you will need
1. At least two (actually three, or in my case four) Raspberry Pi 4
2. An external SSD hard drive
3. A switch or a router to plugin your RPIs
4. Some basic understanding about engineering and this article

### Basic installation

#### Make your Raspberries ready
If you bought a Raspberry Pi with a preinstalled Rasbian OS we first want to get rid of this an install Ubuntu 20.04. Reason is, I am not quite an expert with Raspbian and Ubuntu gives us plenty of useful features with some minor drawbacks in terms of performance. 

##### Flash the RPIs
So plug your Raspberry SD card to your Macbook or PC and do the following:
1. Download the [Raspberry Imager](https://www.raspberrypi.org/downloads/) for your desired OS
2. Open the Imager and click 'Choose OS' 
3. Take Ubuntu 20.04
4. Choose your SD card
5. Write OS on your SD card and relax

*Do these steps for all RPIs SD cards

This can take a couple of minutes, read on and be prepared. 

Afther the process is done, plug the SD cards back to the RPIs and plug them to your router or switch and give them some power. 

##### Start with installation
Afterwards login to your RPI (find the IP address in your router):
```
 ssh ubuntu:ubuntu@<yourIPaddress>
```
It will probabbly ask you to change the password, so be a PRO and do that. 

In our Kuberenetes cluster you will have one master and several worker nodes. For simplicity we will call the master 'master' and the worker nodes 'nodes'. So in my case I have one master and three worker nodes. 

The next steps is the basic setup and we will do that on ALL RPIs. 
#### Basic setup
Rename the nodes accorind to your naming concept:
Add your deisred name e.g. 'k8s-master' or 'k8s-worker-1' to: 
```
sudo nano /etc/hosts
sudo nano /etc/hostname
sudo hostnamectl set-hostname <hostname>
```
Now you should be able to resolve these guys with their names instead of IP addresses. 

Let's do some updates first
```
sudo apt update
sudo apt-get install wget
```
Change timezone (yes this is important)
```
sudo timedatectl set-timezone Europe/Zurich # change to your timezone if different
```

Now we are going to install Docker on a specific version, targeted for ARM64.
```
wget https://download.docker.com/linux/ubuntu/dists/bionic/pool/stable/arm64/docker-ce_18.06.1~ce~3-0~ubuntu_arm64.deb
sudo dpkg -i docker-ce_18.06.1~ce~3-0~ubuntu_arm64.deb 
```
Now we gonna start the docker service and twerk with some permissions.
```
sudo systemctl start docker
sudo systemctl enable docker

sudo groupadd docker
sudo usermod -aG docker $USER
```
Now let's install kubernetes
```
sudo apt install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt install -y kubeadm kubelet kubectl kubernetes-cni
```
Since we are dealing with a cluster (shared resources) we have to make some clever memory choices and turn off the SWAP.

```
sudo swapoff -a
sudo nano /etc/fstab # —> add '#' for all 'UUID=' entries or delete them
```
Now we have to add some lines to the cmdline.txt
```
sudo nano /boot/firmware/cmdline.txt
# add at line end (NOT ON NEW LINE): cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```
And now let's do the obvious:
```
sudo reboot
```

If you executed these steps just for one RPI do this now for all the others. As mentioned, this is our basic installation or prerequisite for all other steps. 

### Master
The next steps we just execute on your master. So do a quick login with 'ssh user@k8s-master' and proceed.

#### Init Kubernetes
Hey, we are going to build a kubernetes cluster, so at one point we have to initialize kubernetes on your master. For this we are using 'kubeadm init' and we choose a network range of 10.244.0.0/16. 
>If you don't like that range, feel free to change, but then let me know in the comments sections why in gods name you don't like that!

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
**IMPORTANT** After the init command you get a nice 'kubeadm join' command. We need this command to join our nodes to the cluster. So note this thing down. This thing is valid for 24h. So if you ever need to join a node after this time, create a new join command with this: kubeadm token create --print-join-command

### Node
Let's join or nodes to the cluster with the previous command:
```
kubeadm join <MASTERIP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash <HASH>
```
Congratulations, you now have a cluster. Close the connection to your nodes, you won't need them anymore and you will thank me later by not executing the next few steps on the nodes! 

### Back on Master
**HEY! We are back on the master, so don't execute the following steps on the nodes!**

#### Install some Fannel
SSH back on the master, if you did not do that already, because we are going to install our POD network and add some fannel:
###### (I think it is about time to add a giphy to get you thorugh this wall-of-text)
![fannel-guy]({{ site.baseurl }}/assets/images/fannel.gif)

So execute these commands:
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

#### Load Balancer & Ingress
If you are using cloud provider like GKS or AKS, you can usually create and ingress of type LoadBalancer and they will create a LB in the background for you. Since we are hosting everything on our own we have to find our own solution for that. I was using MetalLB that surves that purpose for us. You can just create a ingress with type LoadBalancer and MetalLB will expose an IP address from a predefined range. You can then forward your traffic from your router to this IP and happy days.

##### Install MetalLB
To install MetalLB use the following command:
```
kubectl apply -f https://gist.githubusercontent.com/saendu/d2f0a33bd73f86684f86099390fce67a/raw/e531759ef8ed89943824f1aa4185f8a068f6aead/metallb.yaml
```
Now we are going to define a configmap for our MetalLB controller. Change the IP range if you don't like it. 
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.200-192.168.1.220
```
No let's apply it:
```
kubectl apply -f ./metallb-config.yaml
```

##### Nginx Ingress Controller
Load balancers are nice, but creating a seperate load balancer for each ingress is for sure not a nice design. Also we might want to route our traffic accoring some hostnames or path in the URL. For that we are using a Nginx controller.
We are creating is with helm for our convenience and save some yaml files. At that point you probabbly don't have helm installed, so let's do that first:
```
wget https://get.helm.sh/helm-v3.3.0-rc.1-linux-arm64.tar.gz
tar xvzf helm-v3.3.0-rc.1-linux-arm64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
```
If your console outputs the helm version then you are good to go (on). Else go here and help yourself: [Helm Install Guide](https://helm.sh/docs/intro/install/).
Before we can use helm we have to add and update some helm repos. 
```
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
helm repo update
```
Now we can finally install the ingress controller:
```
helm install nginx-ingress stable/nginx-ingress --set rbac.create=true --set controller.service.type=LoadBalancer
```
Since this helm chart however has some evil amd64 images included, we have to change our created nginx deployments to use a arm image.
```
kubectl set image deployment/nginx-ingress-controller nginx-ingress-controller=quay.io/kubernetes-ingress-controller/nginx-ingress-controller-arm:0.30.0
kubectl set image deployment/nginx-ingress-default-backend nginx-ingress-default-backend=k8s.gcr.io/defaultbackend-arm:1.5
```
Now when we check our pods, we should get two running ones:
```
nginx-ingress-controller-796cf8f9b8-kw4mw        1/1     Running   0          24h
nginx-ingress-default-backend-8546479f64-nn845   1/1     Running   0          24h
```

### Cert Manager
Not having HTTPS/SSL is an absolute nogo. Mainly of security but also because it is nowadays so easy (and free) to get an SSL cert thanks to Let's Encrypt. It comes with one downside, your SSL certs can very quickly expire. To get rid of that we are using a cert manager (with certbot). Also checkout my [article]({{ site.baseurl }}/letsencrypt/) I wrote about this. 

##### Install CRUDs
First we gonna install some CRUDs.
```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager.crds.yaml
```

##### Install Cert Manager
Since we have helm now, we can use that to install the cert manager.
```
kubectl create namespace cert-manager 
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v0.15.1
```

#### Cluster Issuer
And to conveniently use the cert manager we are going to create a clusterissuer (for all namespaces) to generate the certs for us. 

```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: default
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: sandrek@gmail.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    # An empty 'selector' means that this solver matches all domains
    - selector: {}
      http01:
        ingress:
          class: nginx
```
Now you can conveniently just add the following lines to your ingress to make cert bot work for you:
```
...
annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
...
```

### Persistance Volume
Like databases, don't you? Well who doesn't. A little bit of persistance storage never hurt anyone. To make this work we need our external SSD hard drive and use our master to serve a NFS server. 

#### Install NFS Server
So plug in your SSD hard drive and let's strart!

##### On master
First we need some NFS packages. 
```
sudo apt-get install nfs-common nfs-kernel-server
```
Now we check the path of our external hard drive (partition):
```
sudo fdisk -l
```
Note down the Device path, in my case it was /dev/sda2
Now we have to get the PARTUUID of our partition:
```
sudo blkid /dev/sda2
```
In my case the UUID was the following:
6bd5ab90-ec2c-4833-93bc-6619b4875316
Now we crate a directory were we host our peristance volume claims:
```
sudo mkdir -p /srv/nfs
sudo chmod -R 777 /srv/nfs
```
Next step is to make sure that we automatically mount this partition on startup. 

> **IMPORTANT**: I removed this afterwards because my /srv/nfs was always automatically mounted without these entries in fstab. And I noticed an issue after reboot with my nfs-service not being able to start (check with systemctl status nfs-server.service), when having this entry in fastab. So if you have the same problem, maybe remove these lines again if you see a /srv/nfs path being mounted after restart. (Sorry for this unclear description but I don't know more ATM)

But it would go like this:
```
sudo nano /etc/fstab 
# add this to the next line
UUID="6bd5ab90-ec2c-4833-93bc-6619b4875316"    /srv/nfs    auto    nosuid,nodev,nofail,noatime 0 0
```
Now we gonna startup some services:
```
sudo systemctl enable rpcbind.service
sudo systemctl enable nfs-server.service
sudo systemctl start rpcbind.service
sudo systemctl start nfs-server.service
```
Check the status:
```
systemctl status rpcbind.service
systemctl status nfs-server.service
```

Now we are going to export our shared path:
```
sudo nano /etc/exports
/srv/nfs    192.168.1.0/24(rw,no_root_squash)
# restart the service
sudo systemctl restart nfs-server.service
```

Now you should be all done and be able to use the NFS server in your kubernetes cluster as your storage provisioner. 

##### On worker nodes
> Ok sorry I told you we won't execute anything on the worker nodes anymore. I was lying about this, but I just wanted to make sure that you are not executing all these commands in the wrong terminal and regret it afterwards.

Our worker nodes need to read and write the files via NFS. So they need the 'nfs' command in order to connect to our master serving this. 
```
sudo apt-get install -y nfs-common
```
You can now close your worker terminals again. ;-) 
##### Back on the master
Yeah we are back guys, back on the master with the next commands!

#### Persistance Volume on Kubernetes
To use our above installed NFS server we need to install some roles and give them access (RBAC).
```
kubectl -f https://gist.githubusercontent.com/saendu/8a5f84508b36753d14663bf6f5d71b7c/raw/37f165a5e4b5e26827b6bf88ce3697e2e25e9b8d/nfs-rbac.yaml
```
Afterwards we can finally deploy our NFS client!
<script src="https://gist.github.com/saendu/1e52b7b8e0a1205f45c91cf624e069a4.js"></script>
```
kubectl -f nfs-deployment-arm.yaml
```
Change to your environment values
<script src="https://gist.github.com/saendu/1e5395b8298572fb39a5adcb4aacb764.js"></script>
```
kubectl -f nfs-class.yaml
```

#### Test our persistance volume

##### Create PVC
<script src="https://gist.github.com/saendu/df2f21581f6ea75fe7898c567a9ac175.js"></script>
```
kubectl -f https://gist.githubusercontent.com/saendu/df2f21581f6ea75fe7898c567a9ac175/raw/a73a7a158f378924236ac45751c87200c11885b7/nfs-test-pvc.yaml
```

##### Create POD with PVC
<script src="https://gist.github.com/saendu/148e5875cf06e4e1fcf70fb483842450.js"></script>
```
kubectl -f https://gist.githubusercontent.com/saendu/148e5875cf06e4e1fcf70fb483842450/raw/0a7986a1bd6da75ddfcf8d13bc3330d638225fb4/nfs-test-pod.yaml
```

To check if everything works navigate to your NFS path and check if you have a SUCCESS file.
In my case this looked like this (your GUID will be different!):
```
cd /srv/nfs/default-test-claim-pvc-dac79781-ae45-489a-aa71-96191b700f63/
ls
```
If you can see a SUCCESS file, then this was a success! 

Now we are able to create simple PersistanceVolumeClaim and just make sure that we are adding the line:
```
annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
```

### Dashboard
And for the fans of GUIs like this you can install your very own Kubernetes dashboard:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
kubectl create clusterrolebinding dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa
kubectl proxy --address='0.0.0.0' --accept-hosts='^*$'
http://k8s-master:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
```
### Monitoring
Last but not least we also need something to monitor our cluster. Else it would just be some nerd project that a guy posted in the internet. Adding monitoring to our setup, makes us look a little bit like PROs. 
