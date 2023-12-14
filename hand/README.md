Deploy turbok8s step-by-step by hand! Remember, turbok8s is nothing special, merely an opinionated method of giving you "everything you'll probably want". As such, it's only fitting that there are clear, step-by-step instructions for deploying things by hand without using any turbok8s tooling. We also hope by following this install method, you'll find that Kubernetes isn't that scary after all!

You may wish to go through this guide for learning purposes, or for customizing your own solution.

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

# Argo CD
[OperatorHub reference](https://operatorhub.io/operator/argocd-operator)

1. Create a `subscription` to the ArgoCD  operator for the OLM
  - You can grab the subscription manifest from here:
      ```
      curl https://operatorhub.io/install/argocd-operator.yaml > argocd/argocd-operator-subscription.yaml
      ```
  - Or use the manifest already at `argocd/argocd-operator-subscription.yaml`
2. Create the subscription
    ```
    k apply -f argocd/argocd-operator-subscription.yaml
    ```
3. Verify the operator is up
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