## WSO2 APIM Operator for Kubernetes

Deploying WSO2 APIM Patterns in kubernetes through a simple command. Introducing a new Custom Resource Definition called APIManager to efficiently and easily deploy all 4 patterns, and the custom pattern in Kubernetes.

![K8S CRD workflow](https://github.com/wso2-incubator/wso2am-k8s-operator/blob/master/docs/images/crd-overview.png "K8S CRD workflow")

## Quick Start Guide

In this document, we will walk through the following.
* Deploy a default pattern in Kubernetes
* Deploy other patterns
* Deploy Custom pattern
* Expose Service via NodePort (default-LoadBalancer)
* Override Configurations (configmap, pvc, deploy-config)
* Add new Configurations (configmap, pvc)
* Invoke the API

### Installation Prerequisites
* [Golang](https://golang.org/doc/install) v1.12+ 
* [Kubernetes cluster](https://kubernetes.io/docs/setup/) and client v1.12 or above
* [Docker](https://docs.docker.com/install/) & [DockerHub](https://hub.docker.com/) / private docker registry account
* [Helm](http://docs.shippable.com/deploy/tutorial/deploy-to-gcp-gke-helm/)

## Steps:
1. Clone the wso2am-k8s-operator repository

``` 
    git clone https://github.com/wso2-incubator/wso2am-k8s-operator.git 
```

2. Setup the required Storage
```
    helm install stable/nfs-server-provisioner
```
   
<details>
<summary>Advanced</summary>
<br>
<ul><li>GCP Users:</li>
    
#### Running External-nfs

**Prerequisites**
1. NFS Server Ip

2. Create Paths inside the server
```
sudo mkdir -p $HOME/test/wso2-apim/pattern-1/synapse-configs
sudo mkdir -p $HOME/test/wso2-apim/pattern-1/executionplans
sudo mkdir -p $HOME/test/wso2-apim/pattern-1/mysql
```

Before running the controller, do the following steps.
      
1. Create a new file with the name “pv.yaml” and copy the following code and paste it there. 

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wso2am-pattern-1-shared-apim-synapse-configs-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 1Gi
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: “$HOME/test/wso2-apim/pattern-1/synapse-configs”
    server: “enter_your_server_ip”
 
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: wso2am-pattern-1-shared-apim-executionplans-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: “$HOME/test/wso2-apim/pattern-1/executionplans”
    server: “enter_your_server_ip”
 
---
 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wso2apim-with-analytics-mysql-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  Nfs:
    path: “$HOME/test/wso2-apim/pattern-1/mysql”
    server: “enter_your_server_ip”
  
```
then change the server IP and run the following command with the namespace.

```
kubectl create -f pv.yaml -n <USER-NAMESPACE>
```

Create a new Configmap for PVC  using below template with the name of “pvc-conf.yaml” and replace the “nfs” value with the name of your storage class.
```
kind: ConfigMap
apiVersion: v1
metadata:
 name: pvc-config
 namespace: wso2-system
data:
 wso2amP1AmSynapseConfigsPvcName: "wso2am-p1-am-synapse-configs"
 wso2amP1AmExecutionPlansPvcName: "wso2am-p1-am-execution-plans"
 wso2amAmMysqlPvcName: "wso2am-p1-mysql"
 
 wso2amPvcAccessmode: "ReadWriteMany"
 
 wso2amPvcSynapseConfigsStorage: "1Gi"
 wso2amPvcExecutionPlansStorage: "1Gi"
 wso2amPvcMysqlStorage: "20Gi"
 
 # for internal-nfs-server-provisioner
 storageClassName: "nfs"
 
```

<li>Minikube Users:</li>
    _HostPath setup_
 </ul>
</details>

  
    
3. Apply the command to create the controller-artifacts (in wso2-system namespace)

``` 
    kubectl apply -f artifacts/install/controller-artifacts/ 

    Output: 

    namespace/wso2-system created
    serviceaccount/wso2am-pattern-1-svc-account created
    clusterrole.rbac.authorization.k8s.io/wso2am-controller-role created
    clusterrolebinding.rbac.authorization.k8s.io/wso2am-controller-role-binding created
    customresourcedefinition.apiextensions.k8s.io/apimanagers.apim.wso2.com created
    deployment.apps/wso2am-controller created

```
4. Apply the command below to create controller-configs (in wso2-system namespace)
```
    kubectl apply -f artifacts/install/controller-configs/
    
    Output:
    
    configmap/controller-config created
    configmap/pvc-config created
```

5. Apply the command below to create pattern-spceific api manager artifacts
```
    kubectl apply -f artifacts/install/api-manager-artifacts/pattern-1/
    
    Output:
    
    configmap/wso2am-p1-analytics-dash-bin created
    configmap/wso2am-p1-analytics-dash-conf created
    configmap/wso2am-p1-analytics-worker-conf created
    configmap/wso2am-p1-apim-1-conf created
    configmap/wso2am-p1-apim-2-conf created
    configmap/wso2am-p1-mysql-dbscripts created

```

6. Now Let's deploy WSO2 API Manager based on Scenarios. The default and the simplest one is Scenario-1.

```
    kubectl apply -f scenarios/scenario-1/wso2-apim.yaml 

    Output:

    apimanager.apim.wso2.com/cluster-1 created

```

7. Relavant artifact's pods based on specified pattern will be up and running. Check them through below command.
```
    kubectl get pods
    
    Output:
    NAME                                                       READY   STATUS    RESTARTS   AGE
    mysql-cluster-1-787bbf7d7f-dd97x                           1/1     Running   0          3m35s
    wso2-am-1-cluster-1-7c965f55db-r98zh                       1/1     Running   0          3m35s
    wso2-am-2-cluster-1-7dd4669dd-tzl6q                        1/1     Running   0          3m35s
    wso2-am-analytics-dashboard-cluster-1-7654b7b7b7-nnfwc     1/1     Running   0          3m35s
    wso2-am-analytics-worker-cluster-1-67b995dd64-4d4tz        1/1     Running   0          3m35s
```
8. Also you can view the running services through this command.
```
kubectl get svc

NAME                                    TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                                                                     AGE
analytics-dash-svc                      LoadBalancer   10.43.246.200   34.93.74.215     32201:31562/TCP                                                                             118m
apim-1-svc                              LoadBalancer   10.43.245.163   35.244.26.60     8280:32339/TCP,8243:32247/TCP,9763:30327/TCP,9443:31757/TCP                                 118m
apim-2-svc                              LoadBalancer   10.43.244.31    34.93.171.163    8280:32289/TCP,8243:31366/TCP,9763:30954/TCP,9443:31909/TCP                                 118m
wso2apim-analytics-service              LoadBalancer   10.43.252.140   35.200.217.231   7612:30414/TCP,7712:32469/TCP,9444:32169/TCP,9091:30755/TCP,7071:30125/TCP,7444:31236/TCP   118m
wso2apim-with-analytics-rdbms-service   ClusterIP      10.43.242.130   <none>           3306/TCP                                                                                    118m

```
9. To access the portals, get the EXTERNAL-IP of any of the 2 apim services (apim-1-svc /apim-2-svc) and add it to your /etc/hosts file of your machine.
```
/etc/hosts
----------
35.244.26.60       wso2apim

```

10. Finally you can successfully access the following portals.
   
   _APIM Publisher_ - https://wso2apim:9443/publisher
   
   _APIM Devportal_ - https://wso2apim:9443/devportal


  
