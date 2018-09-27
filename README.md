# k8s Cluster

## Dependencies

- `sudo apt-get install -y kubeadm kubelet kubectl`

## Install k8s via kubeadm

- `sudo su`
- `swapoff -a`
- `kubeadm init`
- `exit`

Setting up kube config for the cluster:

- `mkdir -p $HOME/.kube`
- `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`
- `sudo chown $(id -u):$(id -g) $HOME/.kube/config`

Edit `coredns` configmaps to use google nameservers:

- `kubectl edit configmaps coredns -n kube-system`

``
data:
  upstreamNameservers: |
    ["8.8.8.8","8.8.4.4"]
``

- https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#example
- https://github.com/kubernetes/minikube/issues/2027
      
## Allow pods to launch on master node

- `kubectl taint nodes --all node-role.kubernetes.io/master-`
      
## Install virtual networking

- `kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

## Installing helm

- `kubectl apply -f rbac.yml`
- `helm init --service-account tiller`

## Installing k8s dashboard

Current dashboard release `1.10.0` uses `heapster` for host metrics, however kubernetes `1.11` has deprecated heapster in favour of `metrics-server`. Until dashboard supports `metrics-server`, `heapster` needs to be installed manually:

- `helm install --namespace kube-system stable/heapster`
- `helm install --namespace kube-system stable/kubernetes-dashboard -f k8s_dashboard_values.yml`

To access the dashboard with port-forwarding:

- `kubectl port-forward <POD_NAME> 8443:8443`

Then access at https://localhost:8443

## Uninstalling

- `kubectl drain <node name> --delete-local-data --force --ignore-daemonsets`
- `kubectl delete node <node name>`
- `kubeadm reset`
