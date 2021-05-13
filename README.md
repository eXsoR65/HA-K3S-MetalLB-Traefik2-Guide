
## K3S Cluster Setup with etcd embedded DB, Managed with Rancher
Setting up a HA K3S Cluster with embedded etcd DB, Rancher, metallb and traefik2

### I have to give credit to [TechnoTim](https://github.com/techno-tim) 
#### (This is based on his [guide](https://techno-tim.github.io/posts/k3s-traefik-rancher/) But I opted to use embedded etcd db instead of an external db and show how to crate it from start to end.)

<br>

----
# Part 1: 
##### Setting up and configure Nginx LB for K3S API servers.

You can install Nginx on a Linux machine or in a Docker Container 

Installing Nginx on Ubuntu.
`sudo apt install -y nginx`

Uninstall Nginx (for any reason you may need to).
`sudo apt purge nginx* libnginx* && sudo apt autoremove`

###### If you are using a container then all you need is the config you'll find below. 
###### Otherwise if installing on a VM uncomment the load_module line

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
<br>

----
# Part 2
##### Installing K3S.

I will be choosing to run version v1.20.5+k3s1 so do a before running the script. 
`export INSTALL_K3S_VERSION=v1.20.5+k3s1`

Run the on first server **only** (as it enables Etcd).

```Bash
export INSTALL_K3S_VERSION=v1.20.5+k3s1
curl -sfL https://get.k3s.io | sh -s - server --cluster-init --node-taint CriticalAddonsOnly=true:NoExecute --tls-san ip_address_of_lb --write-kubeconfig-mode 644 --disable traefik --disable servicelb
```


To get the K3S_TOKEN run the following on the first server
`sudo cat /var/lib/rancher/k3s/server/node-token`


Run on the other Server nodes. 
```Bash
export INSTALL_K3S_VERSION=v1.20.5+k3s1
curl -sfL https://get.k3s.io | sh -s - server --server https://ip_address_lb:6443 --token longe_token_here --node-taint CriticalAddonsOnly=true:NoExecute --tls-san ip_address_of_lb --write-kubeconfig-mode 644 --disable traefik --disable servicelb
```


#### Run on the worker nodes. 

```Bash
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

<br>

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
```Bash
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

```Bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.2.0
```

Check rollout of cert-manager
`kubectl get pods --namespace cert-manager`

Now we can proceed to install Rancher. 
```Bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --version 2.5.7
```
(_Note: here we are installing version 2.5.7 change if needed._)

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

```Bash
kubectl expose deployment rancher -n cattle-system --type=LoadBalancer --name=rancher-lb --port=443
```

Check the service to see what IP it was given. Make note of the rancher-lb service EXTERNAL-IP
`kubectl get svc -n cattle-system -o wide`


Now **important** to access the Rancher UI you need to create a local DNS entry for you URL that point to the IP Address it got from Metal LB. 

> example: rancher.domain.com = 192.16.1.240

_Note: you can validate it done right by doing an nslookup rancher.domain.com_

<br>

----
# Part 4
###### Installing and configuring Traefik v2 as a Reverse Proxy.


## Install Traefik 2

You can can choose between creating `Ingress` in Rancher or `IngresRoute` with `traefik` If you choose `IngressRoute` pleasse use the config in `/configingress-route/traefik-config.yaml`

-   You must have a persistent volume set up already for `acme.json` certificate
-   This uses cloudflare, check providers if you want to switch
-   This will get wildcard certs
-   This is pointed at staging, if you want production be sure comment staging the line (and delete your staging certs)

We will be installing this into the `kube-system` namespace, which already exists. If you are going to use anther namespace you will need change it everywhere.

add `traefik` helm repo and update

```
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
```

create `traefik-config.yaml` with the contents from `/config/traefik-config.yaml`

this holds our cloudflare secrets along with a configmap

update this file with your values

apply the config

```
kubectl apply -f traefik-config.yaml
```

create `traefik-chart-values.yaml` with the contents from `/config/traefik-chart-values.yaml`

Update `loadBalancerIP` in `traefik-chart-values.yaml` with your Metal LB IP

Before running this, be sure you only have one default storage class set. If you are using Rancher it is Cluster>Storage>Storage Classes. Make sure only one is default.

create config then update the values

```
kubectl apply -f traefik-config.yaml
```

```
helm install traefik traefik/traefik --namespace=kube-system --values=traefik-chart-values.yaml
```

If all went well, you should now have traefik 2 installed and configured.

## Exposing a service with traefik and Rancher Ingress

In Rancher go to Load Balancing

-   create ingress
-   choose a host name (service.example.com)
-   choose a target (your workload)
-   set the port to the exposed port within the container
-   go to labels and annotations and add `kubernetes.io/ingress.class` = `traefik-external`
-   note, `traefik-external` comes from `--providers.kubernetesingress.ingressclass=traefik-external` in `traefik-chart-values.yml`. If you used something else, you will need to set your label properly.
-   when you visit your website (`https://service.example.com`) you should now see a certificate issues. If it's a staging cert, see the note about switching to production in `traefik-chart-values.yaml`. After changing, you will need to delete your certs in storage and reapply that file

```
kubectl delete -n kube-system persistentvolumeclaims acme-json-certs
kubectl apply -f traefik-config.yaml
```

## Exposing a service with traefik `IngressRoute`

copy the contents os `/config-ingress-route/kubernetes` to your local machine

then run

```
kubectl apply -f kubernetes
```

This will create the deployment, service, and ingress.

## Dashboard

First you will need `htpassword` to generate a password for your dashboard

```
sudo apt-get update
sudo apt-get install apache2-utils
```

You can then genate one using this, be sure to swap your username and password

```Bash
htpasswd -nb techno password | openssl base64
```

it should output

```
dGVjaG5vOiRhcHIxJFRnVVJ0N2E1JFpoTFFGeDRLMk8uYVNaVWNueG41eTAKCg==
```

copy `traefik-dashboard-secret.yaml` locally and update it with your credentials

then apply

```
kubectl apply -f traefik-config.yaml
```

copy `traefik-dashboard-ingressroute.yaml` and update it with your hostname

Save this in a secure place, it will be the password you use to access the traefik dashboard
