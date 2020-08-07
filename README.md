# Readme.md

## Installation of loki, prometheus-operator, grafana, ingress-nginx

For this we have use helmfile. Loki, prometheus-operator, grafana are installed in monitoring namespace whereas ingress-nginx has been installed in ingress-nginx namespace.

## Install helmfile
```
  1. url : https://github.com/roboll/helmfile/releases
  2. Download the package: helmfile_linux_amd64
  3. mv helmfile_linux_amd64 helmfile; chmod +x helmfile; sudo mv helmfile /usr/local/bin
```

Create a file called helmfile.yaml and enter the following

```
repositories:
  - name: loki 
    url: https://grafana.github.io/loki/charts
  - name: stable
    url: https://kubernetes-charts.storage.googleapis.com
  - name: ingress-nginx 
    url: https://kubernetes.github.io/ingress-nginx

releases:
  - name: loki
    namespace: monitoring
    chart: loki/loki-stack
    set:
    - name: persistence.enabled
      value: true
    - name: loki.persistence.size
      value: 40Gi

  - name: prometheus-operator
    namespace: monitoring 
    chart: stable/prometheus-operator
    version: 8.16.1
    set:
    - name: grafana.persistence.enabled
      value: true 
    - name: grafana.persistence.size
      value: 1Gi

  - name: ingress-nginx
    namespace: ingress-nginx 
    chart: ingress-nginx/ingress-nginx
    values:
    - "./ingress-nginx/values.yaml"
    set:
    - name: controller.metrics.enabled
      value: true
    - name: controller.metrics.serviceMonitor.enabled
      value: true
    - name: controller.service.externalTrafficPolicy
      value: Local
    - name: controller.metrics.serviceMonitor.namespace
      value: monitoring
    - name: controller.metrics.serviceMonitor.additionalLabels.release
      value: prometheus-operator 
```
We have to add two annotation in ingress-nginx therefore we have make changes in ingress-nginx/values.yaml file as below

```
  metrics:
    port: 10254
    # if this port is changed, change healthz-port: in extraArgs: accordingly
    enabled: false

    service:
-      annotations: []
+      annotations:
+        prometheus.io/scrape: "true"
+        prometheus.io/port: "9913"
```

Now run the following command which will install all of the above.

```
helmfile sync
```

## FluxCD

Flux is a continuous delivery tool that automates the deployment of containers to Kubernetes.  It fills the automation void that exists between building and monitoring. It is strightforward to install and easy to setup.  The most promising feature it delivers is that it allows teams to manage their Kubernetes deployments declaratively. Flux CD ensures that the Kubernetes cluster is always in sync with the configuration defined in the source code repository.


**Why to use FluxCD?**

- Enables continuous delivery of container images, using version control.
- Deployment is faster. As soon as the code is pushed it will get deployed. 
- It can be reverted if something goes wrong.
- Elimate the use of kubectl command

## Install fluxctl binary
```
  sudo snap install fluxctl
```
 
Create flux namespace

```
kubectl create ns flux
```

Add flux helm repo and update

```
helm repo add fluxcd https://charts.fluxcd.io
helm repo update
```

Export the github username
```
export GHUSER="github-username"
```

Install

```
fluxctl install \
--git-user=${GHUSER} \
--git-email=${GHUSER}@users.noreply.github.com \
--git-url=git@github.com:${GHUSER}/simpleapp \
--git-path=alertmanager-slack/rules \
--namespace=flux | kubectl apply -f -
```

* Make sure to change the github repo, --git-path where there are manifests file 

Get the SSH public key

```
fluxctl identity --k8s-fwd-ns flux
```

upload it in github --> simpleapp repo --> settings --> deploy keys

* Make sure to give write access

Install helm-operator

```
helm upgrade -i helm-operator fluxcd/helm-operator --namespace flux --set git.ssh.secretName=flux-git-deploy --set helm.versions=v3
```

## Simpleapp

Simpleapp has been installed using helmrelease.

Create a file called helmrelease.yaml and enter the following

```
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: simpleapp 
  namespace: default 
spec:
  releaseName: simpleapp
  chart:
    git: git@github.com:sarosejoshi/simpleapp.git 
    ref: master
    path: ./charts/simpleapp
```

Apply it

```
kubectl apply -f helmrelease.yaml
```

Now, we need to change some values in ***charts/simpleapp/values.yaml***

Set the image tag
```
-  tag: ""
+  tag: "v0.0.1"
```

Set the domain name

```
ingress:
-  enabled: false
+  enabled: true
-  annotations: {}
+  annotations: 
-  # kubernetes.io/ingress.class: nginx
+    kubernetes.io/ingress.class: nginx
  hosts:
-    - host: chart-example.local
+    - host: domain.com
-     paths: []
+      paths:
+        - /
+        - /*
```

Add everything and push it to git repo.

Get the loadbalancer

```
kubectl --namespace ingress-nginx get services -o wide -w ingress-nginx-controller
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE   SELECTOR
ingress-nginx-controller   LoadBalancer   10.245.244.17   64.225.86.144   80:30047/TCP,443:30812/TCP   9h    app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
```

Map domain.com to 64.225.86.144 in /etc/hosts

Open /etc/hosts file and paste the following

```
64.225.86.144 domain.com
```

Then browse http://domain.com/log/hello. You will get {"foo":"bar"}

To view it in loki we have to port-forward.

```
kubectl port-forward svc/prometheus-operator-grafana 8080:80 -n monitoring
```

Then browse http://localhost:8080

Get the password of grafana "admin" using the following command

```
kubectl get secret --namespace monitoring prometheus-operator-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

![](https://i.imgur.com/z5qFA8R.png)

Make sure to select data-source as loki.

## Prometheus-alerts

We will need to create a secret of alertmanager.yaml. First of all save the current secret

```
$ kubectl get secrets -n monitoring
```

NAME                                 TYPE            DATA   AGE
alertmanager-prometheus-operator-alertmanager       Opaque            1      14m

Save it to a file

```
kubectl -n monitoring get secret alertmanager-prometheus-operator-alertmanager -o yaml > alertmanager-secret-k8s.yaml
```

Save the following file as alertmanager.yaml. Make sure to change SLACK_URL_WEBHOOK and SLACK_CHANNEL

```
global:
 resolve_timeout: 5m
 slack_api_url: 'SLACK_URL_WEBHOOK'

route:
 group_by: [Alertname]
 receiver: 'slack-notifications'
 group_wait: 30s
 group_interval: 5m
 repeat_interval: 12h

receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#SLACK_CHANNEL'
    send_resolved: true
    icon_url: https://avatars3.githubusercontent.com/u/3380462
    title: |-
     [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }} for {{ .CommonLabels.job }}
     {{- if gt (len .CommonLabels) (len .GroupLabels) -}}
       {{" "}}(
       {{- with .CommonLabels.Remove .GroupLabels.Names }}
         {{- range $index, $label := .SortedPairs -}}
           {{ if $index }}, {{ end }}
           {{- $label.Name }}="{{ $label.Value -}}"
         {{- end }}
       {{- end -}}
       )
     {{- end }}
    text: >-
     {{ range .Alerts -}}
     *Alert:* {{ .Annotations.title }}{{ if .Labels.severity }} - `{{ .Labels.severity }}`{{ end }}

     *Description:* {{ .Annotations.description }}

     *Details:*
       {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
       {{ end }}
     {{ end }}
```

Now create the base64 of the above file:

```
cat alertmanager.yaml | base64 -w0
```

Open the alertmanager-secret-k8s.yaml file and paste the output of above to BASE64-ENCODED-STRINGS

```
apiVersion: v1
data:
  alertmanager.yaml: BASE64-ENCODED-STRINGS
kind: Secret
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus-operator
    meta.helm.sh/release-namespace: monitoring
  labels:
    app: prometheus-operator-alertmanager
    app.kubernetes.io/managed-by: Helm
    chart: prometheus-operator-8.14.0
    heritage: Helm
    release: prometheus-operator
  name: alertmanager-prometheus-operator-alertmanager
  namespace: monitoring
type: Opaque
```

Then apply it

```
kubectl apply -f alertmanager-secret-k8s.yaml
```

For alert rules we will create

imagepullfailure-alert.yaml  

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
 labels:
   app: prometheus-operator
   chart: prometheus-operator-8.16.1
   heritage: Helm
   release: prometheus-operator
 name: prometheus-operator-imagepullerr.rules
 namespace: monitoring
spec:
 groups:
 - name: imagepull-failure-rules
   rules:
   - alert: ImagePullBackOff
     annotations:
       title: 'Instance {{ $labels.instance }} Imagepullerror'
       description: 'Pod name {{$labels.pod}} and endpoint {{ $labels.instance }}  of namespace {{ $labels.namespace }} is unable to pull the image form the provided repository.'
     expr: kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"} > 0
     for: 1m
     labels:
       severity: warning
```

cpu-memory-alertrules.yaml 

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
 labels:
   app: prometheus-operator
   chart: prometheus-operator-8.16.1
   heritage: Helm
   release: prometheus-operator
 name: prometheus-operator-pod-cpu-memory-threshold.rules
 namespace: monitoring
spec:
 groups:
 - name: cpu-memory-pod-threshold-rules
   rules:
   - alert: memory-threshold
     annotations:
       title: 'Memory utilization greater than  80%' 
       description: 'Pod name {{$labels.pod}} and endpoint {{ $labels.instance }}  of namespace {{ $labels.namespace }} Memory utilization is greater than  80%.'
     expr:  sum (container_memory_usage_bytes{container_name!="POD",container_name!="",container_name="simpleapp",namespace="default"}) by ( pod ) / sum (kube_pod_container_resource_limits_memory_bytes{container="simpleapp",namespace="default" }) by ( pod )  *100 > 80 
     for: 2m
     labels:
       severity: warning
   - alert: cpu-threshold
     annotations:
       title: 'CPU utilization greater than  80% '
       description: 'Pod name {{$labels.pod}} and endpoint {{ $labels.instance }}  of namespace {{ $labels.namespace }} CPU utilization is greater than  80%.'
     expr: sum(rate(container_cpu_usage_seconds_total{container_name!="POD",container_name!="",container_name="simpleapp",namespace="default"}[2m]) ) by (pod) / sum (kube_pod_container_resource_limits_cpu_cores{container="simpleapp",namespace="default"} ) by (pod) *100 > 80
     for: 2m
     labels:
       severity: warning
```

node-cpu-disk-memory-alert-rule.yaml

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
 labels:
   app: prometheus-operator
   chart: prometheus-operator-8.16.1
   heritage: Helm
   release: prometheus-operator
 name: prometheus-operator-node-cpu-memory-threshold.rules
 namespace: monitoring
spec:
 groups:
 - name: cpu-memory-disk-threshold-rules
   rules:
   - alert: node-memory threshold
     annotations:
       title: 'Nodes Memory utilization greater than 80%' 
       description: 'Node name {{$labels.node}} has reached Memory utilization is greater than  80%.'
     expr: sum(container_memory_usage_bytes{id="/"})  by (node) / sum(kube_node_status_capacity_memory_bytes) by (node) *100 > 80
     for: 2m
     labels:
       severity: warning
   - alert: nodecpu-threshold
     annotations:
       title: 'Nodes CPU utilization greater than  80%'
       description: 'Node name {{$labels.node}} has reached CPU utilization is greater than  80%.'
     expr: sum (rate (container_cpu_usage_seconds_total{id="/"}[1m])) by (node) / sum (machine_cpu_cores) by (node)  *100 > 80
     for: 2m
     labels:
       severity: warning
   - alert: node-disk-threshold
     annotations:
       title: 'Nodes Disk Space utilization greater than  80%'
       description: 'Node name {{$labels.node}} has reached disk utilization is greater than  80%.'
     expr:  sum ( sum (node_filesystem_size_bytes) by (instance) - sum (node_filesystem_avail_bytes) by (instance)  ) by (instance)  / sum (node_filesystem_size_bytes) by (instance) *100 > 80
     for: 2m
     labels:
       severity: warning
```

pod-deployment-failure-alert.yaml

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
 labels:
   app: prometheus-operator
   chart: prometheus-operator-8.16.1
   heritage: Helm
   release: prometheus-operator
 name: prometheus-operator-deployment-failure.rules
 namespace: monitoring
spec:
 groups:
 - name: pod-deployment-failure-rules
   rules:
   - alert: PodFailure-DeploymentFailure
     annotations:
       title: 'Deployment failed'
       description: 'Deployment failed. Check the informations on the Details Section'
     expr: kube_pod_container_status_waiting_reason{reason=~"CrashLoopBackOff|CreateContainerConfigError|CreateContainerError"} > 0
     for: 1m
     labels:
       severity: warning
```

status-check.yaml

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
 labels:
   app: prometheus-operator
   chart: prometheus-operator-8.16.1
   heritage: Helm
   release: prometheus-operator
 name: prometheus-operator-status-check.rules
 namespace: monitoring
spec:
 groups:
 - name: status-check
   rules:
   - alert: ErrorStatusCheck
     annotations:
       title: '4xx/5xx Error'
       description: ''
     expr:   sum (rate(nginx_ingress_controller_request_duration_seconds_count{ingress =~ "simpleapp",status =~"[4-5].*",}[1m])) by(path, status) > 0
     for: 1m
     labels:
       severity: warning
```
Apply it

```
kubectl apply -f imagepullfailure-alert.yaml
kubectl apply -f cpu-memory-alertrules.yaml
kubectl apply -f node-cpu-disk-memory-alert-rule.yaml
kubectl apply -f pod-deployment-failure-alert.yaml
kubectl apply -f status-check.yaml 
```
