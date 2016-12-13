Prerequisit:
- Cloudstack cloud provider (ex: exoscale)
- An ubuntu VM where you will run ansible recipes, and manage k8s with kubeclt

# Clone

    git clone https://github/gregbkr/k8s-ansible-elk k8s && cd k8s

# Deploy k8s on Cloudstack infra

I will use the setup made by Seb: https://www.exoscale.ch/syslog/2016/05/09/kubernetes-ansible/
I just added few lines in file: roles/k8s/templates/k8s-node.j2 to be able to get log with fluentd

    # In order to have logs in /var/log/containers to be pickup by fluentd
    Environment="RKT_OPTS=--volume dns,kind=host,source=/etc/resolv.conf --mount volume=dns,target=/etc/resolv.conf --volume var-log,kind=host,source=/var/log --mount volume=var-log,target=/var/log"

    cd ansible-cloudstack
    nano k8s.yml   <-- edit k8s version, num_node, ssh_key if you want to use your own

Run recipe

	ansible-playbook k8s.yml
	watch kubectl get node    <-- wait for the nodes to be up


# install kubectl locally (with same version as server)

    curl -O https://storage.googleapis.com/kubernetes-release/release/v1.4.6/bin/linux/amd64/kubectl
    chmod +x kubectl
    mv kubectl /usr/local/bin/kubectl

# Checks:
	
	kubectl get all --all-namespaces    <-- should have no errors here
	

# Deploy elk v5 to get k8s logs

# Pre-requisit 
On all node: Fix an issue with hungry es v5
    ssh -i ~/.ssh/id_rsa_foobar core@185.19.29.212
    sudo sysctl -w vm.max_map_count=262144
make it persistent:
	sudo vi /etc/sysctl.d/elasticsearch.conf
    vm.max_map_count=262144
    sudo sysctl --system
	
# run elasticsearch, kibana

    cd .. 
    kubectl create -f es.yaml
    kubectl create -f kibana.yaml

# check services with ubuntu temp container

    kubectl create -f utils/ubuntu.yaml
	kubeclt get all
	kubectl exec ubuntu -- curl es:9200       <-- returns ... "cluster_name" : "elasticsearch"...
	kubectl exec ubuntu -- curl kibana:5601   <-- returns ... var defaultRoute = '/app/kibana'...

# LoadBalancer: access kibana from public IP

Create load-balancer https://github.com/kubernetes/contrib/tree/master/service-loadbalancer
Give 1 or more nodes the loadbalancer role (so you can balance with public DNS later)

    kubectl label node 185.19.30.121 role=loadbalancer
    kubectl create -f service-loadbalancer.yaml

# Access kibana
loadbalancer_node_ip:5601

# fluentd (log collecter)

Create fluentd parsing config:

    kubectl create configmap fluentd-conf --from-file=fluentd
    kubectl describe configmap fluentd-conf
	
Deploy fluent on all nodes (DaemonSet)

    kubectl create -f fluentd.yaml

# See logs in kibana

Check logs coming in kibana, you just need to refresh, select Time-field name : @timestamps + create
Load and view the dashboard: management > Saved Object > Import > dashboard/elk-v1.json

# Add kubenetes dashboard addon (not elk)
kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
Check (carefull dashboard is running in namespace=kube-system )

    kubectl get all --all-namespaces

Loadbalancer should already be forwarding public query to service dashboard. However you will probably have to restart lb pod or recreate it
Access it: loadbalancer_node_ip:8888 


# Troubleshooting

If issue connecting to svc for ex elasticsearch:
- Use: kubectl exec ubuntu -- curl es_internal_service_ip:9200
- Check port 9200 on node running es container: ssh -i ~/.ssh/id_rsa_foobar core@185.19.29.212 netstat -plunt
- Uncomment type: NodePort and nodePort: 39200
- Check data in es
    kubectl exec ubuntu -- curl es:9200/_search?q=*
    curl node_ip:39200/_search?q=*       <-- if type: NodePort set in es.yaml

No logs coming in kibana:
- check that there are file in node: ssh -i ~/.ssh/id_rsa_foobar core@185.19.29.212 ls /var/log/containers

Can't connect to k8s-dashboard addon:
- Carefull, you can't curl a service inside another namespace. If curling svc in --kube-system, ubuntu container needs to be in that namespace too!
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

	
# add another node?
Edit ansible-cloudstack/k8s.yml and run again the deploy

# Shell Alias for K8s
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
