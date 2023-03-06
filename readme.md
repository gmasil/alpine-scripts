# Kubernetes Cluster on Alpine

## Manual Prerequisites

Your machine where you are going to run the provisioning from should have ansible installed.

This host also has to be able to login into all nodes via SSH without password prompt, so install your SSH key to all machines.

Login to all hosts and install python:

```bash
apk add python3 --update-cache
```

## Install Prerequsites with Ansible

Checkout this repo or download the files.

Make sure to change the inventory file according to your nodes.

```bash
ansible-playbook -i inventory prerequisites.yml
```

## Create Cluster

Login to your **master node** and execute:

```bash
kubeadm init --pod-network-cidr=192.168.0.0/16

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml

watch kubectl get pods -n calico-system
```

The last command is interactive and will show you when the calico pods are up and running, wait until everything is up.


## Join Worker Nodes to Cluster

Generate the join commands:

```bash
kubeadm token create --print-join-command
```

Login into all your worker nodes one after another and execute the command you got from the `kubeadm token create` command, e.g.:

```bash
kubeadm join 10.10.10.126:6443 --token 6vs48o.qevhyvc2zwbvkqot --discovery-token-ca-cert-hash sha256:df9ce7d4b3e7d727802fb292851c3b756a15596c8158b5386fd9a450aa2a8ad8
```

## Check Cluster Status

On your master node execute:

```bash
kubectl get nodes
kubectl get all
```
