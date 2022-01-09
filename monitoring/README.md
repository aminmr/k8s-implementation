# Monitoring

Monitoring is one of the most important needs for kubernetes cluster to monitor all kubernetes components.

In this section, I deployed the Prometheus + Grafana monitoring via Helm chart.

## Prerequisites

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

### Kube-proxy

The metrics bind address of kube-proxy is default to `127.0.0.1:10249` that prometheus instances **cannot** access to. You should expose metrics by changing `metricsBindAddress` field value to `0.0.0.0:10249` if you want to collect them.

```bash
kubectl -n kube-system edit cm kube-proxy
```

```
apiVersion: v1
data:
  config.conf: |-
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    # ...
    # metricsBindAddress: 127.0.0.1:10249
    metricsBindAddress: 0.0.0.0:10249
    # ...
  kubeconfig.conf: |-
    # ...
kind: ConfigMap
metadata:
  labels:
    app: kube-proxy
  name: kube-proxy
  namespace: kube-system
```

### Kube-Control-Manager

The kube-control-manager default metricsBindAddresss id `0.0.0.0` but there is another problem with this component. In the latest releases, `--port=0` was added to the kube-control-manager yaml file. This parameter means the insecure port which is `10252` is disabled. The secure port is `10257`. At the time of writing this document, I haven't found a way to use the secure port. So I enabled the insecure port by deleting the `--port=0` parameter from the pod yaml files.

## Installing

Now it's the time to deploy the helm chart. The helm chart is `prometheus-community`chart which is contains a collection of Kubernetes manifests, [Grafana](http://grafana.com/) dashboards, and [Prometheus rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/) combined with documentation and scripts to provide easy to operate end-to-end Kubernetes cluster monitoring with [Prometheus](https://prometheus.io/) using the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator).

### Get Repo Info

First you need to add the `prometheus-community` repo.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Installation

Time to install the chart:

```
helm install stable prometheus-community/kube-prometheus-stack
```

### Customization

In many situations, you need to customize the configuration. For example, I deployed the etcd in my cluster on my host and it's not the pods in `kube-system` namespace. So I need to add etcd configuration to the helm chart. To see the configurations:

```bash
helm show values prometheus-community/kube-prometheus-stack
```

For saving the configurations:

```bash
helm show values prometheus-community/kube-prometheus-stack > values.yaml
```

After changing the configuration you can `upgrade` the chart by the following command:

```bash
helm upgrade stable prometheus-community/kube-prometheus-stack -f values.yaml
```

### ETCD on host

By my mistake, I was deployed the etcd on the host. So I need to expose the metrics and configure them for the prometheus.

In values I changed the following configurations:

![etcd-values](../images/etcd-values.png)

## Problem not solved yet

1. The secure port of kube-controll-manager
2. Some grafana dashboards still don't have any data that is probably related to the metrics

## Refereneces

[Metric-Server](https://github.com/kubernetes-sigs/metrics-server)

[Metric-server ignore tls](https://github.com/kubernetes-sigs/metrics-server/issues/196#issuecomment-746363974)

Kube-Proxy default metricsBindAddress:

- [Link1](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#kubeproxy)

- [Link2](https://github.com/helm/charts/issues/16476#issuecomment-528681476)

[Kube-Scheduler & Kube-Control-Manager issues](https://github.com/kubernetes/kubernetes/issues/93194)
