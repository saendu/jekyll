---
layout: post
title:  "Building your own private Cloud with Raspberry Pi 4 and Kubernetes"
author: sandro
categories: [ Projects ]
image: assets/images/rpi4-cluster-deck.jpg
tags: [sticky, featured]
---
If you also having some Raspberry Pi 4 lying around, and not really sure what you wanna do with them, then I might have an idea that you could like. Build you own private cloud! And learn something about **Ubuntu, Docker, Kubernetes** and get a taste of **ARM64**. Or at least, I've learned a lot about these technologies when I was building this. However, I was sitting in a lock-down and had plenty of time to spend. So, if you want to speed up a little bit and skip the hours debugging and fiddling around Github issue pages, then I recommend you follow my guide and enjoy. 

I tried to put everything you'll need into this article that it will serve as a ‘goto reference’ for the future.  


### What we will build
Our goal is to build a proper mini cloud. So that means you should be able to run various applications behind a single IP address and having proper load balancing and all of this cramped up in 10x8x6cm of space in your living room. 

More precisely we will build a Kubernetes cluster with at least two (actually you would need three to have proper load balancing) Raspberry Pi 4s (RPIs4) and an external hard drive for persistence storage.

We are using the following components:

- **Docker** for running our containers
- **Kubernetes** as our container orchestrator
- **MetalLB** as load balancer to claim IP addresses from our router
- **Nginx** as ingress controller to have multiple PODs sitting behind the same IP for convenience
- An **NFS** server for persistent storage
- **Grafana** to get some health metrics about our cluster

> It will help if you have a basic understanding of those technologies. If not, I think you should still be able to follow and learn on the way along. 

### What hardware you will need
1. At least two (actually three, or in my case four) RPIs4
2. An external SSD hard drive
3. A switch or a router to plug in your RPIs4
4. Some basic understanding about engineering and this article

### Basic Raspberry Pi installation
Our RPIs4 need a basic setup before we can start with the fun. For completeness sake, I added these setup instructions to this guide as well and not just be lazy and write 'setup Ubuntu 20.04', 'install docker', etc. Because with these little ARM64 guys some commands might be a bit different. 

#### Make your Raspberries ready
If you bought a Raspberry Pi with a preinstalled Raspbian OS we first want to get rid of this an install Ubuntu 20.04. Reason is, I am not quite an expert with Raspbian, and Ubuntu gives us plenty of useful features with some minor drawbacks in terms of performance. 

##### Flash the RPIs4
So, plug your Raspberry SD card to your MacBook or PC and do the following:
1. Download the [Raspberry Imager](https://www.raspberrypi.org/downloads/) for your desired OS
2. Open the Imager and click 'Choose OS' 
3. Take Ubuntu 20.04
4. Choose your SD card
5. Write OS on your SD card and relax

*Do these steps for all RPIs4 SD cards

This can take a couple of minutes, read on and be prepared. 

After the process is done, plug the SD cards back to the RPIs4 and plug them to your router or switch and give them some power. 

##### Start with installation
Afterwards login to your RPI (find the IP address in your router).
It will probably ask you to change the password, so be a PRO and do that. 

In our Kubernetes cluster you will have one master and several worker nodes. For simplicity we will call the master node just 'master'. So, in my case I have one master and three worker nodes. 

The next steps are the basic setup and we will do that on ALL RPIs4. 

#### Basic setup
Rename the nodes according to your naming concept:
Add your desired name e.g. 'k8s-master' or 'k8s-worker-1' to: 
```
sudo nano /etc/hosts
sudo nano /etc/hostname
sudo hostnamectl set-hostname <hostname>
```
Now you should be able to resolve these guys with their names instead of IP addresses. 

Let's do some updates first
```
sudo apt update
sudo apt-get install -y wget sudo
```
Change timezone (yes, this is important believe me)
```
sudo timedatectl set-timezone Europe/Zurich # change to your time zone if different
```

Now we are going to install Docker on a specific version (18.06.1), targeted for ARM64.
```
wget https://download.docker.com/linux/ubuntu/dists/bionic/pool/stable/arm64/docker-ce_18.06.1~ce~3-0~ubuntu_arm64.deb
sudo dpkg -i docker-ce_18.06.1~ce~3-0~ubuntu_arm64.deb 
```
Now, we gonna start the docker service and twerk with some permissions.
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
# add at line end (NOT ON A NEW LINE!): cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```
And now let's do the obvious:
```
sudo reboot
```

If you executed these steps just for one RPI do this now for all the others. As mentioned, this is our basic setup for all following steps. 

### Master
The next steps we are just executing on your master. So do a quick SSH login to your master.

#### Init Kubernetes
Hey, we are going to build a kubernetes cluster, so at one point we have to initialize kubernetes on our master. For this we are using 'kubeadm init' and we choose a network range of 10.244.0.0/16. 
>If you don't like that range, feel free to change, but then let me know in the comments sections why in God’s name you don't like that!

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
**IMPORTANT** After the init command you get a nice 'kubeadm join' command. We need this command to join our nodes to the cluster. So, note this command down. This thing is valid for 24h. So, if you ever need to join a node after this time, create a new join command with this: kubeadm token create --print-join-command

### Node
SSH to your nodes and join the cluster with the previous command:
```
kubeadm join <MASTERIP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash <HASH>
```
Congratulations, you now have a cluster. Close the connection to your worker nodes, you won't need them anymore and you will thank me later by not executing the next few steps on the nodes! 

### Back on Master
**HEY! We are back on the master, so don't execute the following steps on the worker nodes!**

#### Install some flannel
SSH back on the master, if you did not do that already, because we are going to install our POD network and add some flannel:
##### flannel guy
###### (I think it is about time to add a giphy to get you through this wall-of-text)

<div style="width:100%;height:0;padding-bottom:46%;position:relative;"><iframe src="https://giphy.com/embed/3ohzdIuqJoo8QdKlnW" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>

So execute these commands:
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

#### Load Balancer & Ingress
If you are using a managed Kubernetes cloud provider like GKS or AKS, you can usually create and ingress of type `LoadBalancer` and they will create a load balancer in the background for you. Since we are hosting everything on our own, we have to find our own solution for that. I was using MetalLB that serves that purpose for us. You can just create an ingress with type LoadBalancer and MetalLB will expose an IP address from a predefined range. You can then forward your traffic from your router to this IP and happy days. This is very useful if you want to work with other ports than 80 and 443. 

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
Load balancers are nice but creating a separate load balancer for each ingress is for sure not a nice design. Also, we might want to route our traffic according some hostnames or pathin the URL. For that we are using a Nginx controller.
We are creating this with helm for our convenience and save us time creating some yaml files. At that point you probably don't have helm installed, so let's do that first (in the future you might want to use a more recent helm version):
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
Since this helm chart however has some evil amd64 images included (that won’t work on a RPI4), we have to change our created nginx deployments to use an arm image.
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
Not having HTTPS/SSL is an absolute 'no go'. Mainly of security but also because it is nowadays so easy (and free) to get an SSL cert thanks to Let's Encrypt. It comes with one downside; your SSL certs can very quickly expire (90 days). To get rid of that we are using a cert manager (with certbot). Also check out my [article]({{ site.baseurl }}/letsencrypt/) I wrote about this. 

##### Install CRUDs
First we gonna install some CRUDs.
```
kubectl apply --validate=false -f https://gist.githubusercontent.com/saendu/a9626d540950a1a96c82d81e7a2c0e54/raw/a466d7603c116584e6d3ddbc64e9dbca2c1070e6/cert-manager.crds.yaml
```

##### Install Cert Manager
Since we have helm now, we can use that to install the cert manager.
```
kubectl create namespace cert-manager 
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v0.15.1
```

#### Cluster Issuer
And to conveniently use the cert manager we are going to create a ClusterIssuer (for all namespaces) to generate the certs for us. 

```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  #do not use namespace for a ClusterIssuer  
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: your@email.com
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

### Persistence Volume
Like databases, don't you? Well who doesn't. A little bit of persistence storage never hurt anyone and you might want to run a Wordpress instance on your cluster anyway. To make this work we need an external SSD hard drive and use our master to serve as an NFS server. 

#### Install NFS Server
So, plug in your SSD hard drive and let's start!

##### On master
First, we need some NFS packages. 
```
sudo apt-get install nfs-common nfs-kernel-server
```
Now we check the path of our external hard drive (partition):
```
sudo fdisk -l
```
Note down the Device path, in my case it was /dev/sda2
Now we have to get the PARTUUID or UUID of our partition/disk:
```
sudo blkid /dev/sda2
```
In my case the PARTUUID was the following:
6bd5ab90-ec2c-4833-93bc-6619b4875316
Now we create a directory where we host our persistent volume claims:
```
sudo mkdir -p /srv/nfs
sudo chmod -R 777 /srv/nfs
```
Next step is to make sure that we automatically mount this partition on startup. 

But it would go like this:
```
sudo nano /etc/fstab 
# add this to the next line
PARTUUID=6bd5ab90-ec2c-4833-93bc-6619b4875316    /srv/nfs    auto    nosuid,nodev,nofail,noatime 0 0
```
Now we gonna startup some services:
```
sudo systemctl enable rpcbind.service
sudo systemctl enable nfs-server.service
sudo systemctl start rpcbind.service
sudo systemctl start nfs-server.service
```
Check the status:
```
systemctl status rpcbind.service
systemctl status nfs-server.service
```

Now we are going to export our shared path (you might want to use your IP range):
```
sudo nano /etc/exports
/srv/nfs    192.168.1.0/24(rw,root_squash)
# restart the service
sudo systemctl restart nfs-server.service
```

Now you should be all done and be able to use the NFS server in your kubernetes cluster as your storage provisioner. 

##### On worker nodes
> Ok, sorry I told you we won't execute anything on the worker nodes anymore. I was lying about this, but I just wanted to make sure that you are not executing all these commands in the wrong terminal and regret it afterwards.

Our worker nodes need to read and write the files via NFS. So, they need the 'nfs' command in order to connect to our master serving this. 
```
sudo apt-get install -y nfs-common
```
You can now close your worker terminals again!
 
##### Back on the master
Yeah, we are back guys, back on the master with the next commands!

#### Persistence Volume on Kubernetes
To use our above installed NFS server we need to install some roles and give them access (RBAC).
```
kubectl apply -f https://gist.githubusercontent.com/saendu/8a5f84508b36753d14663bf6f5d71b7c/raw/37f165a5e4b5e26827b6bf88ce3697e2e25e9b8d/nfs-rbac.yaml
```
Afterwards we can finally deploy our NFS client!
Create a new file with 
```
sudo nano nfs-deployment-arm.yaml
```
and edit the `# CHANGE ACCORDING YOUR ENVIRONMENT` pieces if necessary.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner-arm:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-storage # CHANGE ACCORDING YOUR ENVIRONMENT
            - name: NFS_SERVER
              value: 192.168.1.106 # CHANGE ACCORDING YOUR ENVIRONMENT
            - name: NFS_PATH
              value: /srv/nfs # CHANGE ACCORDING YOUR ENVIRONMENT
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.1.106 # CHANGE ACCORDING YOUR ENVIRONMENT
            path: /srv/nfs # CHANGE ACCORDING YOUR ENVIRONMENT
```
Apply the file:
```
kubectl apply -f nfs-deployment-arm.yaml
```
Create a new file with 
```
sudo nano nfs-class.yaml
``` 
and edit provisioner name if necessary.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: nfs-storage # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
```
Apply the file:
```
kubectl apply -f nfs-class.yaml
```
Now we have our persistence volume setup and can create a PVC like so (this is just an example):
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```
The important piece is the annotation `volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"` that we've created previous. 

> NFS is might not the most performant solution to do ReadWriteMany operations. Maybe something like Rook would fit that problem better. Maybe I will come back to that in a later article. 

### Monitoring
Last but not least we also need something to monitor our cluster. Else it would just be some nerd project that a guy posted in the internet. Adding monitoring to our setup, makes us look a little bit more like PROs. 

#### Install 
So, let's do this. First, we have to login to our master and switch our user to root user:
```
sudo su
```
Now we are going to install some prerequisites (to be able to use `make`):
```
apt-get update && apt-get install -y build-essential golang
```
Now clone the project cluster-monitoring project: 
```
git clone https://github.com/saendu/cluster-monitoring.git
```
Now we have to edit the `vars.jsonnet` file and set `k3s.enabled` to `true`, `armExporter.enabled` to `true` and change the `# CHANGE ACCORDING YOUR ENVIRONMENT` pieces below:
```
{
  _config+:: {
    namespace: 'monitoring',
  },
  modules: [
    { 
      name: 'smtpRelay',
      enabled: false,
      file: import 'modules/smtp_relay.jsonnet',
    },
    {
      name: 'armExporter',
      enabled: true,
      file: import 'modules/arm_exporter.jsonnet',
    },
    {
      name: 'upsExporter',
      enabled: false,
      file: import 'modules/ups_exporter.jsonnet',
    },
    {
      name: 'metallbExporter',
      enabled: false,
      file: import 'modules/metallb.jsonnet',
    },
    {
      name: 'traefikExporter',
      enabled: true,
      file: import 'modules/traefik.jsonnet',
    },
    {
      name: 'elasticExporter',
      enabled: false,
      file: import 'modules/elasticsearch_exporter.jsonnet',
    },
  ],

  k3s: {
    enabled: true,
    master_ip: ['192.168.1.106'], // CHANGE ACCORDING YOUR ENVIRONMENT
  },

  suffixDomain: '192.168.1.106.nip.io',  // CHANGE ACCORDING YOUR ENVIRONMENT
  TLSingress: true,
  UseProvidedCerts: false,
  TLSCertificate: importstr 'server.crt',
  TLSKey: importstr 'server.key',
  enablePersistence: {
    prometheus: false,
    grafana: false,
    prometheusPV: '',
    grafanaPV: '',
    storageClass: '',
    prometheusSizePV: '2Gi',
    grafanaSizePV: '20Gi',
  },
  prometheus: {
    retention: '15d',
    scrapeInterval: '30s',
    scrapeTimeout: '30s',
  },
  grafana: {
    from_address: 'your@email.com', // CHANGE ACCORDING YOUR ENVIRONMENT
    //Ex. plugins: ['grafana-piechart-panel', 'grafana-clock-panel'],
    plugins: [],
  },
}
```

> As `suffixDomain` your can use `192-168-1-106-nip.io` (in order to not edit your hosts file) or as in my case I used a public domain because nip.io did not really serve me well. 

We are not done yet, so now the fun begins:
1. Execute the following command in the root dictionary (this can take a while)
```
sudo make vendor && sudo make
```
2. Create the namespace for your monitoring components 
```
kubectl create namespace monitoring
```
3. If you are like me and want the Grafana dashboard exposed to the outside, you have to edit the 
```
sudo manifests/nano ingress-grafana.yaml 
``` 
And add the following lines:
```
...
annotations:
   kubernetes.io/ingress.class: nginx
   cert-manager.io/cluster-issuer: letsencrypt-prod
...
tls:
  - hosts:
    - grafana.yourdomain.ch
    secretName: grafana-yourname-tls
...
```
4. Now we can apply the created yaml files with the following command:
```
make deploy
``` 

You should now be able to navigate to `https://grafana.yourdomain.ch` and login with username: `admin` and password `admin`. Be a PRO again and change that password to a stronger one. 

As you might see, we can't see much on our dashboard. This is because we don't have one. 

#### Create a dashboard
The project from [Carlos Eduardo](https://github.com/carlosedp) has a sweet kubernetes cluster dashboard ready for us to use. So, let's install that thing.

1. Navigate to your dashboard section `https://grafana.sandrofelder.ch/dashboards`
2. Click on `Import` and paste [grafana-dashboard.json](https://gist.githubusercontent.com/saendu/d34b4fc0b07ec6d885a32379d4154790/raw/5d83927a4e658a20586123df55fd151286f29d15/grafana-dashboard.json) to `Import via panel json` 
3. Click `Load`
4. Now, go back to your dashboards click on the newly created dashboard and after some minutes you should see all metrics collected from Prometheus automatically for you

I think this is pretty straight forward way of getting some basic metrics about CPU, Memory, Network and you even have some alerts predefined for you from the alertmanager like:
- CPU temperature
- CPU usage
- Memory usage
- Node down

## That's it folks!
You have your own private cloud sitting in your living room that can basically do everything you would expect from a managed cloud provider:
- Load balancing
- Ingress
- Monitoring

<div style="width:100%;height:0;padding-bottom:65%;position:relative;"><iframe src="https://giphy.com/embed/Um3ljJl8jrnHy" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>

Feel free to hit me with some questions, improvements or thoughts in the comment section below.

Until next time . . .

#### Acknowledgements 
Here some articles that help get through all of this:
- [How to build your own Raspberry Pi Kubernetes Cluster](https://wiki.learnlinux.tv/index.php/How_to_build_your_own_Raspberry_Pi_Kubernetes_Cluster)
- [Building a Pi Kubernetes Cluster – Part 3 – Worker Nodes and MetalLB](https://www.shogan.co.uk/kubernetes/building-a-pi-kubernetes-cluster-part-3-worker-nodes-and-metallb/)
- [Develop and Deploy Kubernetes Applications on a Raspberry Pi Cluster](https://medium.com/better-programming/develop-and-deploy-kubernetes-applications-on-a-raspberry-pi-cluster-fbd4d97a904c)
- [Provision Kubernetes NFS clients on a Raspberry Pi homelab](https://opensource.com/article/20/6/kubernetes-nfs-client-provisioning)
- [Setup a Kubernetes 1.9.0 Raspberry Pi cluster on Raspbian using Kubeadm](https://kubecloud.io/setup-a-kubernetes-1-9-0-raspberry-pi-cluster-on-raspbian-using-kubeadm-f8b3b85bc2d1)
- And many many more...

:heart:
