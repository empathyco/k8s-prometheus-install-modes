
<!-- vim-markdown-toc GFM -->

* [Introduction](#introduction)
* [Installation modes](#installation-modes)
    * [Custom installation](#custom-installation)
    * [Prometheus Operator](#prometheus-operator)
    * [The kube-prometheus way](#the-kube-prometheus-way)
    * [Prometheus Operator Helm Charth](#prometheus-operator-helm-charth)
* [Common things](#common-things)
* [Summary](#summary)

<!-- vim-markdown-toc -->

# Introduction

Prometheus has become the facto open source solution for Kubernetes monitoring, with multiple ways to install and run it. Those ways can be classified, firstly, attending to the locality of installation and then to the grade of automation and effort for the overall process. 

For the external world, “cluster” is the term to refer to a production ready Kubernetes deployment, then the locality can be defined as the place of Prometheus’s components in regard to the cluster, naming the possibilities as internal and external. Of course, a mixing of both is possible, but without real advantages for the more automated ways of deployment.

The automation part, subject to the locality, includes multiple options, ranging from the need to build the infrastructure resources to run the monitoring components to the definition of the desired configuration values for then. This way, the process could require the launch of some virtual machines for running a high availability solution or set up a Helm Chart with the number of instances and the Docker image versions for a couple of Kubernetes’s StatefulSet.

What follows is an intent to describe and explore the above elements with the aim to help those that need to select an alternative to monitor Kubernetes. Firstly the constraint that locality impose on the installation modes is presented and then each mode is described, to continue with some common elements to all of them, like alerts definitions and Grafana’s dashboards. At the end a comparison is presented to summarise the principal elements of each alternative.


# Installation modes

As mentioned before there are multiple installation modes, from the launch of some virtual machines to provide a high available setup to a definition of a handful configuration values to specify the desired state of the final arrangement inside a Kubernetes cluster. These ways are:



*   Custom installation
*   Prometheus Operator
*   kube-prometheus
*   Prometheus Operator Hem Chart

These options are constrained by the desired locality of the Prometheus´s components regarding the cluster; with _locality_ defined as the place of each component in regards to the cluster, internal or external. The following table summarizes that.


<table>
  <tr>
   <td><strong>Installation Mode</strong>
   </td>
   <td><strong>Locality Constraint</strong>
   </td>
  </tr>
  <tr>
   <td>Custom installation
   </td>
   <td>None
   </td>
  </tr>
  <tr>
   <td>Prometheus Operator
   </td>
   <td>Internal
   </td>
  </tr>
  <tr>
   <td>kube-prometheus
   </td>
   <td>Internal
   </td>
  </tr>
  <tr>
   <td>Hem Chart
   </td>
   <td>Internal
   </td>
  </tr>
</table>



## Custom installation

Custom installation could be defined as the way for which a human operator takes the burden of defining all the resources and configurations required to install and run the monitoring solution. Among other things, the mains are:



*   Select and launch virtual machines
*   Install and run Prometheus components
    *   Prometheus
    *   Alertmanager
    *   Pushgateway
    *   Node Exporter
    *   cAdvisor (for containerized environments)
*   Install and run Grafana for visualizations
*   Setup for a high availability
*   Define the way to change and reload configurations (including Prometheus’s rules)
*   Upgrades of services and virtual machines

If the installation takes place outside Kubernetes all these elements need to be manually defined; otherwise, only the second is needed, requiring the definition of Kubernetes resources with any of the multiple ways that currently exists: plain YAML files, kustomize, jsonnet, Helm, etc. 

But if the former is discarded, why carry the burden of defining what possibly exists? Guided by the DRY principle, work repetition it’s not needed because the option is present; leading to the next section.


## Prometheus Operator

As the project itself established:


    “The Prometheus Operator provides Kubernetes native deployment and management of Prometheus and related monitoring components. The purpose of this project is to simplify and automate the configuration of a Prometheus based monitoring stack for Kubernetes clusters.”

With that goal the Operator brings a set of custom resources and ensures that the current state matches which the one specified by them. The custom resources are:



*   Prometheus
*   Alertmanager
*   ThanosRuler
*   ServiceMonitor
*   PodMonitor
*   PrometheusRule

The internal semantics and how it works can be found in the [introduction post](https://coreos.com/blog/the-prometheus-operator.html) about the Operator, but for the aim of the present what matter here is:



*   How to use it to deploy a production ready monitoring solution? and
*   What elements the Operator leaves out?

The [github](https://github.com/coreos/prometheus-operator) project has resources describing how to install the custom resources and examples of how to create the corresponding objects, but no enforcement regarding any tool to do that aside from plain text YAML definitions. Based on the solution nature itself is obvious that the locality of the installation must be internal to the cluster.

On the other side the Operator does not prescribe the use of node-exporter although it can be easily incorporated. The same applies to Grafana, without any set of predefined dashboards, and Prometheus rules are missing too.

Of course all this manual work could be done one time and use it thereafter. That is what the next mode brings.

## The kube-prometheus way

If the Operator represents a more convenient way than a custom installation [kube-prometheus](https://github.com/coreos/kube-prometheus) goes a step forward taking it as a starting point to built over and bring:



*   node-exporter and it’s related resources
*   Grafana with of ready to use set of dashboards
*   a predefined ensemble of Prometheus rules

A singular feature of this project is the language in which it is written -jsonnet- but also the possibility to use it as a library and combine with additional mixings or bundles, illustrated by the presence of the built in set of rules and dashboards, resulting from the inclusion of the [kubernetes-mixin](https://github.com/kubernetes-monitoring/kubernetes-mixin) project.

With a proper CI/CD pipeline and after the initial configuration, a human operator only needs to add ServiceMonitor’s objects, rules and dashboards; within the same project or as part of an external bundle, allowing for self contained projects that include the monitoring configuration alongside with the source code. The pipeline outcome is a set of kubernetes manifests generated by the library to be applied with kubectl.

As the previous option, this solution is constrained to be inside to the cluster, for the same reasons. 

The project documentation is self explanatory, with multiple examples to illustrate different configurations and arrangements.


## Prometheus Operator Helm Charth

One of the most popular tools in the kubernetes ecosystem is [helm](https://helm.sh). Helm is a package manager for kubernetes applications. His [hub](https://hub.helm.sh) offer a [prometheus helm chart](https://github.com/helm/charts/tree/master/stable/prometheus-operator).

The chart will install the following components:

- [prometheus-operator](https://github.com/coreos/prometheus-operator)
- [prometheus](https://prometheus.io/)
- [alertmanager](https://prometheus.io/)
- [node-exporter](https://github.com/helm/charts/tree/master/stable/prometheus-node-exporter)
- [kube-state-metrics](https://github.com/helm/charts/tree/master/stable/kube-state-metrics)
- [grafana](https://github.com/helm/charts/tree/master/stable/grafana)

and also will create service monitors to scrape the some internal kubernetes components like kube-apiserver, kube-scheduler, kube-controller-manager, etcd, kube-dns/coredns and kube-proxy.

It is worth noting, the workload controller that it is used depending of the components, meanwhile the alert manager and the prometheus server are deployed as a [statefulset](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/), because they need data persistence, grafana runs as a [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), and it is store the configuration and dashboards in [configmaps](https://kubernetes.io/docs/concepts/configuration/configmap), the same happen with the operator. Finally the node exporter run as a daemon set to run in all **non master** nodes of the cluster to be able to expose metrics for each one of them. 


Some interesting parameter to change are:

- The configuration directives and the alert template of the alert manager.
- The labeling for the service monitor selector.
- The way to access the components, by default the charts create a _ClusterIP_ service that is only accessible inside the cluster, if you want to use a ingress to access the component it can be configured.
- The storage of each component, by default does NOT create a persistence volume.
- The credentials of each component.

The strongest points of this solution is the installation process and the simplification of the upgrade process, plus having all advantages described in the prometheus operator section.

As constrain, this installation will only work for services monitored in the same cluster, having to configured some kind of federation like [thanos](https://thanos.io/) to solve this problem.

# Common things

Both kube-prometheus and the Prometheus Operator  Helm Chart came with a default set of ready to use Grafana’s dashboard and rules; but a custom installation or a direct use of the Prometheus Operator imposed the need to add both of them. However, it is possible to take advantage of the [kubernetes-mixin](https://github.com/kubernetes-monitoring/kubernetes-mixin) outcome and incorporate it to the solution workflow. 

This way although a certain amount of manual work is required for the initial setup, it's possible to end with a custom solution in some manner similar to the more advanced ones.

Anyway, it is up to the human operator to select one of the above alternatives. The next and final section present a summarized view of them.


# Summary

Four modes were presented previously to accomplish the deployment of a monitoring solution for Kubernetes with the Prometheus ecosystem. Those modes form a continuous, covering from a custom setup with a considerable amount of manual effort to a more advanced one that only requires setting some configuration options to establish the desired end state.

The following table presents each of them with a comparison based on the locality with regards to the cluster and the amount of manual effort to get it up and running.


<table>
  <tr>
   <td><strong>Installation Mode</strong>
   </td>
   <td><strong>Locality Constraint</strong>
   </td>
   <td><strong>Manual Effort</strong>
   </td>
  </tr>
  <tr>
   <td>Custom installation
   </td>
   <td><strong>None</strong>. 
<p>
The nature of the solution does not impose any constraint about locality. 
   </td>
   <td><strong>High.</strong>
<p>
All the required elements to run a complete solution should be defined manually, at least one time.
   </td>
  </tr>
  <tr>
   <td>Prometheus Operator
   </td>
   <td><strong>Internal.</strong>
<p>
The nature of the solution constraints the installation to be only internal to the cluster.
   </td>
   <td><strong>Medium.</strong>
<p>
All the custom resources objects need to be defined. Dashboards and rules need to be added manually or from a third party source.
   </td>
  </tr>
  <tr>
   <td>kube-prometheus
   </td>
   <td><strong>Internal.</strong>
<p>
Same as the Operator.
   </td>
   <td><strong>Low.</strong>
<p>
Impose a understanding of the jsonnet language to take advantage of the more advanced features of the solution. A default set of dashboard and rules come built in.
   </td>
  </tr>
  <tr>
   <td>Prometheus Operator  Helm Chart
   </td>
   <td><strong>Internal.</strong>
<p>
Same as the Operator.
   </td>
   <td><strong>Low.</strong>
<p>
Some basic knowledge of helm. Cluster monitoring out of the box.
    </td>
  </tr>
</table>


From the above is clear that kube-prometheus and Helm Char for Prometheus Operator are the more effortless solutions. But if the project constraints impose an installation which must be external to the cluster then a custom option is the only way; in that case at least the possibility of reuse the [kubernetes-mixin](https://github.com/kubernetes-monitoring/kubernetes-mixin) outcome is present.

Whatever the case, the option for a full stack monitoring solution for Kubernetes is there; the unthinkable is not to use it.
