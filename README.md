# Deploy kubernetes via ansible (on cloudstack servers) with logging (efk) & monitoring (prometheus) support #

## What you will get:
- 1 master node running : k8s for container orchestration
- 2(or more) slave nodes : running the actual containers (workers)
- Elk: we will send all k8s container logs to an elasticsearch DB, via fluentd. 
- Visualize the logs with kibana and an example dashboard
- k8s dashboard addon (not elk dashboard), where you can visualize k8s component in a GUI. 

*Prerequisit:*
- Cloudstack cloud provider (ex: exoscale) | but you can deploy anywhere else with a bit of adaptation in ansible 
- An ubuntu VM where you will run ansible recipes, and manage k8s with kubeclt

# 1. Deploy kubernetes

### 1.1 Clone repo

    git clone https://github.com/gregbkr/k8s-ansible-elk.git k8s && cd k8s

### 1.2 Deploy k8s on Cloudstack infra

I will use the setup made by Seb: https://www.exoscale.ch/syslog/2016/05/09/kubernetes-ansible/
I just added few lines in file: ansible/roles/k8s/templates/k8s-node.j2 to be able to get log with fluentd
(# In order to have logs in /var/log/containers to be pickup by fluentd
    Environment="RKT_OPTS=--volume dns,kind=host,source=/etc/resolv.conf --mount volume=dns,target=/etc/resolv.conf --volume var-log,kind=host,source=/var/log --mount volume=var-log,target=/var/log" )

    cd ansible
    nano k8s.yml      <-- edit k8s version, num_node, ssh_key if you want to use your own

Run recipe

	ansible-playbook k8s.yml
	watch kubectl get node    <-- wait for the nodes to be up

### 1.3 install kubectl locally (with same version as server)

    curl -O https://storage.googleapis.com/kubernetes-release/release/v1.4.6/bin/linux/amd64/kubectl
    chmod +x kubectl
    mv kubectl /usr/local/bin/kubectl

### 1.4 Checks:
	
	kubectl get all --all-namespaces    <-- should have no errors here
	
# 2. Deploy elk v5 to collect k8s logs

### 2.1 Pre-requisit (not needed anymore)

Fix present in roles/k8s/templates/k8s-node.j2 

if needed, manually on all node: fix an issue with hungry es v5

    ssh -i ~/.ssh/id_rsa_foobar core@185.19.29.212
    sudo sysctl -w vm.max_map_count=262144

make it persistent:

    sudo vi /etc/sysctl.d/elasticsearch.conf
    vm.max_map_count=262144
    sudo sysctl --system
	
### 2.2 run elasticsearch, kibana, fluentd

    cd .. 
    kubectl apply -f logging.yaml

    kubeclt get all       <-- if you see es container restarting, please restart all nodes one time only (setting vm.max_map_count)

### 2.3 LoadBalancer: access kibana from public IP

Create load-balancer https://github.com/kubernetes/contrib/tree/master/service-loadbalancer
Give 1 or more nodes the loadbalancer role (so you can balance with public DNS later)

    kubectl label node 185.19.30.121 role=loadbalancer
    kubectl apply -f service-loadbalancer.yaml

### 2.4 Access kibana
loadbalancer_node_ip:5601

### 2.5 See logs in kibana

Check logs coming in kibana, you just need to refresh, select Time-field name : @timestamps + create
Load and view the dashboard: management > Saved Object > Import > dashboard/elk-v1.json

# 3. Troubleshooting
    

If needed Check services with an ubuntu container



If issue connecting to svc (for example elasticsearch), use ubuntu container: 
- First see if ubuntu will be in the same namespace as the service you want to check:

```
nano utils/ubuntu.yaml
kubectl apply -f utils/ubuntu.yaml
```

- Depending in which namespace ubuntu runs, you can check services with one of these commands:

```
kubectl exec ubuntu -- curl elasticseach:9200   <-- should returns ... "cluster_name" : "elasticsearch"...
kubectl exec ubuntu -- curl kibana:5601         <-- should returns ... var defaultRoute = '/app/kibana'...
    
kubectl exec ubuntu -- curl elasticsearch.logging.svc.cluster.local:9200         <-- ubuntu in default namespace
kubectl exec ubuntu --namespace=logging -- nslookup elasticsearch               <-- ubuntu in logging namespace
kubectl exec ubuntu --namespace=logging -- nslookup kubernetes.default.svc.cluster.local     <-- ubuntu in logging namespace
```

- Check port 9200 on the node running elasticsearch container: ssh -i ~/.ssh/id_rsa_foobar core@185.19.29.212 netstat -plunt
- Uncomment type: NodePort and nodePort: 39200 if you want to access elasticsearch from any node_ip
- Check data in elasticsearch
    kubectl exec ubuntu -- curl es:9200/_search?q=*
    curl node_ip:39200/_search?q=*       <-- if type: NodePort set in es.yaml

No logs coming in kibana:
- check that there are file in node: ssh -i ~/.ssh/id_rsa_foobar core@185.19.29.212 ls /var/log/containers

Can't connect to k8s-dashboard addon:
- Carefull, addon are in kube-system namespace!
If stuck use type: NodePort and
- Find the node public port: kubectl describe service kubernetes-dashboard --namespace=kube-system
- Access it from nodes : http://185.19.30.220:31224/

DNS resolution not working? Svc kube-dns.kube-system should take care of the resolution

    kubectl exec ubuntu -- nslookup google.com
    kubectl exec ubuntu -- nslookup kubernetes
    kubectl exec ubuntu -- nslookup kubernetes.default
    kubectl exec ubuntu -- nslookup kubernetes-dashboard.kube-system

Pod can't get created? See more logs:

    kubectl describe po/es
    kubectl logs -f es-ret5zg

# 4. Annexes

### 4.1 Shell Alias for K8s
```
alias k='kubectl'
alias kk='kubectl get all'
alias wk='watch kubectl get all'
alias ka='kubectl get all --all-namespaces'
alias kc='kubectl create -f'
alias kdel='kubectl delete -f'
alias kcdel='kubectl delete configmap'
alias kd='kubectl describe'
alias kl='kubectl logs'

```

### 4.2 Need another slave node?
Edit ansible-cloudstack/k8s.yml and run again the deploy

### 4.3 Want to start from scrash? Delete the corresponding namespace:

    kubectl delete namespace monitoring
    kubectl delete namespace logging

### 4.4 Deploy Kubenetes dashboard addon (not elk)
kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
Check (carefull dashboard is running in namespace=kube-system )

    kubectl get all --all-namespaces

Loadbalancer should already be forwarding public query to service dashboard. However you will probably have to restart lb pod or recreate it
Access it: loadbalancer_node_ip:8888 

