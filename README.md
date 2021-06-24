# JupyterHub on k8s
Prerequisite: min 2 Ubuntu VMs (1 head node, 1 worker node). Head node must be externally accessible preferrably via port 80 which is where we will expose the JupyterHub service. Nodes can have different base OS, but bash scripts will need to be tweaked.

## Create a k8s cluster
Run Head node script, this will install any prerequisite packages, Docker, Kubernetes 1.20.0, Helm, NFS.  
`bash runOnHeadNode.sh`   
Initialise the cluster, set up networking via Flannel by default, and enable bash autocompletion for kubectl. Calico can be used for networking if you do not require MetalLB/if your cloud provider can provide a load balancer. Here Flannel is used due to [current issues with Calico + MetalLB](https://metallb.universe.tf/configuration/calico/).  
`bash setupCluster.sh`   
Run worker node script on each worker node, this will install any prerequisite packages, Docker, Kubernetes 1.18.1, NFS.  
`bash runOnWorkerNode.sh`   
Run the kubeadm join command on each node, this can be found in the output of the cluster initialisation either on your terminal or will be saved in clusterclusterInit.out.  
From this point, you could jump straight to setting up the [JupyterHub service](https://github.com/rohinijoshi06/jupyterhub-on-k8s/blob/master/README.md#jupyterhub) and use the fallback options in the jhub config file. 
Optional: Import your cluster into Rancher if you are using it to centrally manage your clusters, (ssh -L 8443:localhost:443 -Nfl ubuntu 130.246.212.36)
## Monitoring the cluster 

This is not essential to the functionality of JupyterHub, but a nice to have.

1. Create a monitoring namespace 
```
$ kubectl create namespace monitoring
```
2. Deploy the prometheus stack (prometheus, grafana, node exporters, and CRDs) https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack 
```
$ helm install prometheus-stack -n monitoring prometheus-community/kube-prometheus-stack -f prometheus/prometheus-values.yaml
```
3. Set up an ssh tunnel (if needed) to view the grafana dashboard by looking at the port number for the grafana service with something like
4. Login to Grafana with admin/prom-operator (edit yaml to change this)
```
$ ssh -L 30290:localhost:30290 -Nfl rjoshi [external facing IP of the head node]
```

## Deploy NFS for JupyterHub storage (hub + users)
Add Helm stable repo to access NFS provisioner helm chart 
```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update 
```

Create k8s namespace nfsprovisioner and deploy NFS via helm chart with the following command. This will also set the default storage class to be NFS.  
`helm install nfs stable/nfs-server-provisioner --namespace nfsprovisioner --set=storageClass.defaultClass=true`

## Add a PostGresDB for the hub service
This is technically optional.
Add helm charts, create kubernetes namespace, and install a PostGresDB via helm chart with
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm upgrade --install pgdatabase --namespace pgdatabase bitnami/postgresql \
--set postgresqlDatabase=jhubdb \
--set postgresqlPassword=postgres
```
## JupyterHub
See Helm chart to JupyterHub App version mapping here: https://jupyterhub.github.io/helm-chart/#jupyterhub


The JupyterHub config file can be used to customise your deployment by overriding defaults in the base helm chart.
A sample jupyterhub config file is jupyterhub-config.yaml and it can be tweaked as per your requirements.
Create proxy secret token and hub's cookie secret with `openssl rand -hex 32`

As before we will create a kubernetes namespace for the application ie jhub, add the helm repo, and install Jhub helm chart
```
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update
helm upgrade --install jhub jupyterhub/jupyterhub --namespace jhub --version=0.11.0 --values jupyterhub-config.yaml
```
## Load Balancer for bare metal cluster
Here we use [Metallb 0.9.3](https://metallb.universe.tf/). This is required for bare metal clusters or if your cloud provider doesnt provide a load balanacer for k8s clusters.  
Prerequisite: enable strict ARP mode by editing the kube-proxy configmap, search for IPVS, and set strictARP to true. 
`kubectl edit configmap -n kube-system kube-proxy`   
Apply namespace yaml. 
`kubectl apply -f metallb/namespace.yaml`  
Apply metallb yaml. 
`kubectl apply -f metallb/metallb.yaml`  
Create secret on first install only    
`kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"`  
MetalLB will be idle until you apply the config map that lists the addresses/address pools available. (edit this based on the IPs you have available!).  
`kubectl apply -f metallb/metal_config.yaml` 

## NGINX reverse proxy on the head node
Create logs dir, copy in the config file and reload NGINX   
```
sudo mkdir /root/logs
sudo cp nginx/default /etc/nginx/sites-available/default
sudo nginx -t
sudo systemctl reload nginx
``` 

You should be able to view the JupyterHub service at `http://<IP>/hub/login/` 
