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
export INTERNAL_IP=10.1.1.1
# keep subnet at 10.244.0.0/16
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address $INTERNAL_IP --control-plane-endpoint $INTERNAL_IP

mkdir ~/.kube
ln -s /etc/kubernetes/admin.conf /root/.kube/config

# fix internal IP of master node
sed -i "s/\"$/ --node-ip=$INTERNAL_IP\"/" /var/lib/kubelet/kubeadm-flags.env
service kubelet restart # if that does not work: reboot

kubectl get nodes -o wide # verify that all internal IPs are actually internal not external

kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

watch kubectl get pods --all-namespaces
```

The last command is interactive and will show you when the flannel and coredns pods are up and running, wait until everything is up.


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

## Run Nginx Deployment

```bash
kubectl apply -f https://raw.githubusercontent.com/gmasil/alpine-scripts/master/nginx-deployment.yaml
kubectl expose deployment nginx-deployment --port=80 --type=LoadBalancer

kubectl get pods
kubectl get deployments
kubectl get services
```

The last command will show you the node port where your nginx service is reachable:

```
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP      10.96.0.1       <none>        443/TCP        14m
nginx-deployment   LoadBalancer   10.108.80.176   <pending>     80:32508/TCP   6m30s
```

So now you can see the Nginx welcome page under `http://<your-master-ip>:32508/`.

## Ingress Stuff

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.6.4/deploy/static/provider/cloud/deploy.yaml
kubectl patch service ingress-nginx-controller -p '{"spec":{"externalIPs":["10.10.10.137"]}}' --namespace ingress-nginx
```

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
?? name: example.com
?? namespace: default
?? annotations:
?????? nginx.ingress.kubernetes.io/rewrite-target: /
spec:
?? ingressClassName: nginx
?? rules:
?? - host: example.com
?????? http:
?????????? paths:
?????????????? - path: /apple
?????????????????? pathType: Prefix
?????????????????? backend:
?????????????????????? service:
?????????????????????????? name: apple-service
?????????????????????????? port:
?????????????????????????????? number: 5678

```

kubectl apply -f ingress.yml
