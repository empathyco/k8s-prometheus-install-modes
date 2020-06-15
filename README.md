
One of the most popular tools in the kubernetes ecosystem is [helm](https://helm.sh). Helm is a package manager for kubernetes applications. 
We will install the Prometheus Operator with it. For that we will use [**stable/prometheus-operator** helm chart](https://github.com/helm/charts/tree/master/stable/prometheus-operator).

The chart will install the following components:

- [prometheus-operator](https://github.com/coreos/prometheus-operator)
- [prometheus](https://prometheus.io/)
- [alertmanager](https://prometheus.io/)
- [node-exporter](https://github.com/helm/charts/tree/master/stable/prometheus-node-exporter)
- [kube-state-metrics](https://github.com/helm/charts/tree/master/stable/kube-state-metrics)
- [grafana](https://github.com/helm/charts/tree/master/stable/grafana)


