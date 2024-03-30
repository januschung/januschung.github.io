# Note to build k8s homelab

## Install MiniKube on Ubuntu

``` bash

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb


sudo groupadd docker
sudo usermod -aG docker $USER
newgrp 
sudo chmod 666 /var/run/docker.sock

minikube start
```


### Deploy hello-minikube and expose it remotely

``` bash
kubectl create deployment hello-minikube --image=kicbase/echo-server:1.0
kubectl expose deployment hello-minikube --type=NodePort --port=8080
kubectl port-forward --address 0.0.0.0 service/hello-minikube 8080:8080
kubectl port-forward service/hello-minikube 8080:8080 # this will only expose it within the host

```
Figured out I cannot connect to minikube app remotely easily. Time to switch to k3s.

## Install K3s on Ubuntu

```bash
curl -sfL https://get.k3s.io | sh -
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

Create sample app that can be accessed within LAN

```bash
kubectl create deployment hello-world --image=kicbase/echo-server:1.0
kubectl expose deployment hello-world --type=NodePort --port=8080
```

install argocd 

```bash 
export ARGOCD_VERSION=v2.10.0
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/$ARGOCD_VERSION/manifests/core-install.yaml
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/$ARGOCD_VERSION/manifests/core-install.yaml

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/$ARGOCD_VERSION/manifests/install.yaml


```