## Basic

``` bash
# export KUBECONFIG
export KUBECONFIG=~/.kube/[config-file]

# create a namespace
k create ns [namespace]

# check current cluster
k config current-context
k config get-contexts

# describe
k describe [type] [name] -n [namespace]

# get everything
k get all
k get [type] -A
```

## DBInstance

``` bash
k get dbinstance [instance-name] -n [namespace]

# to get address and port
k get dbinstance [instance-name] -n [namespace] -o json | jq =r '.status.endpoint.address'
k get dbinstance [instance-name] -n [namespace] -o json | jq =r '.status.endpoint.port'
```
