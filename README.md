
# Ansible Install 

Insatll Ansible Node.
Install SSH on Kubernetis nodes.


generate ssh-keys on local machine and copying the pub key to vm's
	ssh-keygen -t rsa

ssh-copy-id username@vm-ip-address

create file name on /etc/ansible: inventory.ini with the vm's ip.

ansible all -i /etc/ansible/inventory.ini -m ping



# Prevent Installation of RKE2 Built-in Nginx Ingress RKE2 

create file name on /etc/rancher/rke2/: config.yaml



##############################################################################
############################## RKE2 Install ##################################
##############################################################################

Server Node Installation
------------------------

curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh -

systemctl enable rke2-server.service
systemctl start rke2-server.service
systemctl status rke2-server.service


Linux Agent (Worker) Node Installation
--------------------------------------

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



Install additional Server Node
------------------------------

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



#######################################################################################
################################ Add Kubectl to PATH ##################################
#######################################################################################

configuration to connecting the k8s cluster 

sudo nano ~/.bashrc
	export KUBECONFIG=/etc/rancher/rke2/rke2.yaml PATH=$PATH:/var/lib/rancher/rke2/bin

source ~/.bashrc



#########################################################################
########################### RKE2 Remove Node ############################
#########################################################################

Run on the node to be removed:

cd /usr/local/bin/rke2
rke2-killall.sh
rke2-uninstall.sh

Run on the master node:
kubectl delete node [node name]



Manually Remove the Built-in NGINX Ingress Deployment
-----------------------------------------------------

kubectl delete namespace kube-system --ignore-not-found=true
kubectl delete deployment rke2-ingress-nginx-controller -n kube-system
kubectl delete daemonset rke2-ingress-nginx-controller -n kube-system
kubectl delete service rke2-ingress-nginx-controller -n kube-system
kubectl delete clusterrolebinding rke2-ingress-nginx
kubectl delete clusterrole rke2-ingress-nginx
kubectl delete serviceaccount rke2-ingress-nginx -n kube-system

sudo systemctl restart rke2-server
kubectl get pods -A | grep ingress


Drain and Reboot Node (If Necessary)
------------------------------------

kubectl drain $(hostname) --ignore-daemonsets --delete-local-data
sudo reboot

kubectl get pods -A | grep ingress

  
  
##############################################################################################
####################################### Install HELM #########################################
##############################################################################################

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh
./get_helm.sh



#############################################################################
###################### Install Nginx Ingress with HELM ######################
#############################################################################

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

create file name: nginx-values.yaml
 
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace default \
  --values nginx-values.yaml
  

  
#############################################################################
############################# Test Nginx Ingress ############################
#############################################################################

kubectl run nginx-pod --image=nginx --restart=Never --port=80 -n default
kubectl expose pod nginx-pod --type=NodePort --port=80 --name=nginx-service


create file name: test-ingress.yaml


                         
########################################################################################
########################### Add NFS Storage to the Cluster #############################
########################################################################################

Install NFS Server on the host
------------------------------

sudo apt-get update
sudo apt-get install nfs-kernel-server -y

sudo mkdir -p /srv/nfs/kubedata
sudo chown nobody:nogroup /srv/nfs/kubedata
sudo chmod 777 /srv/nfs/kubedata

echo "/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -ra


Install NFS Client on Server\Agent node
---------------------------------------

sudo apt-get update
sudo apt-get install nfs-common -y


Create Storage Class
--------------------

create file name: rbac.yaml
  
create file name: deployment.yaml
       
create file name: storage-class.yaml

kubectl apply -f rbac.yaml
kubectl apply -f deployment.yaml
kubectl apply -f storage-class.yaml   


Test the Storage
----------------

create file name: test-pvc.yaml
  
kubectl apply -f test-pvc.yaml
kubectl get pvc
           
              



