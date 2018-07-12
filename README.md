# kubeadm-TeslaGPU
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
kubectl apply -f manifests/
```

