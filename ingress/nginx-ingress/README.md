# Nginx-Ingress Implementation

For the first implementation, I chose `Nginx-ingress`. The following document is about how to deploy Nginx-ingress.

## HaProxy Configuration

First of all, you need to add the `frontend` and `backend` sections to your Haproxy configuration file. The backend port for your master nodes will be configured randomly by the NodePort is assigned to the ingress Controller.

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

First clone the repository and checkout to the latest release branch(By the time of writing this document the latest release was v2.1

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

> By default, it is required to create custom resource definitions for VirtualServer, VirtualServerRoute, TransportServer and Policy. Otherwise, the Ingress Controller pods will not become Ready.

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

Now check the NodePort and change the Haproxy configuration related to this port.

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

Create the service for the deployment:

```shell
kubectl expose deploy nginx-deployment --port 80
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

## References

For more configuration and other things you can read the following:

Nginx Documents:

- [Nginx installation documents](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/)

And these articles which are not completely correct! :

- [jhooq article](https://jhooq.com/ingress-controller-nginx/#4-setup-kubernetes-ingress-controller)

- [dev.to article](https://dev.to/mrturkmen/nginx-ingress-controller-with-haproxy-for-k8s-cluster-52e1#setup-nginx-ingress-controller)
