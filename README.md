
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


It is worth noting, the workload controller that it is used depending of the components, meanwhile the alert manager and the prometheus server are deployed as a [statefulset](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/), because they need data persistence, grafana runs as a [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), and it is store the configuration and dashboards in [configmaps](https://kubernetes.io/docs/concepts/configuration/configmap), the same happen with the operator. Finally the node exporter run as a daemon set to run in all **non master** nodes of the cluster to be able to expose metrics for each one of them. 

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

This installation could be fully modified by passing a configuration files as a parameter. To get the current configuration of our installation, just run.

```bash
$ helm get vaules prometheus > values.yaml

$ cat values.yaml

USER-SUPPLIED VALUES:
additionalPrometheusRules: []
alertmanager:
  alertmanagerSpec:
    additionalPeers: []
    affinity: {}
    configMaps: []
    containers: []
    externalUrl: null
    image:
      repository: quay.io/prometheus/alertmanager
      tag: v0.20.0
    listenLocal: false
    logFormat: logfmt
    logLevel: info
    nodeSelector: {}
    paused: false
    podAntiAffinity: ""
    podAntiAffinityTopologyKey: kubernetes.io/hostname
    podMetadata: {}
...
```

Some interesting parameter to change are:

- The configuration directives and the alert template of the alert manager.
- The labeling for the service monitor selector.
- The way to access the components, by default the charts create a _ClusterIP_ service that is only accessible inside the cluster, if you want to use a ingress to access the component it can be configured.
- The storage of each component, by default does NOT create a persistence volume.
- The credentials of each component.

## Monitoring your service.

One of good thing of using the prometheus operator is that we could define the rules, alerting and dashboards in the service definition, in our case a helm chart. 

Take this example, we have an application that has implemented a `/metrics` endpoint that we want to be scraped by our prometheus.

We have create a helm chart for it. 

```bash
$ helm create empathy-app

$ tree empathy-app
empathy-app
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

3 directories, 9 files
```
Then expose the metrics port in the `templates/deployment.yaml`

```diff
ports:
  - name: http
    containerPort: 80
    protocol: TCP
+ - name: metrics
+    containerPort: {{ .Values.service.metricsPort }}
+    protocol: TCP
```

### Service monitoring

To monitor our service we need to create a **service monitor** resource, to do so we will create a new file named `monitoring.yaml`, in the templates folder. 

```yaml
{{- if .Values.prometheus.enabled -}}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    {{- include "empathy-app.labels" . | nindent 4 }}
  {{- toYaml .Values.prometheus.annotations  | nindent 4}}
  name: {{ include "empathy-app.fullname" . }}
spec:
  endpoints:
  - honorLabels: true
    port: metrics
  selector:
    matchLabels:
      {{- include "empathy-app.selectorLabels" . | nindent 6 }}
  namespaceSelector:
    matchNames:
    - {{ .Release.Namespace }}
{{- end }}
```

This template will be filled with the parameters set in the `values.yaml`. 

```yaml
prometheus:
  enabled: true
  annotations:
    release: prometheus
    app: prometheus-operator
```

The template function `{{- include "empathy-app.selectorLabels" . | nindent 6 }}` will help us to **not** have misconfigured labels in the service monitor, as it will match always the service that we want to monitor.


### Alerts

To add alerts we will create a new template `templates/alerts.yaml`.

```yaml
{{- if .Values.prometheus.enabled -}}
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    {{- toYaml .Values.prometheus.annotations  | nindent 4}}
    {{- include "empathy-app.labels" . | nindent 4 }}
  name: {{ include "empathy-app.fullname" . }}
spec:
  groups:
  {{ range $path, $bytes := .Files.Glob "alerts/*.yml" -}}
  - name: {{ $path }}
    {{ tpl ($.Files.Get $path) $ | nindent 4 }}
  {{ end -}}
{{- end }}
```

This template will iterate through all files under the files `alerts/*.yaml` and create and prometheus rule that will be scraped by our prometheus by matching the labels that we have configured in the `values.yaml`. This file need to have a [prometheus alert](https://prometheus.io/docs/alerting/overview/).


```yaml
rules:
  - alert: "Empathy-{{ include "empathy-app.fullname" .}}-mem-usage"
    expr: jvm_memory_bytes_used{namespace="{{ $.Release.Namespace }}", service="{{ include "EMPATHY_TEMPLATE_CHART.fullname" . }}", area="heap"}/jvm_memory_bytes_max{namespace="{{ $.Release.Namespace }}", area="heap"} > 80
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Service - {{`{{$labels.service}}`}} user too much mem"
      description: "Alarm - {{`{{$labels.service}}`}} reports that is using more than 80% of mem ($value) Please check it"
```

Because how go template work (is the template engine used by helm), a small hack need to be added to

```text
{{$labels.service}} for {{`{{$labels.service}}`}} if you want to user variables
```


### Grafana dashboards

For the grafana dashboard we will do a similar approach as we did with the alerts.

We will create a template that iterate though all files under the folder dashboards, and create a configmap under the **namespace** `monitoring` with all dashboard configuration. The grafana installation  configured with helm before (almost the default installation) will check any creation or change every configmap that is in the `monitoring` namespace and has the label `grafana_dashboard: service_name` set.

```yaml
{{- if .Values.prometheus.enabled -}}
apiVersion: v1
data:
{{ (.Files.Glob "dashboards/*.json").AsConfig | indent 2 }}
kind: ConfigMap
metadata:
  namespace: monitoring
  annotations:
  labels:
    grafana_dashboard: {{ include "empathy-app.fullname" . }}
    {{- include "empathy-app.labels" . | nindent 4 }}
  name: {{ include "empathy-app.fullname" . }}-dashboard
{{- end }}
```

After all our **helm chart** should be something like that.

```tree
.
├── Chart.yaml
├── README.md
├── add_dashboard.sh
├── alerts
│   └── empathy-alerts.yml
├── charts
├── dashboards
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── alerts.yaml
│   ├── dashboard.yaml
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── monitoring.yaml
│   ├── secret.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

To have all service related monitoring inside the helm charts give us some strong adventages.


- it will allow us to version our dashboards and alerts correlated to the service version, avoiding to update different configuration every time a service is deploy.
- it will empower developer to create dashboard and alerts, as the helm chart can be place in service code repo. We want to specially emphasize this point, as is the developer who knows the service the best.

The constrain of this solution is that it will work only for services running in the same kubernetes cluster. 

Other point to mention is the possibility to configured the alert manager also with helm, it could be configured to send the alerts to the channel of the team owner of the service.
