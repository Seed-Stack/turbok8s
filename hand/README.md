Deploy turbok8s step-by-step by hand! Remember, turbok8s is nothing special, merely an opinionated method of giving you "everything you'll probably want". As such, it's only fitting that there are clear, step-by-step instructions for deploying things by hand without using any turbok8s tooling. We also hope by following this install method, you'll find that Kubernetes isn't that scary after all!

Sections are in order of how they must be completed for things to work properly.

# Operating Systems
This guide has been tested from the following systems:
- Manjaro Linux

# Basics
You'll need some basics for things like grabbing yaml from remote sources and parsing/editing yaml/json easily. Make sure your system has:
- curl
- jq
- grep
- kubectl (obviously :D )
- A cluster in which to ctl your kube

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
    kubectl get crd | grep "operators\.coreos"

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
    kubectl get deploy -n olm

    # should give you something like
    NAME               READY   UP-TO-DATE   AVAILABLE   AGE
    catalog-operator   1/1     1            1           9m59s
    olm-operator       1/1     1            1           9m59s
    packageserver      2/2     2            2           9m29s
    ```

# Argo CD