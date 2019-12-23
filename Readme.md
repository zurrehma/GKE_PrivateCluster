# GKE_PrivateCluster
This repo contains three configuration files.
* GKE_template.yml (responsible for creation of private cluster inside gcp)
* deployment.yml (responsib for creation of [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/),
[service](https://kubernetes.io/docs/concepts/services-networking/service/) and  [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/))
* ingress-resource.yaml (responsible for creation of [ingress-controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/])
## prerequisites
* GCP account
* GCP project
* Docker environment
* git
## Creation of a GKE private cluster 
* connect to gcp project
   **gcloud config set project [ProjectID]**
* For creating kubernetes cluster,As cluster is private, replace your machine IP address in GKE_template.yml in following line (- cidrBlock: YourIP/32), This allows to expose master end-point to your machine.
   **gcloud deployment-manager deployments create GKE-deployment --config GKE_template.yaml**
* Connect to cluster
   **gcloud container clusters get-credentials [Cluster Name] --zone [Zone] --project [ProjectID]**
* Create deployments,services and HPA 
   **kubectl apply -f deployment.yml**
* For installation of helm (inorder to have ingress controller we have to install [helm](https://hackernoon.com/what-is-helm-and-why-you-should-love-it-74bf3d0aafc))
   **curl -o get_helm.sh https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get
   chmod +x get_helm.sh
   ./get_helm.sh**
* Installing Tiller with RBAC enabled
   **kubectl create serviceaccount --namespace kube-system tiller
   kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
   helm init --service-account tiller**

* Download and install helm-chart(helm-chart contains configurations for installing nginx-ingress controller)
In this case cluster is private, so when we install helm-chart from offical reposiotry it try to download docker image.
 Therfore, we have to either enable NAT or change the helm-chart configurations.
In our case we are changing helm-chart configurations,so we can use same docker image from gcr registry which is accessable by cluster
   **docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1
   docker tag quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1 gcr.io/[ProjectID]/nginx-ingress:latest
   docker push gcr.io/[ProjectID]/nginx-ingress**

## Download Helm Chart
   **mkdir helmchart && cd helmchart
   helm fetch stable/nginx-ingress
   tar xvzf [.tgz file]
   cd nginx-ingress/
   vi values.yaml**
Replace docker image in values.yaml with the gcr pushed image
   **helm install --name nginx-ingress .**
For deploying ingress controller to cluster
   **kubectl apply -f ingress-resource.yaml**


## Checking HPA functionality
HPA take decisions on the base of metrics server, therfore install metrics server
**git clone https://github.com/kubernetes-incubator/metrics-server.git
 cd metrics-server/
 vi deploy/1.8+/metrics-server-deployment.yaml**
Add following lines after image tag
**command:
    - /metrics-server
    - --metric-resolution=30s
    - --kubelet-insecure-tls
    - --kubelet-preferred-address-types=InternalIP**
Now install the metrics server.
 **kubectl apply -f deploy/1.8+/**
Check initial state of HPA, note number of replicas in output
 **kubectl get hpa**
 **kubectl run -it deployment-for-testing --image=busybox /bin/sh**
Inside deployment-for-testing container enter
 **wget -q -O- http://ExternalIP/spring** 
For getting external IP of load balancer enter command **kubectl get services**
If entering http://ExternalIP/spring in browser and in container outputs same, this means everything is working fine
Now put some stress on pod by entering following loop inside container
**echo "while true; do wget -q -O- http://ExternalIP/spring; done" > loops.sh
chmod +x /loops.sh
sh /loops.sh**

After some time, again enter following command to see changes in replicas
 **kubectl get hpa**

## Delete Cluster
gcloud deployment-manager deployments delete [Project Name]
