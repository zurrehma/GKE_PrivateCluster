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
* Connect to gcp project <br/>
   **gcloud config set project [ProjectID]**
* For creating kubernetes cluster,As cluster is private, replace your machine IP address in GKE_template.yml in following line (- cidrBlock: YourIP/32), This allows to expose master end-point to your machine. <br/>
   **gcloud deployment-manager deployments create GKE-deployment --config GKE_template.yaml**
* Connect to cluster <br/>
   **gcloud container clusters get-credentials [Cluster Name] --zone [Zone] --project [ProjectID]**
* Create deployments,services and HPA <br/>
   **kubectl apply -f deployment.yml**
* For installation of helm (inorder to have ingress controller we have to install [helm](https://hackernoon.com/what-is-helm-and-why-you-should-love-it-74bf3d0aafc)) <br/>
   **curl -o get_helm.sh https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get <br/>
   chmod +x get_helm.sh <br/>
   ./get_helm.sh**
* Installing Tiller with RBAC enabled <br/>
   **kubectl create serviceaccount --namespace kube-system tiller <br/>
   kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller <br/>
   helm init --service-account tiller**

* Download and install helm-chart(helm-chart contains configurations for installing nginx-ingress controller) <br/>
In this case cluster is private, so when we install helm-chart from offical reposiotry it try to download docker image. <br/>
 Therfore, we have to either enable NAT or change the helm-chart configurations.<br/>
In our case we are changing helm-chart configurations,so we can use same docker image from gcr registry which is accessable by cluster<br/>
   **docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1 <br/>
   docker tag quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1 gcr.io/[ProjectID]/nginx-ingress:latest <br/>
   docker push gcr.io/[ProjectID]/nginx-ingress**

## Download Helm Chart
   **mkdir helmchart && cd helmchart <br/>
   helm fetch stable/nginx-ingress <br/>
   tar xvzf [.tgz file] <br/>
   cd nginx-ingress/ <br/>
   vi values.yaml <br/>**
Replace docker image in values.yaml with the gcr pushed image <br/>
   **helm install --name nginx-ingress .** <br/>
For deploying ingress controller to cluster <br/>
   **kubectl apply -f ingress-resource.yaml** <br/>


## Checking HPA functionality
HPA take decisions on the base of metrics server, therfore install metrics server <br/>
**git clone https://github.com/kubernetes-incubator/metrics-server.git <br/>
 cd metrics-server/ <br/>
 vi deploy/1.8+/metrics-server-deployment.yaml** <br/>
Add following lines after image tag <br/>
**command: <br/>
    - /metrics-server <br/>
    - --metric-resolution=30s <br/>
    - --kubelet-insecure-tls <br/>
    - --kubelet-preferred-address-types=InternalIP** <br/>
Now install the metrics server. <br/>
 **kubectl apply -f deploy/1.8+/** <br/>
Check initial state of HPA, note number of replicas in output <br/>
 **kubectl get hpa** <br/>
 **kubectl run -it deployment-for-testing --image=busybox /bin/sh** <br/>
Inside deployment-for-testing container enter <br/>
 **wget -q -O- http://ExternalIP/spring**  <br/>
For getting external IP of load balancer enter command **kubectl get services** <br/>
If entering http://ExternalIP/spring in browser and in container outputs same, this means everything is working fine
Now put some stress on pod by entering following loop inside container <br/>
**echo "while true; do wget -q -O- http://ExternalIP/spring; done" > loops.sh <br/>
chmod +x /loops.sh <br/>
sh /loops.sh** <br/>

After some time, again enter following command to see changes in replicas <br/>
 **kubectl get hpa**

## Delete Cluster
gcloud deployment-manager deployments delete [Project Name]
