# OpenEBS

OpenEBS is one of the most popular Kubernetes storage solutions.

OpenEBS [Local](https://openebs.io/docs#local-volumes) and [Distributed](https://openebs.io/docs#replicated-volumes) volumes are implemented using a collection of OpenEBS Data Engines. OpenEBS Control Plane integrates deeply into Kubernetes and uses Kubernetes to manage the provisioning, scheduling and maintenance of OpenEBS Volumes.

In this document I used the simple `Local HostPath` solution that OpenEBS developed.

I prefer to use OpenEBS instead of built-in local kubernetes storage because of the `dynamic provisioning` and other solution that I'll like to challenge with them  later.

## Installation

First, you need to create the `BasePath`. The default BasePath for OpenEBS is `/var/openebs/local`.

### Kubectl 

Now you need to apply the following manifest:

```shell
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
```

In my case, I would like to test other features of OpenEBS but if you just want to use the HostPath you can install the lite version of OpenEBS:

```shell
kubectl apply -f https://openebs.github.io/charts/openebs-operator-lite.yaml 
kubectl apply -f https://openebs.github.io/charts/openebs-lite-sc.yaml
```

### Helm Chart

```shell
helm repo add openebs https://openebs.github.io/charts
helm repo update
helm install --namespace openebs --name openebs openebs/openebs
```

Verify the installation by the following commands:

```shell
kubectl get pods -n openebs -l openebs.io/component-name=openebs-localpv-provisioner
kubectl get sc
```

## Create PVC

Now you can create a PVC and test the functionality. To create the PVC define the `hostPath.yaml` with the following parameters:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: local-hostpath-pvc
spec:
  storageClassName: openebs-hostpath
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5G
```

and apply it:

```shell
kubectl apply -f local-hostpath-pvc.yaml
```

Now check if the pvc is created sucessfully:

```shell
kubectl get pvc local-hostpath-pvc
```

## Create Pod With PVC

To test the PVC you can create a simple Pod or deployment. In this section refer to the official document I deploy a simple pod by `pod-localpath.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-local-hostpath-pod
spec:
  volumes:
  - name: local-storage
    persistentVolumeClaim:
      claimName: local-hostpath-pvc
  containers:
  - name: hello-container
    image: busybox
    command:
       - sh
       - -c
       - 'while true; do echo "`date` [`hostname`] Hello from OpenEBS Local PV." >> /mnt/store/greet.txt; sleep $(($RANDOM % 5 + 300)); done'
    volumeMounts:
    - mountPath: /mnt/store
      name: local-storage
```

and apply it:

```shell
kubectl apply -f local-hostpath-pod.yaml
```

After that the pod is running you can exec the pod and create any files under the `/mnt/store` and redeploy the pod to check the storage functionality.

## References

For further read check the following links:

[Main OpenEBS Document Page](https://openebs.io/docs)

[LocalPV-hostPath OpenEBS](https://openebs.io/docs/user-guides/localpv-hostpath)

[Kubernetes Built-in Local Storage](https://kubernetes.io/docs/concepts/storage/storage-classes/#local)