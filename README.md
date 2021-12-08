# Monitor easytravel on kubernetes with DynaKube

## Install easytravel on k3s rancher
Rollout the easytravel application on bare metal VM (VM on a cloud provider) with k3s and istio gateway.  
(tested with Azure VM Standard B2s - 2 vCPU, 4 GB)  

    #install k3s
    cd ~
    curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.19 K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="--disable=traefik" sh -s -

    #install istio (if you use the shell in the box as putty access on tcp 443, you will lost your access and need to have a direct access with putty on tcp 22)
    echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> .profile; source ~/.profile
    curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.9.1 sh -
    sudo mv istio-1.9.1/bin/istioctl /usr/local/bin/istioctl
    istioctl install -y
    kubectl label namespace default istio-injection=enabled

    #easytravel
    kubectl create -f https://raw.githubusercontent.com/dynatrace-ace-services/easy-dynakube-deployment/main/manifest-easytravel/easytravel.yaml

    #waiting for easytravel_www pod report READY before installing istio gateway > 3 minutes
    while [[ `kubectl get pods -n easytravel | grep easytravel-www | grep "0/"` ]];do kubectl get pods -n easytravel;echo "==> waiting for easytravel_www pod ready";sleep 1; done
    kubectl apply -f https://raw.githubusercontent.com/dynatrace-ace-services/easy-dynakube-deployment/main/manifest-easytravel/istio-easytravel.yaml


## Deploy Dynakube

From Dynatrace > Deploy Dynatrace > Start Installation > Kubernetes
Generate DynaKube script installation 

    export TENANT=<YYYY>.live.dynatrace.com
    export API_TOKEN=<API_TOKEN>
    export PAAS_TOKEN=<PAAS_TOKEN>
    
    wget https://github.com/dynatrace/dynatrace-operator/releases/latest/download/install.sh -O install.sh && sh ./install.sh --api-url "https://$TENANT/api" --api-token $API_TOKEN --paas-token $PAAS_TOKEN --skip-ssl-verification --cluster-name "k3s"


## Kubernetes Monitoring for K3S

Why I don't have workload with k3s ?  
it's because with "K3s" Kubernetes master is running at https://127.0.0.1:6443 ... and can't be changed on k3s   
You can verify it with this command    
    
    kubectl cluster-info 

This very light k3s cluster is not completely supported and you need to do some adjustements.    
As a workaround, you have to manually configured it here `Settings > Cloud and virtualization > Kubernetes`. 

   1) Delete "k3s" on `https://127.0.0.1:6443`
   2) Create new one an ActiveGate  (if the activegate is not yet here, wait 1 or 2 minutes...)   
        Connection name : `k3s`   
        Kubernetes API URL Target : `https://<your-domaine-name>:6443/`   
        Kubernetes Bearer Token : => run the command below  
        Verify hostname in certificate against Kubernetes API URL : `disable`  
        Monitor event : `enable`  
        Opt in to the Kubernetes events feature for analysis and alerting : `enable`  
        Include all events relevant for Davis : `enable`  
        
       *command for the bearer token* :  
       
    kubectl get secret $(kubectl get sa dynatrace-kubernetes-monitoring -o jsonpath='{.secrets[0].name}' -n dynatrace) -o jsonpath='{.data.token}' -n dynatrace | base64 --decode;echo
    
   3) upload the 2 specific dashboards from [here](/dashboard-monitoring-k3s)  

Now you have your workload with k3s :)

## Monitor Logs

   enable the log on the cluster : Settings > Log Monitoring > Log sources and storage 

## Monitor Istio
   
   (Under construction - only heathz/ready with Opentracing)  
   enable Envoy on the monitorid technology : Settings > Monitoring > Monitored Technologies > Envoy

## Restart Services 

    kubectl delete --all pods -n easytravel
    kubectl delete --all pods -n istio-system

## Prometheus integration

  1) Example of prometheus metric : Kube state metric (KSM)   
 
kube-state-metrics is a simple service that listens to the Kubernetes API server and generates metrics about the state of the objects.    
It is not focused on the health of the individual Kubernetes components, but rather on the health of the various objects inside, such as deployments, nodes and pods.   
More details (here)[https://github.com/kubernetes/kube-state-metrics]  
Install KSM in your k3s cluster with this command :   

    kubectl create -f https://raw.githubusercontent.com/dynatrace-ace-services/easy-dynakube-deployment/main/manifest-kube-state-metrics/kube-state-metrics.yaml  

Now, we have some metrics with the prometheus format generated by the KSM service from the Kubernetes API.    
No need to install the prometheus solution to collect these metrics, just use Dynatrace to collect all the prometheus metrics :)  

  2) Enable metrics scraping for Dynatrace  

The process to enable metric scraping for Dynatrace is described [here](https://www.dynatrace.com/support/help/shortlink/monitor-prometheus-metrics#enable-metrics-scraping-required)  
According to this doc we have modified the manifest for KSM **Deployment** with these information   

    kubecetl apply -f https://raw.githubusercontent.com/dynatrace-ace-services/easy-dynakube-deployment/main/manifest-kube-state-metrics/kube-state-metrics-with-dynatrace-integration.yaml  

![image](https://user-images.githubusercontent.com/40337213/145271037-41097192-6143-47f7-a8d7-43fcef53488b.png)  


   Return to Dynatrace  `Settings > Cloud and virtualization > Kubernetes`. 
        Monitor annotated Prometheus exporter : `enable`  
        
   On the metric explorer seacrh 'kube_'
   ![image](https://user-images.githubusercontent.com/40337213/145270856-e741523b-fc47-430d-b257-526648052241.png)  


## Usefull command
Verify istio:

    istioctl analyze
    
Show all namesace

    kubectl get ns
    
Delete All pods (to restart the service)

    kubectl delete --all pods -n easytravel
    kubectl delete --all pods -n system-istio
    
Get pod installed by DynaKube

    kubectl get pods -n dynatrace

Get pod installed by easytravel

    kubectl get pods -n easytravel

Get pod installed by istio

    kubectl get pods -n istio-system

Get all pods system

    kubectl get all -n kube-system
    
Stop loadgen : 

    kubectl -n easytravel scale --replicas=0 deployment/loadgen
 
Restart k3s server:

    sudo systemctl restart k3s
    
Uninstall k3s :

    /usr/local/bin/k3s-uninstall.sh
