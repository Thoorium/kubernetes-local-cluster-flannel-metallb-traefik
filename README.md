# Local Kubernetes cluster with flannel, metallb and traefik
This guide will set you up a local kubernetes cluster. It is the result of me trying many different things and eventually getting everything to run smoothly. I am not too familiar with all the Kubernetes components yet so this guide is mostly "run command X to do Y".
## Prerequisites
- Ubuntu 18.04 on a virtual machine or bare metal computer
- At least 1 CPU with 2 cores
- At least 4GB of memory
- At least 20GB of storage space
  
Repeat this configuration for all the nodes you want in your cluster.

## 1. Host preparation
Disable the swap. Kubernetes doesn't want swap enabled.
```bash
sudo nano /etc/fstab

# Find a line that looks like this
/swap.img      none    swap    sw      0       0

# And comment it out to this
#/swap.img      none    swap    sw      0       0
```
Do your host updates.  
```bash
sudo apt update
sudo apt upgrade -y
```

## 2. Container environment
`Source https://docs.docker.com/v17.09/engine/installation/linux/docker-ce/ubuntu/`  
`Source https://docs.docker.com/v17.09/engine/installation/linux/linux-postinstall/`  

Install docker CE.
```bash
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
```bash
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
```bash
sudo apt-get update
sudo apt-get install docker-ce
```
Add the current user to the docker group.
```bash
sudo usermod -aG docker $USER
```
Reboot to apply the docker and swap changes.
```bash
sudo reboot now
```
## 3. Install kubernetes
`Source https://kubernetes.io/docs/setup/independent/install-kubeadm/`  

Install kubeadm, kubelet and kubectl. This must be done as root! 
```bash
sudo su
```
```bash
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

## 4. Install the master node
`Source https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/` 

Create the master node using Flannel as the CNI.
```bash
sysctl net.bridge.bridge-nf-call-iptables=1
kubeadm init --pod-network-cidr=10.244.0.0/16
```
Take note of the join token for the other nodes.  

Go back to a normal user.
```bash
exit
```
Allow the current user to use kubectl with the cluster.
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Install Flannel.
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
```
### Optionnal
Allow pods to be created on the master node.
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## 5. Install the dashboard
`Source https://github.com/kubernetes/dashboard`
`Source https://github.com/kubernetes/dashboard/wiki/Access-control`
`Source https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/`

### Option 1: Secured (Preferred - But currently broken)
Create the dashboard using the recommended method.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
```

Then create an admin user able to see everything.

```bash
kubectl apply -f dashboard/kubernetes-dashboard-admin.yaml
```

Retrieve the Bearer Token for authentication on the login page.

```bash
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

After exposing the dashboard, copy the token into the token input field and press the login button.

### Option 2: Unsecured
This will expose everything without requiring to login. Do not use this in production.
```bash
kubectl apply -f dashboard/kubernetes-dashboard-unsecured.yaml
```

### Expose the dashboard
Start a proxy to view the dashboard (Replace the IP with the one of your kubernetes master node host).
```bash
kubectl proxy --address 192.168.0.41 --port 8001 --accept-hosts='^*$'
```
View the dashboard at http://192.168.0.41:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/.

If you used the unsecured option, use the `skip` button to login.

## 6. Install a load balancer to allow nodes to communicate with the external world
`Source https://metallb.universe.tf/installation/`  
`Source https://metallb.universe.tf/configuration/`  

We'll use Metallb for this scenario.
```bash
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.1/manifests/metallb.yaml
```
Create a config file to give a range of ip address the load balancer can assign. Make sure those IPs can't be assigned by your DHCP server.

Create a file named `metallb-config.yaml` with the following content:  
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.0.50-192.168.0.99 # Change the range here
```
Apply the load balancer config.
```bash
kubectl apply -f metallb-config.yaml
```

## 7. Configure an ingress controller
`Source https://docs.traefik.io/user-guide/kubernetes/`  
`Source https://medium.com/@dusansusic/traefik-ingress-controller-for-k8s-c1137c9c05c4`  

The ingress controller in this scenario will act as a reverse-proxy for your applications. This will enable domain binding, routing, url rewrite, etc. For more information, please visit the traefik documentation at https://docs.traefik.io/configuration/backends/kubernetes/.

Create a namespace for traefik.
```bash
kubectl create namespace traefik
```
Deploy the traefik config.
```bash
kubectl apply -f traefik/traefik-config.yaml
```
Apply the ClusterRoleBinding.
```bash
kubectl apply -f traefik/traefik-rbac.yaml
```
Deploy the traefik reverse-proxy and dashboard.
```bash
kubectl apply -f traefik/traefik-deployment.yaml
```
Create the ingress rule for the dashboard and bind it to a domain.  
Edit this file if you want to use another domain than `traefik-ui.kube`.
```bash
kubectl apply -f traefik/traefik-ingress-dashboard.yaml
```
You should now be able to visit the domain and view the dashboard. 
Since the dashboard is served via traefik, there should be frontend/backend rule for it.

## 8. Configure another node
On another host, do steps 1 to 3.  
Use the join token you received during step 4 on the master node to join the cluster:
```bash
# This is an exemple. Use your token that was generated during step 4
kubeadm join 192.168.0.41:6443 --token xj28lv.7u8t1d1judei6eqz --discovery-token-ca-cert-hash sha256:bd1f5cf392bb4329ec48b8036340378b2468a424efebef89a226c5edfedf7042
```

## 9. Deploy an app (Optionnal)
Based on the cheese demo from traefik.  
`Source https://docs.traefik.io/user-guide/kubernetes`  

Create the cheese namespace.
```bash
kubectl create namespace cheese
```
Create the deployment. The most important part of the config is the label `k8s-app: traefik-ingress-lb` that is repeated under the spec section. Without this label, traefik won't be able to communicate with the backend.
```bash
kubectl apply -f cheese/cheese-deployment.yaml
```
Deploy the service. Same thing again, the label `k8s-app: traefik-ingress-lb` is crucial for traefik.
```bash
kubectl apply -f cheese/cheese-service.yaml
```
Create the ingress. The first two annotations seems to be required for the frontend functionnalities.  
Edit this file if you want to use another domain than `cheeses.kube`.
```bash
kubectl apply -f cheese/cheese-ingress.yaml
```
You should be able to visit the domain and view a picture of the cheese, according to the path you visit.

- http://cheeses.kube/cheddar/
- http://cheeses.kube/stilton/
- http://cheeses.kube/wensleydale/

For technical reasons, the path must end with `/` for the picture to load.
