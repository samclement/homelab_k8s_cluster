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

Coredns defaults use `/etc/resolv.conf` for proxy/upstream lookups. If this does not work for your system you can apply the following to use google nameservers:

- `kubectl apply -f coredns_configmap.yml`

Update `coredns` to version `1.2.2` (optoinal):

- `kubectl -n kube-system edit deployment coredns`

```
spec:
  template:
    spec:
      containers:
      - args:
        image: k8s.gcr.io/coredns:1.2.2
```

- https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#example
- https://github.com/kubernetes/minikube/issues/2027
      
## Allow pods to launch on master node

- `kubectl taint nodes --all node-role.kubernetes.io/master-`
      
## Install virtual networking

- `sysctl net.bridge.bridge-nf-call-iptables=1`
- `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml`

## Installing helm

- `kubectl apply -f helm_rbac.yml`
- `helm init --service-account tiller`

## Installing k8s dashboard (optional)

Current dashboard release `1.10.0` uses `heapster` for host metrics, however kubernetes `1.11` has deprecated heapster in favour of `metrics-server`. Until dashboard supports `metrics-server`, `heapster` needs to be installed manually:

- `helm install --namespace kube-system stable/heapster`
- `helm install --namespace kube-system stable/kubernetes-dashboard -f k8s_dashboard_values.yml`

To access the dashboard with port-forwarding:

- `kubectl port-forward <POD_NAME> 8443:8443`

Then access at https://localhost:8443

## Installing cert-manager

Cert-manager will provision TLS certificates from providers like LetsEncrypt:

- `helm install --name cert-manager --namespace kube-system -f cert_manager_values.yml stable/cert-manager`
- `k apply -f certificate_issuers.yml`

### Ingress

- `helm install --name ingress --namespace ingress stable/nginx-ingress -f ingress_values.yml`

This will install an ingress controller and default backend. Prometheus monitoring is also made available for both containers using prometheus annotations.

## Installing Prometheus and Grafana

Prometheus will create persistent volume claims. To enable this create a `hostpath` volume provisioner:

- `kubectl apply -f hostpath_storage.yml`

It is now possible to install prometheus:

- `helm install --name mon --namespace monitoring  stable/prometheus`

To view the prometheus dashboard: `kubectl -n monitoring port-forward <prometheus_server_pod> 9090`.  
To view prometheus alert manager: `kubectl -n monitoring port-forward <prometheus_alert_manager> 9093`

Grafana:

- `helm install --name grafana --namespace monitoring -f grafana_values.yml stable/grafana`

#### Oauth2 Ingress

Ingress routes can be protected with Oauth2 authentication https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/auth/oauth-external-auth

1. Create an Oauth2 application in Github (https://grafana.swhurl.com, https://grafana.swhurl.com/oauth2)
2. Replace placeholders for `client_id`, `client_secret` in `grafana_oauth2_proxy.yml`
3. Create a cookie secret and replace the `cookie_secret` placeholder in `grafana_oauth2_proxy.yml`
4. `kubectl apply -f grafana_oauth2_proxy.yml`

This will create an oauth2 deployment that handles oauth callbacks for the hostname configured (in this case grafana.swhurl.com).

## Installing fluent-bit to stream logs to loggly

Set up a loggly account and get a `customer_token`.

### Fluent-bit:

- `kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-service-account.yaml`
- `kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role.yaml`
- `kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role-binding.yaml`
- `kubectl create secret generic --from-literal=<client_token> --namespace logging "loggly-secret"`

- `kubectl create -f fluent-bit-configmap.yml`
- `kubectl create -f fluent-bit-ds.yml`

https://github.com/fluent/fluent-bit-kubernetes-logging
https://moisesbm.wordpress.com/2018/08/25/kubernetes-with-fluent-bit-to-send-logs-to-loggly/

## Uninstalling

- `kubectl drain <node name> --delete-local-data --force --ignore-daemonsets`
- `kubectl delete node <node name>`
- `kubeadm reset`

