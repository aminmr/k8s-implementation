# Manual Monitoring

This section is about manual monitoring. In `helm monitoring` section I deployed the monitoring via the prometheus comunity helm repo. In this section all the process has done by manual manifests. 

## Metrics

### Kube-state-metrics

The kube-state-metrics is deployed on `kube-system` namespace and is a simple service that listens to the Kubernetes API server and generates metrics about the state of the objects.

For sample and blind deploy you can use the following command:

```shell
kubectl apply -f kube-state-metrics//examples/standard
```

### Metric-server

Before starting to deploy the prometheus and other components, you need to install the `metric-server`. 

Metrics Server is a scalable, efficient source of container resource metrics for Kubernetes built-in autoscaling pipelines.

You need to first install it and then follow the steps.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

After deploying the metric-server it won't be up and running. The reason is due to the TLS. To disable the tls connection you need to add the ‍`- --kubelet-insecure-tls` to the `args` section in pod yaml file:

```bash
args:
- --cert-dir=/tmp
- --secure-port=4443
- --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
- --kubelet-use-node-status-port
- --kubelet-insecure-tls
```

### Node-exporter

For node metrics I used `node-exporter`. node-exporter needs to deployed as a `DaemonSet` to expose all nodes metrics. The node-exporter mount `/sys` and `/` for expose metrics.

To deploy the node-exporter follow the commands:

```shell
kubectl apply -f node-exporter/nodeExporterDaemonSet.yaml
kubectl apply -f node-exporter/nodeExportersvc.yaml
```

## Prometheus

### ClusterRole

First of all we are going to implement the prometheus stack. The prometheus needs to fetch the kubernetes metrics. But for security consideration we need to create `ClusterRole` and `ClusterRoleBinding` to only access the `monitoring` namespace to get the metrics.

**Note:** If you want to add more resources of your kubernetes cluster, you need to edit the clusterrole manifests and apply it.

You can apply the `clusterRole.yaml` as following:

```shell
kubectl apply -f prometheus-stack/clusterRole.yaml
```

### ConfigMap

The most important part of prometheus is `prometheus.yaml` for configurations and `prometheus.rules` for prometheus rules. Both these two files are available in one file.

If you want to change any configuration you can apply it through this `ConfigMap`.

You can apply the `configMap.yaml `as following:

```shell
kubectl apply -f prometheus-stack/prometheus-configMap.yaml
```

`prometheus.rules:`

Prepare in future updates.

`prometheus.yaml:`

- node-exporter
- kubernetes-apiservers
- kubernetes-nodes
- kubernetes-pods
- kube-state-metrics
- kubernetes-cadvisor
- kubernetes-service-endpoints

### Deployment

Now it's time to apply the deployment. These configuration are important in prometheus-deployment:

1. Add prometheus ConfigMap(Prometheus.yaml and prometheus.rules)
2. Prometheus args.
3. Prometheus pvc for data store (This should be add in near future)

### Services

For accessing the prometheus you need to provide a service.

You can deploy this service as a `ClusterIP` via the following yaml file:

```shell
kubectl apply -f prometheus-stack/prometheus-deployment.yaml
```

## AlertManager

### ConfigMap

This ConfigMap contains `config.yaml` for alertManager. You can deploy via the following command:

```shell
kubectl apply -f alertManager/alertManagerConfigmap.yaml
```

### TemplateConfigMap

This ConfigMap contains `default.tmpl` for alertmanager template. You can deploy via the following command:

```shell
kubectl apply -f alertManager/alertManagerTemplateConfigmap.yaml
```

### Deployment

And the deployment which use the official image and two previous template to deploy the alertmanager. Try the following command:

```shell
kubectl apply -f alertManager/alertManagerDeployment.yaml
```

### Service

And the service is to access the alertManager:

```
kubectl apply -f alertManager/alertManagerService.yaml
```

## Grafana

### DataSource

This ConfigMap contains `prometheus.yaml` file which contains the datasource. Our datasource is prometheus. To apply this configmap:

```shell
kubectl apply -f grafana/grafana-datasource-config.yaml
```

### Deployment

```shell
kubectl apply -f grafana/grafana-deployment.yaml
```

### Service

```shell
kubectl apply -f grafana/grafana-deployment.yaml
```

## ETCD Monitoring

### ETCD cert generating

1. First of all, Download and install the `cfssl`:

   ```shell
   mkdir ~/bin
   curl -s -L -o ~/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
   curl -s -L -o ~/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
   chmod +x ~/bin/{cfssl,cfssljson}
   export PATH=$PATH:~/bin
   ```

2. Generate the Confugure CA options:

   ```shell
   mkdir ~/cfssl
   cd ~/cfssl
   cfssl print-defaults config > ca-config.json
   cfssl print-defaults csr > ca-csr.json
   ```

3. Because we already have the `CA` we skip generating the CA file.

4. Generate the Client Certificate:

   ```shell
   echo '{"CN":"client","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -ca=/etc/ssl/etcd/ssl/ca.pem -ca-key=/etc/ssl/etcd/ssl/ca-key.pem -config=ca-config.json -profile=client - | cfssljson -bare client
   ```

5. Now you have the new certificates. Let's go to the next step.

### Create the secret

For using these certificates via prometheus we need to create a new `secret` in the monitoring namespace:

```shell
kubectl -n monitoring create secret generic kube-etcd-client-certs --from-file=/etc/ssl/etcd/ssl/ca.pem --from-file=client.pem --from-file=client-key.pem
```

### Apply in configuration

Other configuration included adding certificates in prometheus deployment and prometheus has been done in this repo. For further description:

1. First you need to add the prometheus scrapes in `prometheus.yaml` which is in prometheus `configMap`

2. Add the secret to the prometheus `deployment` 

3. Apply both prometheus `configMap` and `deployment`:

   ~~~shell
   kubectl apply -f prometheus-stack/prometheus-configMap.yaml
   kubectl apply -f prometheus-stack/prometheus-deployment.yaml
   ~~~

   

## References

https://devopscube.com/setup-prometheus-monitoring-on-kubernetes/
