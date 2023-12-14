Deploy turbok8s step-by-step by hand! Remember, turbok8s is nothing special, merely an opinionated method of giving you "everything you'll probably want". As such, it's only fitting that there are clear, step-by-step instructions for deploying things by hand without using any turbok8s tooling. We also hope by following this install method, you'll find that Kubernetes isn't that scary after all!

You may wish to go through this guide for learning purposes, or for customizing your own solution. This guide is intentionally pedantic  to assist with comprehension as well as help show the development process itself.

Sections are in order of how they must be completed for things to work properly in turbok8s' default configuraiton.

# Operating Systems
This guide has been tested from the following systems:
- Manjaro Linux

# Basics
You'll need some basics for things like grabbing yaml from remote sources and parsing/editing yaml/json easily. Make sure your system has:
- curl
- jq
- grep
- k (obviously :D )
- A cluster in which to ctl your kube
- bash, zsh, or some other similar shell

# Assumptions
- This guide assumes you're in the `hand` directory
- `k` is used as an alias for `kubectl`

# Cluster Setup
This is just for setting up a basic cluster for testing, because otherwise Turbok8s assumes you already have a cluster you'd like to install into!

## Local
If you want to use MetalLB see [Local Cluster Networking Setup](./README.md#local-1)
If you're running locally and **don't plan on using MetalLB**, start `minikube` like so:
```
minikube start --memory='4G' --cpus='2'
```

## Prod-like / Cloud / Datacenter
Start however you like, just ensure you have a least 1 node with 4G RAM and 2 vCPU.

# Cluster Networking
This step is only required if setting up MetalLB. Otherwise skip to [OLM setup](./README.md#olm).

To use MetalLB we must make pools of "*real*" IP addresses avaliable for load balancing within the cluster. This step may look a little different depending on if you're running locally or in the cloud/datacenter, because you have complete control over your nodes locally (e.g. with `minikube`) whereas in the cloud/datacenter you may provision IP's/subnets in a declaratice manner and have them "appear" in your cluster, with all of the networking infrastructure managed for you.



## Local
What you'll end up with is really cool! You'll end up with "private" services, which only devices on your LAN can reach, and also "public" services, which are reachable from the internet!

Locally, this looks like creating an `IPPool` of private IP addresses that exist in your LAN, as well as finding your public IP on which to serve traffic coming in from the internet. 

Start `minikube` with a dedicated network interface, otherwise the default docker-bridge (or something similar for your CRI) will be used. We'll use Calico here:
```
minikube start --network-plugin=cni --memory='4G' --cpus='2'
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/calico.yaml
```

> Need conntrack to start with no driver so services are avaliable on the host

> Note we're [not using an operator here](../README.md#calico)

Because you're running locally, we need to find out some information about your LAN. Find out your LAN subnet:
```
ip a

# should see something like
2: wlp0s20f3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether fc:b3:bc:92:f4:e6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.86.107/24 brd 192.168.86.255 scope global dynamic noprefixroute wlp0s20f3
       valid_lft 76825sec preferred_lft 76825sec
    inet6 fe80::80a4:a14:918d:4b3c/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```
In the above example, you can see the IP adddress of the WiFi card in the computer used for this demo: `192.168.86.107` (`/24`) is the subnet. This gives a LAN subnet of `192.168.86.0/24`.

Now run a scan of your LAN to find a slice of it that isn't being used:
```
nmap -sP <your home LAN e.g. 192.168.86.0/24>
```
Based on the results of this, find any small slice of IP's (even just a single one is fine!) that are unused. For example, here is a scan of a home LAN:
```
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-11 06:00 MST
Nmap scan report for 192.168.86.1
Host is up (0.016s latency).
Nmap scan report for 192.168.86.24
Host is up (0.014s latency).
Nmap scan report for 192.168.86.99
Host is up (0.019s latency).
Nmap scan report for brainbox.lan (192.168.86.107)
Host is up (0.000033s latency).
Nmap scan report for tstat-29b664.lan (192.168.86.140)
Host is up (0.020s latency).
Nmap scan report for canona3a68f.lan (192.168.86.149)
Host is up (0.059s latency).
Nmap scan report for galaxy-s22-ultra.lan (192.168.86.151)
Host is up (0.0058s latency).
Nmap done: 256 IP addresses (7 hosts up) scanned in 2.44 seconds
```
We have many free IP addresses. Let's grab a small range of `192.168.86.192/28` to use for our private services. We could also use even just a single IP, such as `192.168.86.192/32` (`/32` to represent a range of 1 IP). Whatever you choose, add this range to `cni/calico-network-local.yaml` in the "private" section and also to `metallb/metal-lb-local.yaml`.

> Although not necessary (especially for a quick demo) It's recommended to configure your router reserve this IP range so it is not handed out by DHCP.

For the public IP, you can go to a site like [ip chicken](https://www.ipchicken.com/) to find your home's public IP. Add this in to the "public" section of `cni/calic-network-local.yaml` and `metallb/metal-lb-local.yaml`

Now create some `IPPools` that will make IP's avaliable in the cluster for use. Again this is Calico-specific but you should be able to use any CNI:

```
k apply -f cni/calico-network-local.yaml
```

We'll `apply` the metalLB configuration in a later step.



## Prod-Like / Cloud / Datacenter
You probably already have your cluster deployed with some sort of CNI. Make sure that you have a pool of private addresses your nodes are able to see, and (assuming you wish for your cluster to be reachable over the public internet), at least one public IP address also visible to your nodes. Turbok8s assumes Calico for your CNI, but you can adopt whichever CNI you prefer, the only thing that matters is that the IP addresses are visible within the cluster so that MetalLB can manage them.


# OLM
The [Operator Lifecycle Manager](https://olm.operatorframework.io/) is a wonderful project that deploys and manages k8s operators, which are our preferred way of running applications in k8s. turbok8s attempts to install as much as possible through this tool.

1. Grab the latest version of the OLM from their [github releases page](https://github.com/operator-framework/operator-lifecycle-manager/releases), which typically also shows the installation instructions. Here's the install instructions for reference:
    ```
    OLM_VERSION=v0.26.0
    curl -L https://github.com/operator-framework/operator-lifecycle-manager/releases/download/$OLM_VERSION/install.sh -o install.sh
    chmod +x install.sh
    ./install.sh $OLM_VERSION
    ```
2. Verify the OLM installed correctly
  1. You should see an `olm` namespace
  2. You should see an `operators` namespace
  3. A bunch of CRD's should have been installed
      ```
      k get crd | grep "operators\.coreos"

      # should give you something like
      catalogsources.operators.coreos.com        
      clusterserviceversions.operators.coreos.com
      installplans.operators.coreos.com          
      olmconfigs.operators.coreos.com            
      operatorconditions.operators.coreos.com    
      operatorgroups.operators.coreos.com        
      operators.operators.coreos.com             
      subscriptions.operators.coreos.com         
      ```
  4. Finally, to functionally confirm everything is set up you should see several `deployments`.
      ```
      k get deploy -n olm

      # should give you something like
      NAME               READY   UP-TO-DATE   AVAILABLE   AGE
      catalog-operator   1/1     1            1           9m59s
      olm-operator       1/1     1            1           9m59s
      packageserver      2/2     2            2           9m29s
      ```
3. You can see a bunch of operators that are avaliable out of the box:
    ```
    k get packagemanifests.packages.operators.coreos.com
    ```

# MetalLB
MetalLB allows your cluster to perform load balancing itself! One of the most expensive aspects of cloud hosting is paying for load balancers, so doing this in-cluster is a great way to go.

## OperatorHub (alpha)
[OLM Reference](https://operatorhub.io/operator/metallb-operator)

1. Create the subscription - note the use of `create` and not `apply`:
    ```
    k create -f metallb/metallb-operator-subscription.yaml
    ```
2. Verify the operator is up - specifically make sure that `PHASE` is `Succeeded`, not `Installing` or anything else. You can `watch` with `-w`
    ```
    k get clusterserviceversions -n operators

    # should look like this
    NAME                        DISPLAY            VERSION   REPLACES                   PHASE
    metallb-operator.v0.13.11   MetalLB Operator   0.13.11   metallb-operator.v0.13.3   Succeeded

3. Create a MetalLB resource
    ```
    k apply -f metallb/metallb.yaml
    ```
  - This will launch the "functional" parts of MetalLB, you should see the `speaker` and `controller` running
    ```
    k get deploy -n operators --field-selector metadata.name=controller
    
    NAME         READY   UP-TO-DATE   AVAILABLE   AGE
    controller   1/1     1            1           9m37s

    k get daemonsets.apps -n metallb-system 

    NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
    speaker   1         1         1       1            1           kubernetes.io/os=linux   10m
    ```

## Configuration
We need to give MetalLB some IP's to manage! Generally-speaking, and how turbok8s behaves in its default configuration, you'll want a *pool* of private IP's, and at least one *public* IP (but you could have a pool if you want). The private IP pool corresponds to a range of IP's in your private network you want to load balance on, whereas the [public ip](../README.md#external-inputs-and-extra-cluster-requirements-and-assumptions) would typically be something you pay for to serve as the "public entrypoint" into your cluster. 

We already have our IP pools ready to go from the [Cluster Networking](./README.md#cluster-networking) step above. Apply this config:
```
k apply -f metallb/metallb-local.yaml
```

Validate that Calico, your Container Networking Interface(CNI) shows the address pools that MetalLB (your load balancer) will be using:
```
# Calcio IP Pools
k get ippools.crd.projectcalico.org
NAME                  AGE
default-ipv4-ippool   14m
private-ips           3m47s <-- THIS ONE
public-entrypoint     3m47s <-- AND THIS ONE

# MetalLB IP Pools
k get ipaddresspools.metallb.io -n metallb-system
NAME                AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES
private-ips         true          false             ["192.168.86.200/32"]
public-entrypoint   true          false             ["134.195.227.140/32"]

```

If you `describe` each of the `ippools` you should see the CIDR's matching that of MetalLB

# ingress-nginx
Now that everything's configured at Layer 4 (L4 - the IP pools we set up) it's time to manage things at Layer 7 (L7) and route HTTP requests! We'll use [ingress-nginx](https://github.com/kubernetes/ingress-nginx) as our ingress controller to do this.

Unfortunately operator install via OLM is currently only supported for OpenShift clusters, so we'll [install with helm](https://github.com/nginxinc/nginx-ingress-helm-operator/blob/main/docs/manual-installation.md).

```
CURRENT=$(pwd);
mkdir /tmp/ingress-nginx;
cd /tmp/ingress-nginx/;
git clone https://github.com/nginxinc/nginx-ingress-helm-operator/ --branch v2.0.2;
cd nginx-ingress-helm-operator;
make deploy IMG=nginx/nginx-ingress-operator:2.0.2;
cd $CURRENT
```

Verify that the nginx-ingress operator is up:
```
k get all -n nginx-ingress-operator-system
```

## Configuration
We'll now create two services, one for traffic coming in on our public IP and one for traffic coming in on our private IP range.

Create a namespace:
```
k apply -f ingress/namespace.yaml
```

And create the services:
```
k apply -f ingress/nginx-ingress-lb.yaml
```

# Argo CD
[OperatorHub reference](https://operatorhub.io/operator/argocd-operator)

1. Create a `subscription` to the ArgoCD  operator for the OLM (note the use of **create** here and not **apply**)
    ```
    k create -f argocd/argocd-operator-subscription.yaml
    ```
3. Verify the operator is up - specifically make sure that `PHASE` is `Succeeded`, not `Installing` or anything else. You can `watch` with `-w`
    ```
    k get clusterserviceversions -n operators

    # should look like this
    NAME                     DISPLAY   VERSION   REPLACES                 PHASE
    argocd-operator.v0.8.0   Argo CD   0.8.0     argocd-operator.v0.7.0   Succeeded
    ```
4. Create an Argo CD cluster
    ```
    k apply -f argocd/namespace.yaml
    k apply -f argocd/argocd-cluster.yaml
    ```
5. Verify Argo CD is now deployed. You should see a bunch of resources now in the `argocd` namespace
    ```
    k get all -n argocd

    # should look something like
    NAME                                      READY   STATUS    RESTARTS   AGE
    pod/argocd-application-controller-0       1/1     Running   0          3m17s
    pod/argocd-redis-b8b598b87-wsn4w          1/1     Running   0          3m17s
    pod/argocd-repo-server-6bdf966d5b-5w9b9   1/1     Running   0          3m17s
    pod/argocd-server-6b69d87bcc-pqfmx        1/1     Running   0          3m17s

    NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
    service/argocd-metrics          ClusterIP   10.102.88.134    <none>        8082/TCP            3m17s
    service/argocd-redis            ClusterIP   10.99.226.100    <none>        6379/TCP            3m17s
    service/argocd-repo-server      ClusterIP   10.104.254.145   <none>        8081/TCP,8084/TCP   3m17s
    service/argocd-server           ClusterIP   10.103.7.249     <none>        80/TCP,443/TCP      3m17s
    service/argocd-server-metrics   ClusterIP   10.110.143.50    <none>        8083/TCP            3m17s

    NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/argocd-redis         1/1     1            1           3m17s
    deployment.apps/argocd-repo-server   1/1     1            1           3m17s
    deployment.apps/argocd-server        1/1     1            1           3m17s

    NAME                                            DESIRED   CURRENT   READY   AGE
    replicaset.apps/argocd-redis-b8b598b87          1         1         1       3m17s
    replicaset.apps/argocd-repo-server-6bdf966d5b   1         1         1       3m17s
    replicaset.apps/argocd-server-6b69d87bcc        1         1         1       3m17s

    NAME                                             READY   AGE
    statefulset.apps/argocd-application-controller   1/1     3m17s
    ```
6. Functionally ensure that Argo CD is up by going to the dashboard (we'll properly expose this in a subsequent step)
    ```
    k port-forward -n argocd services/argocd-server 8080:80
    curl -Lk localhost:8080
    ```
  - this will give you a message about enabling Javascript since we're just doing an HTTP GET with curl
  - you can actually visit the dashboard by going to `https://localhost:8080` in your browser
  - Grab the default `admin` password
      ```
      k get -ojson secret -n argocd argocd-cluster | jq -r '.data."admin.password"' | base64 --decode
      ```
    - You can use this to log in from your browser for the `admin` user
    - NOTE: If the output ends in a `%`, *omit* the `%`, as this is simply indicating that there's no `newline` in the output, not a character that's actually part of the password

# cert-manager
[Operatorhub Reference](https://operatorhub.io/operator/cert-manager)
1. Create a subscription:
  ```
  k apply -f cert-manager/cert-manager-operator-subscription.yaml
  ```
2. You should now see a `cert-manager` operator
  ```
  k get operators

  # something like
  NAME                        AGE
  argocd-operator.operators   29m
  cert-manager.operators      99s
  ```