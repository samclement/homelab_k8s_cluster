# k8s Cluster

## Dependencies

- `sudo apt-get install -y kubeadm kubelet kubectl`

## Install k8s via kubeadm

- `sudo su`
- `swapoff -a`
- `kubeadm init --pod-network-cidr=10.244.0.0/16` (flag is necessary for flannel networking)
- `exit`

Setting up kube config for the cluster:

- `mkdir -p $HOME/.kube`
- `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`
- `sudo chown $(id -u):$(id -g) $HOME/.kube/config`

Enable the `kubelet` service:

- `sudo systemctl enable kubelet.service`

## Allow pods to launch on master node

- `kubectl taint nodes --all node-role.kubernetes.io/master-`
      
## Install virtual networking

- `sysctl net.bridge.bridge-nf-call-iptables=1`
- `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

Once the virtual networking pod is running, the `coredns` pods should change status from `pending` to `running`.

## Install dynamic DNS

- `https://github.com/samclement/aws-dns-updater`

## Installing cert-manager

Cert-manager will provision TLS certificates from providers like LetsEncrypt:

- `kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.11.0/cert-manager.yaml`

### Ingress

Installing `ingress-nginx` from the supplied repo's manifests puts ingress in a `ingress-nginx` namespace. If you want ingress in your own namespace, you'll need to modify the manifests. 

Clone the repo:

- `git clone https://github.com/kubernetes/ingress-nginx/`
- `cd ingress-nginx`

Modify the namespace:

- Update `deploy/cloud-generic/kustomization.yaml` with the new namespace
- Update `deploy/static/mandatory.yaml` with the new namespace
- `kubectl apply -f deploy/static/mandatory.yaml`
- `kustomize build deploy/cloud-generic | kubectl apply -f -`

## Installing fluent-bit to stream logs to loggly

Create a `logging` namespace:

Set up a loggly account and get a `customer_token`.

- `kubectl create namespace logging`

### Fluent-bit:

- `kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-service-account.yaml`
- `kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role.yaml`
- `kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role-binding.yaml`
- `kubectl create secret generic --from-literal=customer_token=<client_token> --namespace logging "loggly-secret"`

- `kubectl create -f fluent_bit_configmap.yaml`
- `kubectl create -f fluent_bit_ds.yaml`

https://github.com/fluent/fluent-bit-kubernetes-logging
https://moisesbm.wordpress.com/2018/08/25/kubernetes-with-fluent-bit-to-send-logs-to-loggly/

## Installing minio

Kubernetes needs to make a `StorageClass` available for minio:

- `kubectl apply -f hostpath_storage.yaml`

Install minio:

- `kubectl create -f https://github.com/minio/minio/blob/master/docs/orchestration/kubernetes/minio-standalone-pvc.yaml?raw=true`
- `kubectl create -f https://github.com/minio/minio/blob/master/docs/orchestration/kubernetes/minio-standalone-deployment.yaml?raw=true`
- `kubectl create -f https://github.com/minio/minio/blob/master/docs/orchestration/kubernetes/minio-standalone-service.yaml?raw=true`
- `kubectl apply -f minio_ingress.yaml`

## Uninstalling

- `kubectl drain <node name> --delete-local-data --force --ignore-daemonsets`
- `kubectl delete node <node name>`
- `kubeadm reset`

