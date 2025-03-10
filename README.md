
# 2. Install PKG

## Ansible Install 

Insatll Ansible Node.

Install SSH on Kubernetis nodes.

Generate ssh-keys on local machine and copying the pub key to vm's

```
ssh-keygen -t rsa

ssh-copy-id username@vm-ip-address
```

create file name on /etc/ansible: inventory.ini with the vm's ip.

```
ansible all -i /etc/ansible/inventory.ini -m ping
```



# 3.Install K8s

## Prevent Installation of RKE2 Built-in Nginx Ingress

create file name on /etc/rancher/rke2/: config.yaml



## RKE2 Install

### Server Node Installation

```
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh -

systemctl enable rke2-server.service

systemctl start rke2-server.service

systemctl status rke2-server.service
```

### Linux Agent (Worker) Node Installation

```
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

mkdir -p /etc/rancher/rke2/

cat << EOF > /etc/rancher/rke2/config.yaml
token: <token from first server node>  (path to token on server /var/lib/rancher/rke2/server/node-token) # Use the same token on all nodes
tls-san:
  - First Node IP   
  - Second Node IP
  - Third Node IP
server: https://[First Node IP]:9345
disable:
  - rke2-ingress-nginx   
EOF

systemctl enable rke2-agent.service

systemctl start rke2-agent.service
```



### Install additional Server Node

```
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh -

mkdir -p /etc/rancher/rke2/

cat << EOF > /etc/rancher/rke2/config.yaml
token: <token from first server node>  (path to token on server /var/lib/rancher/rke2/server/node-token) # Use the same token on all nodes
tls-san:
  - First Node IP   
  - Second Node IP
  - Third Node IP
server: https://[First Node IP]:9345 
disable:
  - rke2-ingress-nginx 
EOF

systemctl enable rke2-server.service

systemctl start rke2-server.service

systemctl status rke2-server.service
```



### Add Kubectl to PATH

configuration for connecting the k8s cluster 

```
sudo nano ~/.bashrc
	
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml PATH=$PATH:/var/lib/rancher/rke2/bin

source ~/.bashrc
```



### RKE2 Remove Node 

Run on the node to be removed:

```
cd /usr/local/bin/rke2

rke2-killall.sh

rke2-uninstall.sh
```

Run on the master node:

```
kubectl delete node [node name]
```


### Manually Remove the Built-in NGINX Ingress Deployment

```
kubectl delete namespace kube-system --ignore-not-found=true

kubectl delete deployment rke2-ingress-nginx-controller -n kube-system

kubectl delete daemonset rke2-ingress-nginx-controller -n kube-system

kubectl delete service rke2-ingress-nginx-controller -n kube-system

kubectl delete clusterrolebinding rke2-ingress-nginx

kubectl delete clusterrole rke2-ingress-nginx

kubectl delete serviceaccount rke2-ingress-nginx -n kube-system

sudo systemctl restart rke2-server

kubectl get pods -A | grep ingress
```

### Drain and Reboot Node (If Necessary)

```
kubectl drain $(hostname) --ignore-daemonsets --delete-local-data

sudo reboot

kubectl get pods -A | grep ingress
```

  
  
# 4. Basic Resources

## Install HELM 

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```


## Install Nginx Ingress with HELM 

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update
```

create file name: nginx-values.yaml

``` 
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace default \
  --values nginx-values.yaml
```  

  
### Test Nginx Ingress 

```
kubectl run nginx-pod --image=nginx --restart=Never --port=80 -n default

kubectl expose pod nginx-pod --type=NodePort --port=80 --name=nginx-service
```

create file name: test-ingress.yaml


                         
## Add NFS Storage to the Cluster 

### Install NFS Server on the host

```
sudo apt-get update

sudo apt-get install nfs-kernel-server -y

sudo mkdir -p /srv/nfs/kubedata

sudo chown nobody:nogroup /srv/nfs/kubedata

sudo chmod 777 /srv/nfs/kubedata

echo "/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

sudo exportfs -ra
```

### Install NFS Client on Server\Agent node

```
sudo apt-get update

sudo apt-get install nfs-common -y
```

### Create Storage Class

create file name: rbac.yaml
  
create file name: deployment.yaml
       
create file name: storage-class.yaml

```
kubectl apply -f rbac.yaml

kubectl apply -f deployment.yaml

kubectl apply -f storage-class.yaml   
```

### Test the Storage

create file name: test-pvc.yaml

``` 
kubectl apply -f test-pvc.yaml

kubectl get pvc
```

create file name: pvc-inspector.yaml

```
kubectl apply -f pvc-inspector.yaml

kubectl get pods

kubectl exec -it pvc-inspector -- sh

kubectl cp rbac.yaml pvc-inspector:/data/
```



# 5. Install Monitoring Resources

## Create TLS Secret

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=kubernetes" \
  -addext "subjectAltName = IP:192.168.122.64,IP:192.168.122.137,IP:192.168.122.139"

kubectl create namespace monitoring

kubectl create secret tls monitoring-tls \
  --key tls.key \
  --cert tls.crt \
  -n monitoring
```


## Install Prometheus and Grafana

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
```
helm pull prometheus-community/kube-prometheus-stack -d .
mkdir kube-prometheus-stack-69.5.2
tar -xvf kube-prometheus-stack-69.5.2.tgz -C kube-prometheus-stack-69.5.2/
helm install prometheus Desktop/sysops_ex/kube-prometheus-stack-69.5.2/kube-prometheus-stack/ -n monitoring
```

### Create Ingress for Prometheus and Grafana

create file name: prometheus-ingress.yaml

```
kubectl apply -f prometheus-ingress.yaml
```


## Install ArgoCD
```
helm repo add argo https://argoproj.github.io/argo-helm  
helm repo update
```
```
kubectl create namespace argocd
```
```
helm install argocd argo/argo-cd -n argocd
```
```
kubectl get pods -n argocd -o wide
```
```
kubectl get secrets -n argocd argocd-initial-admin-secret -o json | jq .data.password -r | base64 -d
```
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Install ArgoCD CLI

```
wget https://github.com/argoproj/argo-cd/releases/download/v2.14.2/argocd-linux-amd64
```
```
mv argocd-linux-amd64 argocd
chmod +x argocd
mv argocd /usr/local/bin/
```
```
argocd login [node ip:argocd-server port]
```

### Use all partition size /dev/ubuntu-vg/ubuntu-lv

```
sudo lvdisplay /dev/ubuntu-vg/ubuntu-lv
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
df -h
```



# 6. Install Logging Resources

```
kubectl create namespace kube-logging
```
```
helm repo add elastic https://helm.elastic.co
helm repo update
```
```
helm pull elastic/elasticsearch -d .
mkdir elasticsearch-8.5.1
tar -xvf elasticsearch-8.5.1.tgz -C elasticsearch-8.5.1/
helm install elasticsearch Desktop/sysops_ex/elasticsearch-8.5.1/elasticsearch/ -n kube-logging
```