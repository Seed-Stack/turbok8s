# turbok8s
turbok8s (*"turbo k8s"* or *"turbo kube"*) enhances a kubernetes cluster with bolt-on additions to make it "ready to run" your app. Like a turbo on your car, it uses the existing engine (kubernetes) to provide enhanced functionality and performance (the turbo [and other things]). This is essentially a kubernetes cluster, with a logical subset of the CNCF projects "ready to run" your app in a production-like manner. turbok8s gives you **"everything you'll probably want"** to run your application on kubernetes!

The input to turbok8s is a cluster. The output of turbok8s is a turbo-charged cluster with all the useful stuff ready to go! turbok8s is cluster and cloud-provider agnostic, provided the minimum hardware requirements are met.

# Philisophy and Goals
The purpose behind turbok8s is to provide an easy way run your application in kubernetes, with perfect parity across all of your environments, while also removing the chore of installing the same services over and over. If you are deploying an application to Kubernetes, the mission of turbok8s is to give you all the tools you need to do that with minimal configuration, and be up in running in minutes, while also being prod-ready and ready to scale. turbok8s in its default form gives you "everything you'll probably want" to run your applicaiton.

While cloud providers may have load balancers to use, none of them work the same. While local development tools like minikube and microk8s may have addons you can install, these don't work the same and often aren't avaliable anywhere else. Relying on even a single tool like this makes your tech stack unportable, locking you into a specific vendor or toolset.
 
turbok8s gives you the ability to run anywhere, on anything, on your own terms. It's worth noting that turbok8s is transparent - You can use it but you could also get to the same point yourself installing the things that turbok8s installs. turbok8s is merely an opinionated method of installing "everything you'll probably want", perhaps with some utilities thrown in. **The only platform you should be locked into is Kubernetes!**

## Goals
Constraints and criteria for turbok8s: 
- Runs the same locally or in the cloud, for parity across environments (`dev`, `staging`, `prod`)
- Optimized for minimal cost, reduce and/or eliminate monthly variable costs.
- Lightweight. Run on your raspberry pi
- "bolt-on" design. Add to any cluster
- Grab only the components you want, add other components at any time
- Leverage operators wherever possible for a self-managing experience
- Do not use any proprietary solitions (such as cloud load balancers). Be portable to any cluster
- Ready to scale. Work in single-master single-node environments, as well as multi-node HA environments, as well as be able to upgrade from the former to the latter

# A Note on Cost
Assuming you run your cluster on some type of public cloud or Virtual Server Provider (VPS), turbok8s aims to give you predictiable, fixed monthly costs, which are especially crucial for startups, small teams, and solo devs. Variable monthly costs are one of the primary reasons we believe so strongly in self-hosting wherever possible on k8s! turbok8s strives to give you a fixed monthly operational cost, which only increases when you are actually ready to scale, while still remaining fixed and your desired scale.

## Case Study
We use vultr for the development of turbok8s. We aren't endorsed by them but we do prefer using VPS over public cloud. The current form of turbok8s runs on the smallest-possible vultr cluster: a 2vCPU 2GRAM single node, which is managed by vultr's kubernetes control plane (think GKE or AKS). In it's original form, this cost $20/mo because turbok8s also used a load balancer provider by vultr: $10/mo for the node, $10/mo for the LB. 

This turned out to be a great driver and motivator for the philisophy of turbok8s: we could do better! This is what led us to purchasing a public IP for $3/mo and managing it with metalLB instead. The entire setup now only costs $13/mo! This is obviously a minimal form, being single-node, however due to using vultr's managed control plane this is an HA setup and can absolutely scale when the need arises.

The case study and the development of turbok8s itself demonstrate several things:
- **Hard numbers showing that managing something yourself with kubernetes can be cheaper** We saved $7/mo by managing out own LB. While this is a small number, it accounts for 35% of the cluster's cost. Now just imagine extrapolating that across an org that's spending tens of thousands of dollars in the cloud each month
- **The ability to eliminate variable monthly cost**. The entire setup costs exactly $13/mo. No more, no less


# Hardware Requirements
This has not been extensively tested yet to find out the limits, but everything runs just fine on a 2CPU 4G RAM system. You can probably get away with less as well. 

# External Inputs and Extra-Cluster Requirements and Assumptions
We are strong believers that you should strive to self-host just about everything. Soapbox can be found [here](https://isaacgentz.com/tech/everything-on-kubernetes) and [here](https://isaacgentz.com/tech/cloud-spaghetti). That said, there are some things where it does make sense to swallow your pride and pay, for the cost of doing it yourself will likely far outweight the benefit. turbok8s aims to give you as many tools as possible to self-host, while assuming that some basic things do indeed exist outside of the cluster. 

- Internet Connection
  - Have to reach your services somehow! 
  - In all seriousness, turbok8s will probably work just fine for air-gapped environments (for the services that make sense of course) but this hasn't been attempted/tested
- DNS provider
  - turbok8s assumes that you have a DNS provider somewhere that you can create A-records and CNAME's on to point to your cluster. There is a minimal amount of configuration, so click-ops is fine
  - Probably doesn't pan out to [become your own registrar](https://www.icann.org/resources/pages/registrar-fees-2018-08-10-en)
- The cluster itself
  - BYOC (Bring Your Own Cluster)
- Git repo
  - turbok8s assumes that you have a place to store your configuration somewhere external to the cluster
- Public IP address
  - If you're using metalLB, turbok8s assumes you bring your own IP addresses

# Opinionated
turbok8s is not perfect. Tools included here are not any better or worse than alternative tools. They are merely the ones we find ourselves reaching for the most often, and what we believe give you the simplest, most effective setup for success.

# Acknowledgement
turbok8s would not be possible without all of the wonderful tools upon which it relies!

# Running
Let's get this thing deployed!
## local dev
"local dev" refers to specifically running on your own machine. 

### Setup
There are many great tools for running locally, such as microk8s, minikube, k3s, etc. Currently we prefer using minikube, but anything works.

1. Make a cluster
  1. `minikube start`
    1. Note that the default options are fine here, because it creates a container with 2vCPU and 4G RAM, which is typically the smallest cloud instance you can create, and the default recommended size
1. For each service, `kubectl apply` or other similar instruction (see below)

### services
Because most personal machines aren't directly exposed to the internet, there are only a small subset of useful services that are currently spun up locally, however the ultimate goal is to be able to run everthing locally so you can mirror production.

#### tekton
See [tekton](#tekton) for how to install (one-liner)

## prod-like cluster
A prod-like cluster is one that is accessible in some manner over the public internet, probably running in a public cloud, VPS, or data center spomewhere.

### services
All services can be deployed 


# Service List
This is the list of services turbok8s plans to support. A checkbox indicates this is working in some form. This is a living list, and "everything you'll probably want" may change in the future!

- [ ] StackGres
  - Horizontally-scaling Postgresql DB
- [x] MetalLB
  - Manage both public and private IP addresses
  - Works with the nginx-ingress controller to allow virtual hosting of infinite domains behind a single public IP address
- [x] nginx-ingress
  - Manage routing to services
  - Set up for virtual hosting
- [x] cert-manager
  - Automatic TLS cert management
- ArgoCD
  - GitOps CD
- [X] Tekton
  - Creation of push-based CI/CD pipelines and task runner
  - [ ] Kaniko
    - Automatic image building
- [ ] Harbor
  - Artifact and image repository
- [ ] Rook
  - Object, block, and file storage
- Signoz
  - [ ] Logs, metrics, alerting
- [X] Kubernetes Dashboard
  - See the state of the cluster in a single pane of glass
- [ ] Hashicorp Vault
  - De-facto standard for secrets management
- [ ] Zero Trust Solution
  - TBD, access private services in the cluster without jumping through hoops

## k8s cluster
Managed by Pulumi on Vultr - use whatever you see fit for your own clusters!

## Stackgres
`kubectl apply -f stackgres-operator.yml`

## MetalLB
Software load balancing so we can do virtual hosting of a theoritically infinite amount of sites. This load-balances a single public IP, which `nginx-ingress` then uses to route traffic based on the actual domain and route

## nginx-ingress
Ingress controller. Examines URL's for domain and path. Works with metal-lb, and in this configuration allows us to virtual host as much as we want behind a single public IP

## cert-manager
Installed statically with `kubectl apply ...` per [instructions](https://cert-manager.io/docs/installation/), uses `cluster-issuer-acme.yaml` to create a ClusterIssuer, which will reach out to [Let'sEncrypt](https://letsencrypt.org/) to sign Certificate Requests.

## ArgoCD
Argo is built on the [gitops engine](https://github.com/argoproj/gitops-engine).

## tekton
CI tool. 
Installed as an [operator](https://tekton.dev/docs/operator/), one-liner:

```
kubectl apply -f https://storage.googleapis.com/tekton-releases/operator/previous/v0.68.1/release.yaml
```

### kaniko
This is not an operator itself, a tool that can be used in the cluster for building images. Use tekton pipelines to run kaniko

## Harbor
Artifact registry. This is where we will be able to store images. Major perk of storing images in-cluster is they'll always be cached, no network latency for pushing or pulling. Also authentication is simpler. Kaniko can push here

## rook
This will deploy/manage Ceph (block storage) and MinIO (object storage).

MinIO is required for Stackgres to work so we can create an S3-compatible object store in the form of a PV for a StackGres Postgres cluster instance to consume

## kubernetes-dashboard
Deployed with manifest one-liner:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```
Also wrote a small `serviceaccount` with `clusterrolebinding` to `cluster-admin` for some quick n' dirty access, deployed from `k8s-dashboard-sa-and-binding.yaml`

## hashicorp Vault
Study showing that Vault is the way to go: https://github.com/derailed pretty much what everyone's using. However the awesome thing is that they've released an [operator](https://developer.hashicorp.com/vault/tutorials/kubernetes/vault-secrets-operator) for it now, which simply interfaces with normal k8s secrets, so it's designed to be "install and use" and way less operational overhead for hosting. 

# Vendors
Organizations certified to help you with your turbok8s installation, and Kubernetes in general.

> [GK Consulting](https://gkconsulting.dev/#ready-to-discuss-further)

Love Kubernetes? Love turbok8s? Become a SeedStack Certified Vendor!