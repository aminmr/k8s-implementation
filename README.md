# k8s-implementation
This is my first attempt to implementing my own Kubernetes cluster. In this process I face a lot of challenges and I will write all these items in here.

## Cluster Specification

a Load-balancer: Ha-proxy
3 Master Nodes + 1 Worker Node
Kubeadm implementation method

CRI: Docker
CNI: Weave-net
CSI: ?
Ingress: ?

Network:

Load-balancer: 192.168.32.5 + Public interface

Master Nodes:
- Master-1: 192.168.32.10
- Master-2: 192.168.32.20
- Master-3: 192.168.32.30

Worker Node:
- Worker-1: 192.168.32.40

## OS Challenges via IaaS

1. The most important challenges was private network gateway. If you have multiple interface on one VM(Public and Private), You do not need this feature and please disable it via IaaS Panel

2. My Haproxy node has not get local IP address automatically via netplan. The problem after too many attempts was about the interface MAC Address. So always checkout the interface's MAC Address through IaaS Panel.

3. After migration of master-2 the filesystem has corrupted and the docker service has not start anymore. So I needed to use `fsck` tools in rescue mode to repair the filesystem.
![Filesystem](./images/docker-failure.png)

This failure and fsck lost metadata of docker. There is no solution and you should 

## Kubernetes Challenges

 For first time I was implement Kubernetes via `Kubeadm`. This method of implementation was common and has its own challenges:

1. In the process of installation, the `kubeadm init` command was not run correctly. The error is realated to kubelet and docker(CRI). After reading the log I found that the problem was related to `cgroup`. The cgroup of kubelet and docker was not the same and caused the problem. The solution:

   add the following lines to `/etc/docker/daemon.json` and restart the docker service to change the docker cgroup to systemd:

   ```shell
   {
           "exec-opts": ["native.cgroupdriver=systemd"]
   }
   ```

   **Note:** In Kubernetes documentation, the best practice was using the `systemd`  related to the performance and etc.

   

2. After Kubernetes implementation, the API and etcd components were restarted repeatedly. The logs were following:

<img src="./images/etcd-log-1.png" alt="Filesystem" />

![Filesystem](./images/etcd-log-2.png)

![Filesystem](./images/etcd-log-3.png)

after reading these logs I was checked the disk latency. The disk latency was unbelievable!

![Disk-latency](./images/dd-test-1.jpg)

62 seconds! I was checked the exact command on multiple region(mobinnet , Amesterdam , etc) and the result was much more better:

![Disk-latency](./images/dd-test-2.jpg)

So to help the etcd work fine as mentioned in the Kubernetes documents, I added volumes to VMs and after backup the etcd, mount volumes to the `/var/lib/etcd` .

### Scenario 1 

For Disk migration on the the Openstack Volumes, I prefer try multiple scenarios to try the etcd actions and undrestanding etcd cluster.

In first scenario we try the following steps:

1.  Stop Docker and Kubelet on one of the master nodes.
2. mount the disk on `/var/lib/etcd` 
3. Start the Docker and Kubelet

After the above steps the etcd on our node was failed. After reading the logs, the following logs was `warn` logs that should be considered:

1. After mount the disk to `/var/lib/etcd` there was a `lost+found` directory which should be deleted.
2. The permission was set to default permisson(755). The recommended permission for etcd data directory is 700.
3. The fatal error was about `cluster ID mismatch`. Unfortunately the etcd cluster will not repair our specific member! 

So I tried to remove the member from cluster and rejoin it. I tried the following commands:

```shell
sudo ETCDCTL_API=3 etcdctl  --endpoints=https://192.168.32.10:2379,https://192.168.32.20:2379,https://192.168.32.30:2379  --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key  --cacert=/etc/kubernetes/pki/etcd/ca.crt  member remove 49aecc6b82b2b4ae
```

now for adding the member:

```shell
sudo ETCDCTL_API=3 etcdctl  --endpoints=https://192.168.32.10:2379,https://192.168.32.20:2379,https://192.168.32.30:2379  --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key  --cacert=/etc/kubernetes/pki/etcd/ca.crt  member add aminyan-k8s-2 --peer-urls=https://192.168.32.20:2380,https://192.168.32.30:2379
```

but after add the member to the cluster and restart pod(remove/recreate) the member state was `unstarted` on cluster! So after all these tries we try the second scenario to restore the etcd member...

### Scenario 2 

So after failure in scenario 1 after mount the disk on `/var/lib/etcd` we need to restore etcd backup. First of all list all etcd member:

```shell
sudo ETCDCTL_API=3 etcdctl  --endpoints=https://192.168.32.10:2379,https://192.168.32.20:2379,https://192.168.32.30:2379  --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key  --cacert=/etc/kubernetes/pki/etcd/ca.crt  member list -w table
```

 and then By following command you can make a snapshot:

```shell
sudo ETCDCTL_API=3 etcdctl --endpoints https://192.168.32.10:2379,https://192.168.32.20:2379,https://192.168.32.30:2379 --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key  --cacert=/etc/kubernetes/pki/etcd/ca.crt snapshot save /home/ubuntu/snapshot-12-19-1.db
```

Now copy the snapshot file to the node you want to recover and follow the steps:

```shell
sudo ETCDCTL_API=3 etcdctl  --endpoints https://192.168.32.20:2379  --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key  --cacert=/etc/kubernetes/pki/etcd/ca.crt  --initial-cluster=aminyan-k8s-2=https://192.168.32.20:2380,aminyan-k8s-master-1=https://192.168.32.10:2380,aminyan-k8s-3=https://192.168.32.30:2380  --initial-cluster-token=etcd-cluster-1  --initial-advertise-peer-urls=https://192.168.32.20:2380  --name=aminyan-k8s-2  --skip-hash-check=true  --data-dir /var/lib/etcd/amin snapshot restore /root/snapshot-12-19.db
```

**Notes:**

1. `--initial-cluster-token=etcd-cluster-1` could be any name.
2. Because when you want to restore snapshot the `--data-dir` should not be exited, so we need to restore the etcd on another directory! Be careful you need to change the data directory in `etcd.yaml` (/etc/kubernetes/manifests/etcd.yaml)

To check the endpoint health you can use the following command:

```shell
sudo ETCDCTL_API=3 etcdctl  --endpoints=https://192.168.32.10:2379,https://192.168.32.20:2379,https://192.168.32.30:2379  --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key  --cacert=/etc/kubernetes/pki/etcd/ca.crt  endpoint health

```



### Kubespray Challenges

The Config file and hosts yaml of Kubespray implementation way was stored on this git repository, but in this section I writing about the challenges I face!

#### Firewall Problem

After execute the ansible playbook and when the cluster was up, the `calico` pods are crashed. After check the pods log I found that the Calico BGP network sockets and etc can't communicate to each other! By disabling the firewall the pods are get running! In Calico documentation you need to allow the following ports:

![Calico](./images/calico-ports.png)

### Hostnames Renaming!

After deploying the k8s with kubespray ansible playbook, by my mistake I forgot to rename the hostnames in inventory files. So the kubespray rename the OS hostnames and sync it to the inventory file!

So  I need a way to rename the hostnames, but all the kubernetes components depends on the hostname!

The mistakes I've done:

1. try to remove one of the nodes manually and rejoin it to the cluster. Don't ask it what happened!
2. After that before run `remove-node.yaml` kubespray playbook, I run `cluster.yaml` again with new hostnames in inventory file. It completely ruined the cluster. Never ever run the `cluster.yaml` again before remove the nodes.

#### Pod's eviction problem

After all I've done, I tried to run `remove-node.yaml` or `reset.yaml` but the playbook freezed on some steps. So I tried to drain nodes and delete it manually. The nodes are freezed on drain process. After googled on it I found the [solution](https://medium.com/@felipedutratine/when-you-try-to-drain-a-kubernetes-node-but-it-blocks-5aba9592d7c9).

The problem was pods that came to `terminating` state but blocked!

![Terminating](./images/terminating-state.jpg)

 So I delete pods by force:

```shell
kubectl delete pod <pod_name> -n=<namespace> --grace-period=0 --force
```

### Misconfiguration in Kubespray

After redeploy the k8s, the CoreDNS pod were crashed! The error in describe of pods was:

```shell
Warning  FailedCreatePodSandBox  5m17s                kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "958cb4c2c74154b1f734689afbbe99d675e4e771f6bf635fc1f71edb3ac9dc59": error getting ClusterInformation: Get "https://[10.171.128.1]:443/apis/crd.projectcalico.org/v1/clusterinformations/default": context deadline exceeded
```

It could not connected to pod netowrks! I tried many solutions but nothing worked. At last I found the solution on [this](https://issueexplorer.com/issue/projectcalico/calico/5014) website:

![CoreDNS](./images/coreDNS-problem.png)

Yeah! That's right! I indeliberately config No_Proxy and it was the mistake:

![no-proxy](./images/no-proxy-all-yaml.png)

 I manually removed the no_proxy from `vim /etc/systemd/system/containerd.service.d/http-proxy.conf` file and restart the containerd. The problem has solved!

![no-proxy](./images/np-proxy.png)

## E2E test on Kubernetes Cluster

After all the redeploying and disk problem on Arvan asiatech IaaS, I decided to try end to end test on my cluster.

First of all you need to install golang:

```shell
wget https://go.dev/dl/go1.17.5.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.5.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
go version
```

clone the `test-infra` repository:

```shell
git clone https://github.com/kubernetes/test-infra.git
```

and execute the following command:

```
GO111MODULE=on go install ./kubetest
```

