# kubeadm-TeslaGPU

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
```



Pretty basic single node not HA kubeadm deploy using openpower ppc64le / Tesla GPU

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



