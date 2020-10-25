+++
author = "Ricardo Rocha"
date = "2020-10-25T22:00:00+00:00"
description = "Upgrade containerd on a GKE cluster"
tags = ["kubernetes", "gke", "containerd"]
title = "Custom Containerd Versions in GKE Clusters"
+++

GKE (Google Kubernetes Engine) is a strong option for managed Kubernetes
cluster deployments, backed by the Google Cloud Platform. Relying on managed
services reduces the maintenance overhead and building on features such as
auto-repair and auto-upgrade is also a plus.

In some rare cases more flexibility is required, usually involving an early or
very experimental feature in a critical component. In this case, we need a more
recent version of containerd.

The need comes from experiments with the containerd [stargz remote
snapshotter](https://github.com/containerd/stargz-snapshotter), a very cool
tool that dramatically improves the container startup time (especially for
large images) while reducing network usage. Being a very recent tool it
requires containerd >=1.4, while GKE currently offers:
* non containerd node flavors still relying on Docker
* a COS containerd node flavor offering containerd 1.3
* an Ubuntu containerd node flavor offering containerd 1.2

For this experiment we will base on the last option above - Ubuntu containerd - 
and we'll upgrade the node's containerd binaries, in an aggressive way that
could break any hope of auto repair, auto upgrade or GKE support still
being possible after.

Here's how an original GKE cluster node looks like.
```bash
kubectl get node -o wide
NAME                                              STATUS   ROLES    AGE     VERSION             INTERNAL-IP   EXTERNAL-IP    OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
gke-stargz-west4-001-default-pool-8ee80358-xrjk   Ready    <none>   4m38s   v1.17.12-gke.1501   10.164.0.9    34.91.157.64   Ubuntu 18.04.5 LTS   5.4.0-1024-gcp   containerd://1.2.10
```

And here's the script that will very quickly break the node and bring it back
with our required containerd version.
```bash
$ cat setup-ctr-14.sh
#!/bin/bash
set -x
set +e

# download / extract cri containerd 1.4
rm -rf usr/ etc/ opt/ cri-containerd*
wget --quiet https://github.com/containerd/containerd/releases/download/v1.4.1/cri-containerd-cni-1.4.1-linux-amd64.tar.gz
tar zxvf cri-containerd-cni-1.4.1-linux-amd64.tar.gz

# stop the kubelet and delete all running containers
sudo systemctl stop kubelet || true
sudo crictl ps | grep Runnin | awk '{print $1}' | xargs sudo crictl rm -f || true

# stop containerd and copy the new binaries in place
sudo systemctl stop containerd || true
sudo ps auxw | grep containerd | awk '{print $2}' | xargs sudo kill -9 || true
sudo cp usr/local/bin/* /usr/bin
sudo rm -rf /opt/cni && sudo cp -R opt/cni /opt/

# restart containerd and the kubelet (the wait between comes from trial and error)
sudo systemctl start containerd
sleep 10
sudo systemctl start kubelet
```

And that's it, go ahead and run this on all your nodes.
```bash
export ZONE=europe-west4-a
for n in $(kubectl get node -o custom-columns='DATA:metadata.name' | grep gke); do
	scp -o StrictHostKeyChecking=no setup-ctr-14.sh ${n}.${ZONE}.nimble-valve-236407:~/
	ssh -o StrictHostKeyChecking=no ${n}.${ZONE}.nimble-valve-236407 ~/setup-ctr-14.sh
done
```

You should almost immediately see the result.
```bash
kubectl get node -o wide
NAME                                              STATUS   ROLES    AGE     VERSION             INTERNAL-IP   EXTERNAL-IP    OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
gke-stargz-west4-001-default-pool-8ee80358-xrjk   Ready    <none>   7m52s   v1.17.12-gke.1501   10.164.0.9    34.91.157.64   Ubuntu 18.04.5 LTS   5.4.0-1024-gcp   containerd://1.4.1
```

Kubernetes is doing the real magic, with the node containers
being recreated as soon as the kubelet reports back - this is really a proof of
Kubernetes's resilience to failure, as we pretty much kill everything running
on the node.
