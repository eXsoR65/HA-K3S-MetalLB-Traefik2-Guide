# HA-K3S-MetalLB-Traefik2
Setting up a HA K3S Cluster with emmended etcd DB, Rancher, metallb and traefik2


## K3S Cluster Setup with Etcd embedded DB

----

# Part 1: 
##### Setting up and configure Nginx LB for K3S API servers.

Installing Nginx on Ubuntu.
`sudo apt install -y nginx`

Uninstall Nginx (for any reason you may need to).
`sudo apt purge nginx* libnginx* && sudo apt autoremove`

##### Nginx conf file found in: /etc/nginx/nginx.conf
Example configuration for K3S

```conf
#Uncomment this next line if you are NOT running nginx in docker
#load_module /usr/lib/nginx/modules/ngx_stream_module.so;

events {}

stream {
  upstream k3s_servers {
    server ip_address:6443;
    server ip_address:6443;
	server ip_address:6443;
  }

  server {
    listen 6443;
    proxy_pass k3s_servers;
  }
}
```

----

# Part 2
##### Installing K3S.

I will be choosing to run version v1.20.5+k3s1 so do a before running the script. 
`export INSTALL_K3S_VERSION=v1.20.5+k3s1`

Run the on first server **only** (as it enables Etcd).

```
export INSTALL_K3S_VERSION=v1.20.5+k3s1
curl -sfL https://get.k3s.io | sh -s - server --cluster-init --node-taint CriticalAddonsOnly=true:NoExecute --tls-san 172.16.1.50 --write-kubeconfig-mode 644 --disable traefik --disable servicelb
```


To get the K3S_TOKEN run the following on the first server
`sudo cat /var/lib/rancher/k3s/server/node-token`


Run on the other Server nodes. 
```
export INSTALL_K3S_VERSION=v1.20.5+k3s1
curl -sfL https://get.k3s.io | sh -s - server --server https://ip_address_lb:6443 --token longe_token_here --node-taint CriticalAddonsOnly=true:NoExecute --tls-san ip_address_of_lb --write-kubeconfig-mode 644 --disable traefik --disable servicelb
```


#### Run on the worker nodes. 

```
export INSTALL_K3S_VERSION=v1.20.5+k3s1
k3s_token="long_token_here"
k2s_url="https://ip_address_of_lb:6443"
curl -sfL https://get.k3s.io | K3S_URL=${k2s_url} K3S_TOKEN=${k3s_token} sh -
```

 #### **Important: **
You need to install kubectl on your dev machine to manage the cluster.  - [Install Kubectl](https://kubernetes.io/docs/tasks/tools/) Then copy your kube config to you dev machine.
 - To view your kube config on a server do: `sudo cat /etc/rancher/k3s/k3s.yaml` 
 - Past the config to your dev machine at `~/.kube/config`
*Note: Be sure to update the `server:` to your load balancer IP Address or Hostname.*

----

# Part 3
##### Installing Rancher.

**Note:** It's advised you consult the [Rancher Support Matrix](https://rancher.com/support-maintenance-terms/all-supported-versions) to get the recommended version for all Rancher dependencies.

[https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/#1-install-the-required-cli-tools](https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/#1-install-the-required-cli-tools)

Install Helm 
`curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash`

Add the Helm stable repository. 
`helm repo add rancher-stable https://releases.rancher.com/server-charts/stable`

Create rancher namespace
`kubectl create namespace cattle-system`

##### SSL configuration
###### Use rancher generated (default)

- Install cert-manager
```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager.crds.yaml
```

- Create name-space for cert-manager.
`kubectl create namespace cert-manager`

- Add the Jetstack Helm repository.
`helm repo add jetstack https://charts.jetstack.io`

- Update Helm repository.
`helm repo update`

- Install `cert-manager` helm chart

_Note: If you receive an "Error: Kubernetes cluster unreachable" message when installing cert-manager, try copying the contents of "/etc/rancher/k3s/k3s.yaml" to "~/.kube/config" to resolve the issue._

```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.2.0
```

Check rollout of cert-manager
`kubectl get pods --namespace cert-manager`

Now we can proceed to install Rancher. 
```
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --version 2.5.7
```
(_Note: here we are installing version 2.5.7 change is needed._)

### Now before we can access the Rancher UI  we need to install a load balancer and expose it as a service. 

## Install Metal LB

[Metal LB installation](https://metallb.universe.tf/installation/)

Metal LB ConfigMap
(_note: replace the IP Address range with your own_)

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
      - 192.168.1.240-192.168.1.250
```


## Exposing Rancher directly to your Metal LB

It's a good idea to do this until traefik is configured otherwise you won't have access to the Rancher Ui

```
kubectl expose deployment rancher -n cattle-system --type=LoadBalancer --name=rancher-lb --port=443
```
