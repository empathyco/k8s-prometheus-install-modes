
One of the most popular tools in the kubernetes ecosystem is [helm](https://helm.sh). Helm is a package manager for kubernetes applications. 
We will install the Prometheus Operator with it. For that we will use [**stable/prometheus-operator** helm chart](https://github.com/helm/charts/tree/master/stable/prometheus-operator).

The chart will install the following components:

- [prometheus-operator](https://github.com/coreos/prometheus-operator)
- [prometheus](https://prometheus.io/)
- [alertmanager](https://prometheus.io/)
- [node-exporter](https://github.com/helm/charts/tree/master/stable/prometheus-node-exporter)
- [kube-state-metrics](https://github.com/helm/charts/tree/master/stable/kube-state-metrics)
- [grafana](https://github.com/helm/charts/tree/master/stable/grafana)

and also will create service monitors to scrape the following internal kubernetes components:

- kube-apiserver
- kube-scheduler
- kube-controller-manager
- etcd
- kube-dns/coredns
- kube-proxy

To install all this components, we will need to run the following commands.

```bash
# First lets create a namespace
kubectl create namespace monitoring
# install the chart
helm install --namespace monitoring po stable/prometheus-operator
# check that chart was install 
helm -n monitoring ls
NAME      	NAMESPACE 	REVISION	UPDATED                             	STATUS  	CHART                     	APP VERSION
prometheus	monitoring	3       	2020-03-04 11:55:34.603854 +0100 CET	deployed	prometheus-operator-8.10.0	0.36.0
```


It is worth noting, the workload controller that it is used depending of the components, meanwhile the alert manager and the prometheus server are deployed as a [statefulset](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/), because they need data persistence, grafana runs as a [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), and it is store the configuration and dashboards in [configmaps](https://kubernetes.io/docs/concepts/configuration/configmap), the same happen with the operator, the reason why will be explain later. Finally the node exporter run as a daemon set to run in all **non master** nodes of the cluster to be able to expose metrics for each one of them. 

```bash
kubectl -n monitoring get pod
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-prometheus-oper-alertmanager-0   2/2     Running   0          6h44m
prometheus-grafana-6fdb56fd8-n2c9v                       2/2     Running   0          16h
prometheus-kube-state-metrics-7dfdb8bb6f-7nfhh           1/1     Running   0          11h
prometheus-prometheus-node-exporter-2fj58                1/1     Running   0          2m8s
prometheus-prometheus-node-exporter-88knc                1/1     Running   0          21m
prometheus-prometheus-node-exporter-ht5gk                1/1     Running   0          38m
prometheus-prometheus-node-exporter-jhpjb                1/1     Running   0          3h41m
prometheus-prometheus-node-exporter-qxh9f                1/1     Running   0          2d22h
prometheus-prometheus-node-exporter-z695d                1/1     Running   0          6h54m
prometheus-prometheus-oper-operator-867867f497-8f2rf     2/2     Running   0          17h
prometheus-prometheus-prometheus-oper-prometheus-0       3/3     Running   1          3h3m
```


