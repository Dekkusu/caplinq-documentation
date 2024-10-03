
# Task 1: Kubernetes Deployment

DevOps Engineer Assessment from Caplinq

## Guide for configuration

### Configure dotnet on local/Ubuntu server
```
sudo apt-get update
sudo apt-get install -y dotnet-sdk-8.0
sudo apt-get install -y aspnetcore-runtime-8.0
sudo apt-get install -y dotnet-runtime-8.0
```

To run dotnet test

*make sure to run this inside the directory where .sln file is located*

```
dotnet test Caplinq.DevOps.sln
```

For Building

```
dotnet build
```

### K8s Deployment Setup/Naming

Namespace: dotnet

Deployment:  dotnet-web-deployment

Service: dotnet-web-service (with Load Balancer)

### How to Access Kubernetes Cluster

### Installing kubectl
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Access linode k8s
First to get access download the kubeconfig file found on linode kubernetes UI

then run 

```
export KUBECONFIG=<yaml file>
```
Get deployments

```
kubectl get pods -n dotnet
```

Access deployments

```
kubectl exec -it -n dotnet <deployment_name> -- bash
```

View services

```
kubectl get services -n dotnet
```


## Deployment

PR to main branch github actions will automatically run

See .github/workflows/main.yml to view CI/CD Pipeline


# Manifest files setup

*follow this guide in sequence*

To make deployment setup cleaner, deployments are contained inside a namespace

`dotnetapinamespace.yaml`

This line specifies the name of our namespace
```
metadata:
  name: dotnet
```

To create one inside the cluster run

```
kubernetes apply -f dotnetapinamespace.yaml
```

To setup deployments, a pods to where our dockerfile will be running is found inside

`dotnetapi.yaml`

The following lines are essential to configure, this sets up how many replicas of our deployment will be made and where to get the docker image.

```
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-api-deployment
  template:
    metadata:
      labels:
        app: web-api-deployment
      namespace: dotnet
    spec:
      containers:
      - name: web-api-deployment
        image: dekkusu/devops-things:caplinq
        imagePullPolicy: Always
        ports:
        - containerPort: 80
```

In this case, 2 replicas are set (important to have more than 1 to make sure downtime is reduced in case of sudden errors or updates).

As for the dockerfile, it is pointing to the docker repository where the github action pushes the docker image created.

To deploy deployments run

```
kubernetes apply -f dotnetapi.yaml
```


For the services, this where the networking is configured (becomes more important when DB is added to not mess up the whole deployment)

`dotnetapiservice.yaml`

To deploy services run

```
kubernetes apply -f dotnetapiservice.yaml
```
