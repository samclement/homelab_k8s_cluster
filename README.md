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

## Installing logging infrastructure (Elasticsearch/Fluent-bit/Kibana)

### Elasticsearch:

- `helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator`
- `helm install --name es --namespace logging incubator/elasticsearch -f elasticsearch_values.yml`

### Fluent-bit:

- `kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-service-account.yaml`
- `kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role.yaml`
- `kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role-binding.yaml`

- `kubectl create -f fluent-bit-configmap.yml`
- `kubectl create -f fluent-bit-ds.yml`

https://github.com/fluent/fluent-bit-kubernetes-logging

### Kibana:

- `helm install stable/kibana --name kib --namespace logging -f kibana_values.yml`

To view aggregated logs:

- `kubectl -n logging port-forward <kibana_pod> 5601`

A pre-configured logging dashboard can be imported by going to: Management > Saved Objets > Import and selecting the `kibana_dashboard.json` configuraiton file.

#### Oauth2 Ingress

Ingress routes can be protected with Oauth2 authentication https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/auth/oauth-external-auth

1. Create an Oauth2 application in Github (https://kibana.swhurl.com, https://kibana.swhurl.com/oauth2)
2. Replace placeholders for `client_id`, `client_secret` in `kibana_oauth2_proxy.yml`
3. Create a cookie secret and replace the `cookie_secret` placeholder in `kibana_oauth2_proxy.yml`
4. `kubectl apply -f kibana_oauth2_proxy.yml`

This will create an oauth2 deployment that handles oauth callbacks for the hostname configured (in this case kibana.swhurl.com).

## Installing Drone CI

Drone provides a simple CI service. It includes its own Github Oauth2 integration and will need a properly configured Github Oauth2 appliction that includes a `client_id` and `client_secret`. The `client_id` can be included in the `drone_values.yml` file, however the secret must be created manually:

- `kubectl create secret generic drone-server-secrets --from-literal=DRONE_GITHUB_SECRET=<client_secret>`

Once the secret has been created, the main application can be installed:

- `helm install --name drone stable/drone -f drone_values.yml`

## Uninstalling

- `kubectl drain <node name> --delete-local-data --force --ignore-daemonsets`
- `kubectl delete node <node name>`
- `kubeadm reset`

