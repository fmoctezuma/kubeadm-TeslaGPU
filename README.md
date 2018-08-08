# kubeadm-TeslaGPU
Pretty basic single node not HA kubeadm deploy using openpower ppc64le / Tesla GPU

Install cuda
```
sudo apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/ppc64el/7fa2af80.pub
sudo apt-get update
sudo apt-get install cuda
```

Setup Docker to use nvidia as a container runtime 
```
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey |sudo apt-key add
curl -s -L https://nvidia.github.io/nvidia-docker/ubuntu16.04/amd64/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
apt-get install -y nvidia-docker2
```

Verify Docker is running using nvidia
```
sudo docker info | grep nvidia
Runtimes: runc nvidia
```

Run nvidia/cuda-ppc64le
```
docker run -it --rm --runtime nvidia nvidia/cuda-ppc64le nvidia-smi

+-----------------------------------------------------------------------------+
| NVIDIA-SMI 396.26                 Driver Version: 396.26                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla P100-SXM2...  Off  | 00000002:01:00.0 Off |                    0 |
| N/A   36C    P0    32W / 300W |      0MiB / 16280MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  Tesla P100-SXM2...  Off  | 00000003:01:00.0 Off |                    0 |
| N/A   36C    P0    32W / 300W |      0MiB / 16280MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   2  Tesla P100-SXM2...  Off  | 0000000A:01:00.0 Off |                    0 |
| N/A   36C    P0    33W / 300W |      0MiB / 16280MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   3  Tesla P100-SXM2...  Off  | 0000000B:01:00.0 Off |                    0 |
| N/A   37C    P0    30W / 300W |      0MiB / 16280MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+

```

### Kubernetes deployment

------

Init kubeadm with flannel network (10.244.0.0/16 is a pre-req) and enable CoreDNS
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --feature-gates="CoreDNS=true"
```

This is a single node kubeadm deploy, so remove the label from master node
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Deploy flannel using the manifest for ppc64le (or depends your arch)
```
kubectl apply -f manifests/kube-flannel-ppc64le-noresourcelimit.yml
```
*Note:* At this point couldn't find calico support for ppc64le, so I'm sticking with flannel for now.

During kubeadm init CoreDNS was enabled, Modify CoreDNS to be schedule on a master node, or just apply the example on manifest directory

```
kubectl apply -f manifests/coredns_single_master.yaml
```
### Preparing your GPU Nodes

The following steps need to be executed on all your GPU nodes.
Additionally, this README assumes that the NVIDIA drivers and nvidia-docker has been installed.

First you will need to check and/or enable the nvidia runtime as your default runtime on your node.
We will be editing the docker daemon config file which is usually present at `/etc/docker/daemon.json`:
```json
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```
> *if `runtimes` is not already present, head to the install page of [nvidia-docker](https://github.com/NVIDIA/nvidia-docker)*

The second step is to enable the `DevicePlugins` feature gate on all your GPU nodes.

If your Kubernetes cluster is deployed using kubeadm and your nodes are running systemd you will have to open the kubeadm
systemd unit file at `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` and add the following environment argument:
```
Environment="KUBELET_EXTRA_ARGS=--feature-gates=DevicePlugins=true"
```

> *If you spot the Accelerators feature gate you should remove it as it might interfere with the DevicePlugins feature gate*

Reload and restart the kubelet to pick up the config change:
```shell
$ sudo systemctl daemon-reload
$ sudo systemctl restart kubelet
```

> In this guide we used kubeadm and kubectl as the method for setting up and administering the Kubernetes cluster,
> but there are many ways to deploy a Kubernetes cluster.
> To enable the `DevicePlugins` feature gate if you are not using the kubeadm + systemd configuration, you will need
> to make sure that the arguments that are passed to Kubelet include the following `--feature-gates=DevicePlugins=true`.

### Enabling GPU Support in Kubernetes

Once you have enabled this option on *all* the GPU nodes you wish to use,
you can then enable GPU support in your cluster by deploying the following Daemonset:



Install NVIDIA device plugin for Kubernetes (This case v1.11)
```
kubectl apply -f https://raw.githubusercontent.com/fmoctezuma/kubeadm-TeslaGPU/master/manifests/k8s-device-plugin-ppc64le.yaml
```

Test cuda vector-add on ppc64el, I had to build my own Docker image based on ppc64le for cuda-vector-add, see manifests to change for a custom image if desired.
Docker file can be found here:
https://github.com/fmoctezuma/kubeadm-TeslaGPU/blob/master/cuda-vector-add/Dockerfile

```
kubectl apply -f manifests/cuda-vector.yaml
kubectl logs cuda-vector-add
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

This means GPUs are consumed by kubernetes, the next step will be deploy kubeflow using ksonnet.io
At this point there is no ksonnet binary for ppc64el, so let's compile one.
Golang should be installed on the system, I'm going to skip that part.

Let's get ksonnnet and compile it to ppc64el
```
go get github.com/ksonnet/ksonnet
cd $GOPATH/src/github.com/ksonnet/ksonnet
```
Add the following line to the Makefile, creating another install supporting ppc64el, see example below
```
install-ppc64le:
        CGO_ENABLED=0 GOOS=linux GOARCH=ppc64le $(GO) build -o $(GOPATH)/bin/ks $(GO_FLAGS) ./cmd/ks
```

Generate the binary
```
$GOPATH/src/github.com/ksonnet/ksonnet[master] → make install-ppc64le
```

Binary should be installed, if not double check $GOPATH/bin/ks exists
```
ks -help
```

#### Deploy tensorflow operator

Install/compile golang/dep for ppc64
```
go get go get github.com/golang/dep/
cd $GOPATH/github.com/golang/dep/
```
Assuming we are on the ppc64le server, Make script takes `ARCH := $(shell go env GOARCH)` to build the package
```
make build
```









