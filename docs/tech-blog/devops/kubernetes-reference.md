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

# drain a node https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/
k drain [node] --ignore-daemonsets --delete-emptydir-data

# edit a deployment
k edit deployment [deployment] -n [namespace]

# set 0 pod for a deployment
k scale deployment [deployment] -n [namespace] --replicas=0
```

## DBInstance

``` bash
k get dbinstance [instance-name] -n [namespace]

# to get address and port
k get dbinstance [instance-name] -n [namespace] -o json | jq -r '.status.endpoint.address'
k get dbinstance [instance-name] -n [namespace] -o json | jq -r '.status.endpoint.port'
```

## Helm

``` bash
# sample workflow to install prometheus from helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm search repo prometheus-community
helm upgrade --install -n istio-system prometheus prometheus-community/prometheus

# to install with customized value file and specific version
helm upgrade --install -n istio-system prometheus prometheus-community/prometheus \
  -f values.yaml \
  --version 23.3.0

# to uninstall
helm uninstall prometheus

# to check template to be generated
helm template . -f values.yaml -f values2.yaml
```

## InitContainer

Init containers are speicalized containers that run before app containers in the same pod. One usage of init container is to download file and share it with the app container via shared volume. To learn more, check out official [Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/).

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mydb
  template:
    metadata:
      labels:
        app: mydb
    spec:
      containers:
        - name: mydb
          image: postgres
          env:
          - name: POSTGRES_PASSWORD
            value: example
          ports:
          - containerPort: 5432
            name: postgres
          volumeMounts:
            - name: data
              mountPath: /docker-entrypoint-initdb.d
      initContainers:
        - name: curl-downloader
          image: appropriate/curl
          args:
            - "-o"
            - "/tmp/data/init.sql"
            - "https://raw.githubusercontent.com/januschung/support-system-db/main/sql-scripts/create_tables.sql"
          volumeMounts:
            - name: data
              mountPath: /tmp/data
      volumes:
        - name: data
          emptyDir: {}
```