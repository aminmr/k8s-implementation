# Ingress Implementation

For first implementation, I chose `Nginx-ingress`. The following document is about how to deploy nginx-ingress.

## HaProxy Configuration

First of all you need to add `frontend` and `backend` section to your haproxy configuration file. The backend port for your master nodes will be configured randomly by the NodePort is assinged to ingress Controller.

```shell
frontend kubernetes-ingress-http
    bind *:80
    default_backend kubernetes-nodes-http

backend kubernetes-nodes-http
    mode http
    balance roundrobin
    option tcp-check
    server kmaster1 192.168.32.10:30540 check fall 3 rise 2
    server kmaster2 192.168.32.20:30540 check fall 3 rise 2
    server kmaster3 192.168.32.30:30540 check fall 3 rise 2
```



first clone the repositroy and checkout to the latest release branch(By the time of writing this document the latest lelease was v2.1):

```bash
git clone https://github.com/nginxinc/kubernetes-ingress/
cd kubernetes-ingress/deployments
git checkout v2.1
```

## Configure the RBAC

```shell
cd deployments
kubectl apply -f common/ns-and-sa.yaml
kubectl apply -f rbac/rbac.yaml
kubectl apply -f rbac/ap-rbac.yaml

```

## Create Common Resources

```shell
kubectl apply -f common/default-server-secret.yaml
kubectl apply -f common/nginx-config.yaml
kubectl apply -f common/ingress-class.yaml
```

## Create Custom Resources

As document said:

> By default, it is required to create custom resource definitions for VirtualServer, VirtualServerRoute, TransportServer and Policy. Otherwise, the Ingress Controller pods will not become Ready

```shell
kubectl apply -f common/crds/k8s.nginx.org_virtualservers.yaml
kubectl apply -f common/crds/k8s.nginx.org_virtualserverroutes.yaml
kubectl apply -f common/crds/k8s.nginx.org_transportservers.yaml
kubectl apply -f common/crds/k8s.nginx.org_policies.yaml
```

## Deploy the Ingress Controller

You can deploy the Ingress Controller in two ways:

- DaemonSet: Use a DaemonSet for deploying the Ingress controller on every node or a subset of nodes
- Deployment: Use a Deployment if you plan to dynamically change the number of Ingress controller replicas

I prefer to deploy in DaemonSet way.

```shell
kubectl apply -f daemon-set/nginx-ingress.yaml
```

### Create a Service for the Ingress Controller Pods

```shell
kubectl create -f service/nodeport.yaml
```

Now check the NodePort and change the haproxy configuration related to this port.

![NodePort](/home/amin/k8s-implementation/images/NodePort.png)

## Very very simple test

For testing the functionality I deployed the following `ingress-resource.yaml` file with very very very simple nginx deployment.

nginx-deployment.yaml

```bash
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

nginx-resource.yaml

```bash
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.aminmr.ir
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-deployment
            port:
              number: 80
```

## More

For more configuration and other things you can read the [Nginx installation documents](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/)

