---
title: Monitoring (Prometheus)
permalink: /docs/prometheus/
description: How to deploy kuberentes cluster monitoring solution based on Prometheus. Installation based on Prometheus Operator using kube-prometheus-stack project.
last_modified_at: "03-08-2022"
---

Prometheus stack installation for kubernetes using Prometheus Operator can be streamlined using [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) project maintaned by the community.

That project collects Kubernetes manifests, Grafana dashboards, and Prometheus rules combined with documentation and scripts to provide easy to operate end-to-end Kubernetes cluster monitoring with Prometheus using the Prometheus Operator.

Components included in kube-stack package:

- The [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)
- Highly available [Prometheus](https://prometheus.io/)
- Highly available [Alertmanager](https://github.com/prometheus/alertmanager)
- Prometheus [node-exporter](https://github.com/prometheus/node_exporter) to collect metrics from each cluster node
- [Prometheus Adapter for Kubernetes Metrics APIs](https://github.com/kubernetes-sigs/prometheus-adapter)
- [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
- [Grafana](https://grafana.com/)

This stack is meant for cluster monitoring, so it is pre-configured to collect metrics from all Kubernetes components.

The architecture of components deployed is showed in the following image.

![kube-prometheus-stack](/assets/img/prometheus-stack-architecture.png)

## About Prometheus Operator

Prometheus operator is deployed and it manages Prometheus and AlertManager deployments and configuration through the use of Kubernetes CRD (Custom Resource Definitions):

- `Prometheus` and `AlertManager` CRDs: declaratively defines a desired Prometheus/AlertManager setup to run in a Kubernetes cluster. It provides options to configure the number of replicas and persistent storage. For each object of these types a set of StatefulSet is deployed.
- `ServiceMonitor`/`PodMonitor`/`Probe` CRDs: manages Prometheus service discovery configuration, defining how a dynamic set of services/pods/static-targets should be monitored.
- `PrometheusRules` CRD: defines alerting rules (prometheus rules)
- `AlertManagerConfig` CRD defines Alertmanager configuration, allowing routing of alerts to custom receivers, and setting inhibition rules. 

{{site.data.alerts.note}}

More details about Prometheus Operator CRDs can be found in [Prometheus Operator Design Documentation](https://prometheus-operator.dev/docs/operator/design/).

{{site.data.alerts.end}}

## Kube-Prometheus Stack installation

Kube-prometheus stack can be installed using helm [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) maintaind by the community

- Step 1: Add the Elastic repository

  ```shell
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  ```
- Step2: Fetch the latest charts from the repository

  ```shell
  helm repo update
  ```
- Step 3: Create namespace

  ```shell
  kubectl create namespace monitoring
  ```
- Step 3: Create values.yml for configuring VolumeClaimTemplates using longhorn and Grafana's admin password, list of plugins to be installed and disabling the monitoring of kubernetes components (Scheduler, Controller Manager and Proxy). See issue [#22](https://github.com/ricsanfre/pi-cluster/issues/22)

  ```yml
      alertmanager:
        alertmanagerSpec:
          storage:
            volumeClaimTemplate:
              spec:
                storageClassName: longhorn
                accessModes: ["ReadWriteOnce"]
                resources:
                  requests:
                    storage: 50Gi
      prometheus:
        prometheusSpec:
          storageSpec:
            volumeClaimTemplate:
              spec:
                storageClassName: longhorn
                accessModes: ["ReadWriteOnce"]
                resources:
                  requests:
                    storage: 50Gi
      grafana:
        # Admin user password
        adminPassword: "admin_password"
        # List of grafana plugins to be installed
        plugins:
          - grafana-piechart-panel
      kubeApiServer:
        enabled: true
      kubeControllerManager:
        enabled: false
      kubeScheduler:
        enabled: false
      kubeProxy:
        enabled: false
      kubeEtcd:
        enabled: false
  ```

- Step 4: Install kube-Prometheus-stack in the monitoring namespace with the overriden values

  ```shell
  helm install -f values.yml kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring
  ```

It creates the following Prometheus CRD

```yml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  annotations:
    meta.helm.sh/release-name: kube-prometheus-stack
    meta.helm.sh/release-namespace: k3s-monitoring
  creationTimestamp: "2022-08-01T16:31:56Z"
  generation: 1
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: kube-prometheus-stack
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 39.2.1
    chart: kube-prometheus-stack-39.2.1
    heritage: Helm
    release: kube-prometheus-stack
  name: kube-prometheus-stack-prometheus
  namespace: k3s-monitoring
  resourceVersion: "37885"
  uid: 9d915c45-690d-4459-b745-20f0fe8f5a9c
spec:
  alerting:
    alertmanagers:
    - apiVersion: v2
      name: kube-prometheus-stack-alertmanager
      namespace: k3s-monitoring
      pathPrefix: /
      port: http-web
  enableAdminAPI: false
  evaluationInterval: 30s
  externalUrl: http://kube-prometheus-stack-prometheus.k3s-monitoring:9090
  image: quay.io/prometheus/prometheus:v2.37.0
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  podMonitorNamespaceSelector: {}
  podMonitorSelector:
    matchLabels:
      release: kube-prometheus-stack
  portName: http-web
  probeNamespaceSelector: {}
  probeSelector:
    matchLabels:
      release: kube-prometheus-stack
  replicas: 1
  retention: 10d
  routePrefix: /
  ruleNamespaceSelector: {}
  ruleSelector:
    matchLabels:
      release: kube-prometheus-stack
  scrapeInterval: 30s
  securityContext:
    fsGroup: 2000
    runAsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: kube-prometheus-stack-prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
    matchLabels:
      release: kube-prometheus-stack
  shards: 1
  storage:
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi
        storageClassName: longhorn
  version: v2.37.0
```

Important configuration is:

-  Which CRDs (PodMonitor, ServiceMonitor and Probe) are applicable to this particular instance of Prometheus services:  `podMonitorSelector`, `serviceMonitorSelector`, and `probeSelector` introduces a defaul filtering rule (Objects must include a label `release: kube-prometheus-stack`)
- Default scrape interval to be applicable (`scrapeInterval`: 30sg). It can be overwitten in PodMonitor/ServiceMonitor/Probe particular configuration.



## Ingress resources configuration

Enable external access to Prometheus, Grafana and AlertManager through Ingress Controller

Create a Ingress rule to make prometheus stack front-ends available through the Ingress Controller (Traefik) using a specific URLs (`prometheus.picluster.ricsanfre.com` , `grafana.picluster.ricsanfre.com` and `alertmanager.picluster.ricsanfre.com`), mapped by DNS to Traefik Load Balancer service external IP.

prometheus, Grafana and alertmanager backend are not providing secure communications (HTTP traffic) and thus Ingress resource will be configured to enable HTTPS (Traefik TLS end-point) and redirect all HTTP traffic to HTTPS.
Since prometheus frontend does not provide any authentication mechanism, Traefik HTTP basic authentication will be configured.
- Step 1. Create a manifest file `prometheus_ingress.yml`

  Two Ingress resources will be created, one for HTTP and other for HTTPS. Traefik middlewares, HTTPS redirect and basic authentication will be used.
  
  ```yml
  ---
  # HTTPS Ingress
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: prometheus-ingress
    namespace: k3s-monitoring
    annotations:
      # HTTPS as entry point
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      # Enable TLS
      traefik.ingress.kubernetes.io/router.tls: "true"
      # Use Basic Auth Midleware configured
      traefik.ingress.kubernetes.io/router.middlewares: traefik-system-basic-auth@kubernetescrd
      # Enable cert-manager to create automatically the SSL certificate and store in Secret
      cert-manager.io/cluster-issuer: ca-issuer
      cert-manager.io/common-name: prometheus.picluster.ricsanfre.com
  spec:
    tls:
    - hosts:
      - prometheus.picluster.ricsanfre.com
      secretName: prometheus-tls
    rules:
    - host: prometheus.picluster.ricsanfre.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: kube-prometheus-stack-prometheus
              port:
                number: 9090
  ---
  # http ingress for http->https redirection
  kind: Ingress
  apiVersion: networking.k8s.io/v1
  metadata:
    name: prometheus-redirect
    namespace: k3s-monitoring
    annotations:
      # Use redirect Midleware configured
      traefik.ingress.kubernetes.io/router.middlewares: traefik-system-redirect@kubernetescrd
      # HTTP as entrypoint
      traefik.ingress.kubernetes.io/router.entrypoints: web
  spec:
    rules:
      - host: prometheus.picluster.ricsanfre.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-stack-prometheus
                port:
                  number: 9090
  ```
- Step 2. Create a manifest file `grafana_ingress.yml`

  Two Ingress resources will be created, one for HTTP and other for HTTPS. Traefik middlewares HTTPS redirect will be used.
  
  ```yml
  ---
  # HTTPS Ingress
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: grafana-ingress
    namespace: k3s-monitoring
    annotations:
      # HTTPS as entry point
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      # Enable TLS
      traefik.ingress.kubernetes.io/router.tls: "true"
      # Enable cert-manager to create automatically the SSL certificate and store in Secret
      cert-manager.io/cluster-issuer: ca-issuer
      cert-manager.io/common-name: grafana.picluster.ricsanfre.com
  spec:
    tls:
    - hosts:
      - grafana.picluster.ricsanfre.com
      secretName: grafana-tls
    rules:
    - host: grafana.picluster.ricsanfre.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: kube-prometheus-stack-grafana
              port:
                number: 80
  ---
  # http ingress for http->https redirection
  kind: Ingress
  apiVersion: networking.k8s.io/v1
  metadata:
    name: grafana-redirect
    namespace: k3s-monitoring
    annotations:
      # Use redirect Midleware configured
      traefik.ingress.kubernetes.io/router.middlewares: traefik-system-redirect@kubernetescrd
      # HTTP as entrypoint
      traefik.ingress.kubernetes.io/router.entrypoints: web
  spec:
    rules:
      - host: grafana.picluster.ricsanfre.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-stack-grafana
                port:
                  number: 80
  ```
- Step 3. Create a manifest file `alertmanager_ingress.yml`

  Two Ingress resources will be created, one for HTTP and other for HTTPS. Traefik middlewares HTTPS redirect will be used
  
  ```yml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: alertmanager-ingress
    namespace: k3s-monitoring
    annotations:
      # HTTPS as entry point
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      # Enable TLS
      traefik.ingress.kubernetes.io/router.tls: "true"
      # Use Basic Auth Midleware configured
      traefik.ingress.kubernetes.io/router.middlewares: traefik-system-basic-auth@kubernetescrd
      # Enable cert-manager to create automatically the SSL certificate and store in Secret
      cert-manager.io/cluster-issuer: ca-issuer
      cert-manager.io/common-name: alertmanager.picluster.ricsanfre.com
  spec:
    tls:
    - hosts:
      - alertmanager.picluster.ricsanfre.com
      secretName: prometheus-tls
    rules:
    - host: alertmanager.picluster.ricsanfre.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: kube-prometheus-stack-alertmanager
              port:
                number: 9093
  ---
  # http ingress for http->https redirection
  kind: Ingress
  apiVersion: networking.k8s.io/v1
  metadata:
    name: alertmanager-redirect
    namespace: k3s-monitoring
    annotations:
      # Use redirect Midleware configured
      traefik.ingress.kubernetes.io/router.middlewares: traefik-system-redirect@kubernetescrd
      # HTTP as entrypoint
      traefik.ingress.kubernetes.io/router.entrypoints: web
  spec:
    rules:
      - host: alertmanager.picluster.ricsanfre.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-stack-alertmanager
                port:
                  number: 9093
  ``` 
- Step 4. Apply the manifest file

  ```shell
  kubectl apply -f prometheus_ingress.yml grafana_ingress.yml alertmanager_ingress.yml
  ```

## K3S components monitoring

In order to monitor Kubernetes components (Scheduler, Controller Manager and Proxy), default resources created by kube-prometheus-operator (headless service, service monitor and grafana dashboards) are not valid for monitoring K3S because  K3S is emitting the same metrics on the three end-points, causing prometheus to consume high memory causing worker node outage. See issue [#22](https://github.com/ricsanfre/pi-cluster/issues/22) for more details.


- Create a manifest file `k3s-metrics-service.yml` for creating the Kuberentes service used by Prometheus to scrape K3S metrics.

  This service must be a [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services), for allowing Prometheus service discovery process of each of the pods behind the service. Since the metrics are exposed not by a pod but by a k3s process, the service need to be defined [`without selector`](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors) and the `endpoints` must be defined explicitly

  The service will be use the k3s-proxy endpoint (TCP port 10249) for scraping all metrics. 
  
  ```yml
  ---
  # Headless service for K3S metrics. No selector
  apiVersion: v1
  kind: Service
  metadata:
    name: k3s-metrics-service
    labels:
      app: k3s-metrics
    namespace: kube-system
  spec:
    clusterIP: None
    ports:
    - name: http-metrics
      port: 10249
      protocol: TCP
      targetPort: 10249
    type: ClusterIP
  ---
  # Endpoint for the headless service without selector
  apiVersion: v1
  kind: Endpoints
  metadata:
    name: k3s-metrics-service
    namespace: kube-system
  subsets:
  - addresses:
    - ip: 10.0.0.11
    ports:
    - name: http-metrics
      port: 10249
      protocol: TCP
  ```

- Create manifest file for defining the service monitor resource for let Prometheus discover this target

  The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover K3S metrics endpoint as a Prometheus target.

  ```yml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    labels:
      app: k3s
      release: kube-prometheus-stack
    name: k3s-prometheus-servicemonitor
    namespace: k3s-monitoring
  spec:
    namespaceSelector:
      matchNames:
      - kube-system
    selector:
      matchLabels:
        app: k3s-metrics
    endpoints:
      - port: http-metrics
        path: /metrics
  ```


- Apply manifest file
  ```shell
  kubectl apply -f k3s-metrics-service.yml k3s-servicemonitor.yml
  ```
- Check target is automatically discovered in Prometheus UI: `http://prometheus/targets`

### K3S Grafana dashboards

Kubernetes-controller-manager, kubernetes-proxy and kuberetes-scheduler dashboards can be donwloaded from [grafana.com](https://grafana.com):

- Kube Proxy: [dashboard-id 12129](https://grafana.com/grafana/dashboards/12129)
- Kube Controller Manager: [dashboard-id 12122](https://grafana.com/grafana/dashboards/12122)
- Kube Scheduler: [dashboard-id 12130](https://grafana.com/grafana/dashboards/12130)

## Traefik Monitoring

The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover Traefik metrics endpoint as a Prometheus target.

- Create a manifest file `traefik-servicemonitor.yml`

```yml
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: traefik
    release: kube-prometheus-stack
  name: traefik
  namespace: k3s-monitoring
spec:
  endpoints:
    - port: traefik
      path: /metrics
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      app.kubernetes.io/instance: traefik
      app.kubernetes.io/name: traefik-dashboard

``` 
{{site.data.alerts.important}}
Set `label.release` to the value specified for the helm release during Prometheus operator installation (`kube-prometheus-stack`).
{{site.data.alerts.end}}

- Apply manifest file
  ```shell
  kubectl apply -f traefik-servicemonitor.yml
  ```

- Check target is automatically discovered in Prometheus UI: `http://prometheus/targets`

### Traefik Grafana dashboard

Traefik dashboard can be donwloaded from [grafana.com](https://grafana.com): [dashboard id: 11462](https://grafana.com/grafana/dashboards/11462). This dashboard has as prerequisite to have installed `grafana-piechart-panel` plugin. The list of plugins to be installed can be specified during kube-prometheus-stack helm deployment as values (`grafana.plugins` variable).


## Longhorn Monitoring

As stated by official [documentation](https://longhorn.io/docs/1.2.2/monitoring/prometheus-and-grafana-setup/), Longhorn Backend service is a service pointing to the set of Longhorn manager pods. Longhorn’s metrics are exposed in Longhorn manager pods at the endpoint `http://LONGHORN_MANAGER_IP:PORT/metrics`

Backend endpoint is already exposing Prometheus metrics.

The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover Longhorn metrics endpoint as a Prometheus target.

- Create a manifest file `longhorm-servicemonitor.yml`
  
  ```yml
  ---
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    labels:
      app: longhorn
      release: kube-prometheus-stack
    name: longhorn-prometheus-servicemonitor
    namespace: k3s-monitoring
  spec:
    selector:
      matchLabels:
        app: longhorn-manager
    namespaceSelector:
      matchNames:
      - longhorn-system
    endpoints:
    - port: manager
  ``` 

{{site.data.alerts.important}}
Set `label.release` to the value specified for the helm release during Prometheus operator installation (`kube-prometheus-stack`).
{{site.data.alerts.end}}

- Apply manifest file

  ```shell
  kubectl apply -f longhorn-servicemonitor.yml
  ```

- Check target is automatically discovered in Prometheus UI:`http://prometheus/targets`


### Longhorn Grafana dashboard

Longhorn dashboard sample can be donwloaded from [grafana.com](https://grafana.com): [dashboard id: 13032](https://grafana.com/grafana/dashboards/13032).

## Velero Monitoring

By default velero helm chart is configured to expose Prometheus metrics in port 8085
Backend endpoint is already exposing Prometheus metrics.

It can be confirmed checking velero service

```shell
kubectl get svc velero -n velero-system -o yaml
```
```yml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: velero
    meta.helm.sh/release-namespace: velero-system
  creationTimestamp: "2021-12-31T11:36:39Z"
  labels:
    app.kubernetes.io/instance: velero
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: velero
    helm.sh/chart: velero-2.27.1
  name: velero
  namespace: velero-system
  resourceVersion: "9811"
  uid: 3a6707ba-0e0f-49c3-83fe-4f61645f6fd0
spec:
  clusterIP: 10.43.3.141
  clusterIPs:
  - 10.43.3.141
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http-monitoring
    port: 8085
    protocol: TCP
    targetPort: http-monitoring
  selector:
    app.kubernetes.io/instance: velero
    app.kubernetes.io/name: velero
    name: velero
  sessionAffinity: None
  type: ClusterIP
```
And executing `curl` command to obtain the velero metrics:

```shell
curl 10.43.3.141:8085/metrics
```

The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover Velero metrics endpoint as a Prometheus target.

- Create a manifest file `velero-servicemonitor.yml`
  
  ```yml
  ---
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    labels:
      app: velero
      release: kube-prometheus-stack
    name: velero-prometheus-servicemonitor
    namespace: k3s-monitoring
  spec:
    endpoints:
      - port: http-monitoring
        path: /metrics
    namespaceSelector:
      matchNames:
        - velero-system
    selector:
      matchLabels:
        app.kubernetes.io/instance: velero
        app.kubernetes.io/name: velero
  ``` 
{{site.data.alerts.important}}
Set `label.release` to the value specified for the helm release during Prometheus operator installation (`kube-prometheus-stack`).
{{site.data.alerts.end}}

- Apply manifest file
  ```shell
  kubectl apply -f longhorn-servicemonitor.yml
  ```

- Check target is automatically discovered in Prometheus UI

  http://prometheus.picluster.ricsanfre/targets


### Velero Grafana dashboard

Velero dashboard sample can be donwloaded from [grafana.com](https://grafana.com): [dashboard id: 11055](https://grafana.com/grafana/dashboards/11055).

## Minio Monitoring

For details see [Minio's documentation: "Collect MinIO Metrics Using Prometheus"](https://docs.min.io/minio/baremetal/monitoring/metrics-alerts/collect-minio-metrics-using-prometheus.html).

{{site.data.alerts.note}}
Minio Console Dashboard integration has not been configured, instead a Grafana dashboard is provided.
{{site.data.alerts.end}}

- Generate bearer token to be able to access to Minio Metrics

  ```shell
  mc admin prometheus generate <alias>
  ```
  
  Output is something like this:
  
  ```
  scrape_configs:
  - job_name: minio-job
  bearer_token: eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJleHAiOjQ3OTQ4Mjg4MTcsImlzcyI6InByb21ldGhldXMiLCJzdWIiOiJtaW5pb2FkbWluIn0.mPFKnj3p-sPflnvdrtrWawSZn3jTQUVw7VGxdBoEseZ3UvuAcbEKcT7tMtfAAqTjZ-dMzQEe1z2iBdbdqufgrA
  metrics_path: /minio/v2/metrics/cluster
  scheme: https
  static_configs:
  - targets: ['127.0.0.1:9091']
  ```

  Where: 
  - `bearer_token` is the token to be used by Prometheus for authentication purposes 
  - `metrics_path` is th path to scrape the metrics on Minio server (TCP port 9091)

- Create a manifest file `minio-metrics-service.yml` for creating the Kuberentes service pointing to a external server used by Prometheus to scrape Minio metrics.

  This service. as it happens with k3s-metrics must be a [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) and [without selector](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors) and the endpoints must be defined explicitly

  The service will be use the Minio endpoint (TCP port 9091) for scraping all metrics.
  ```yml
  ---
  # Headless service for Minio metrics. No Selector
  apiVersion: v1
  kind: Service
  metadata:
    name: minio-metrics-service
    labels:
      app: minio-metrics
    namespace: kube-system
  spec:
    clusterIP: None
    ports:
    - name: http-metrics
      port: 9091
      protocol: TCP
      targetPort: 9091
    type: ClusterIP
  ---
  # Endpoint for the headless service without selector
  apiVersion: v1
  kind: Endpoints
  metadata:
    name: minio-metrics-service
    namespace: kube-system
  subsets:
  - addresses:
    - ip: 10.0.0.11
    ports:
    - name: http-metrics
      port: 9091
    protocol: TCP
  ```
- Create manifest file for defining the a Secret containing the Bearer-Token an the service monitor resource for let Prometheus discover this target

  The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover Minio metrics endpoint as a Prometheus target.
  Bearer-token need to be b64 encoded within the Secret resource
  
  ```yml
  ---
  apiVersion: v1
  kind: Secret
  type: Opaque
  metadata:
    name: minio-monitor-token
    namespace: k3s-monitoring
  data:
    token: < minio_bearer_token | b64encode >
  ---
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    labels:
      app: minio
      release: kube-prometheus-stack
    name: minio-prometheus-servicemonitor
    namespace: k3s-monitoring
  spec:
    endpoints:
      - port: http-metrics
        path: /minio/v2/metrics/cluster
        scheme: https
        tlsConfig:
          insecureSkipVerify: true 
        bearerTokenSecret:
          name: minio-monitor-token
          key: token
    namespaceSelector:
      matchNames:
      - kube-system
    selector:
      matchLabels:
        app: minio-metrics
  ```
- Apply manifest file
  ```shell
  kubectl apply -f minio-metrics-service.yml minio-servicemonitor.yml
  ```
- Check target is automatically discovered in Prometheus UI: `http://prometheus/targets`

### Minio Grafana dashboard

Minio dashboard sample can be donwloaded from [grafana.com](https://grafana.com): [dashboard id: 13502](https://grafana.com/grafana/dashboards/13502).


## Elasticsearch Monitoring

[prometheus-elasticsearch-exporter](https://github.com/prometheus-community/elasticsearch_exporter) need to be installed in order to have Elastic search metrics in Prometheus format. See documentation ["Prometheus elasticsearh exporter installation"](/docs/elasticsearch/#prometheus-elasticsearh-exporter-installation).

This exporter exposes `/metrics` endpoint in port 9108.

The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover Fluentbit metrics endpoint as a Prometheus target.

- Create a manifest file `elasticsearch-servicemonitor.yml`
  
  ```yml
  ---
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    labels:
      app: prometheus-elasticsearch-exporter
      release: kube-prometheus-stack
    name: elasticsearch-prometheus-servicemonitor
    namespace: k3s-monitoring
  spec:
    endpoints:
      - port: http
        path: /metrics
    namespaceSelector:
      matchNames:
        - k3s-logging
    selector:
      matchLabels:
        app: prometheus-elasticsearch-exporter
  ```

### Elasticsearch Grafana dashboard

Elasticsearh exporter dashboard sample can be donwloaded from [prometheus-elasticsearh-grafana](https://github.com/prometheus-community/elasticsearch_exporter/blob/master/examples/grafana/dashboard.json).

## Fluentbit/Fluentd Monitoring

### Fluentbit Monitoring

Fluentbit, when enabling its HTTP server, it exposes several endpoints to perform monitoring tasks. See details in [Fluentbit monitoring doc](https://docs.fluentbit.io/manual/administration/monitoring).

One of the endpoints (`/api/v1/metrics/prometheus`) provides Fluentbit metrics in Prometheus format.

The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover Fluentbit metrics endpoint as a Prometheus target.

- Create a manifest file `fluentbit-servicemonitor.yml`
  
  ```yml
  ---
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    labels:
      app: fluent-bit
      release: kube-prometheus-stack
    name: fluentbit-prometheus-servicemonitor
    namespace: k3s-monitoring
  spec:
    endpoints:
      - path: /api/v1/metrics/prometheus
        targetPort: 2020
      - params:
          target:
          - http://127.0.0.1:2020/api/v1/storage
        path: /probe
        targetPort: 7979
    namespaceSelector:
      matchNames:
        - k3s-logging
    selector:
      matchLabels:
        app.kubernetes.io/instance: fluent-bit
        app.kubernetes.io/name: fluent-bit
  ```

Service monitoring include two endpoints. Fluentbit metrics endpoint (`/api/v1/metrics/prometheus` port TCP 2020) and json-exporter sidecar endpoint (`/probe` port 7979), passing as target parameter fluentbit storage endpoint (`api/v1/storage`)


### Fluentd Monitoring

In order to monitor Fluentd with Prometheus, `fluent-plugin-prometheus` plugin need to be installed and configured. The custom docker image [fluentd-aggregator](https://github.com/ricsanfre/fluentd-aggregator), I have developed for this project, has this plugin installed.

fluentd.conf file must include configuration of this plugin. It provides '/metrics' endpoint on port 24231.

```
# Prometheus metric exposed on 0.0.0.0:24231/metrics
<source>
  @type prometheus
  @id in_prometheus
  bind "#{ENV['FLUENTD_PROMETHEUS_BIND'] || '0.0.0.0'}"
  port "#{ENV['FLUENTD_PROMETHEUS_PORT'] || '24231'}"
  metrics_path "#{ENV['FLUENTD_PROMETHEUS_PATH'] || '/metrics'}"
</source>

<source>
  @type prometheus_output_monitor
  @id in_prometheus_output_monitor
</source>
```

Check out further details in [Fluentd Documentation: Monitoring by Prometheus] (https://docs.fluentd.org/monitoring-fluentd/monitoring-prometheus).

The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover Fluentd metrics endpoint as a Prometheus target.

- Create a manifest file `fluentd-servicemonitor.yml`
  
  ```yml
  ---
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    labels:
      app: fluentd
      release: kube-prometheus-stack
    name: fluentd-prometheus-servicemonitor
    namespace: k3s-monitoring
  spec:
    endpoints:
      - port: metrics
        path: /metrics
    namespaceSelector:
      matchNames:
        - k3s-logging
    selector:
      matchLabels:
        app.kubernetes.io/instance: fluentd
        app.kubernetes.io/name: fluentd
  ```


### Fluentbit/Fluentd Grafana dashboard

Fluentbit dashboard sample can be donwloaded from [grafana.com](https://grafana.com): [dashboard id: 7752](https://grafana.com/grafana/dashboards/7752).

This dashboard has been modified to include fluentbit's storage metrics (chunks up and down) and to solve some issues with fluentd metrics.


## External Nodes Monitoring

- Install Node metrics exporter

  Instead of installing Prometheus Node Exporter, fluentbit built-in similar functionallity can be used.

  Fluentbit's [node-exporter-metric](https://docs.fluentbit.io/manual/pipeline/inputs/node-exporter-metrics) and [prometheus-exporter](https://docs.fluentbit.io/manual/pipeline/outputs/prometheus-exporter) plugins can be configured to expose `gateway` metrics that can be scraped by Prometheus.

  Add to node's fluent.conf file the following configuration:

  ```
  [INPUT]
      name node_exporter_metrics
      tag node_metrics
      scrape_interval 30
  ```
  
  It configures node exporter input plugin to get node metrics

  ```
  [OUTPUT]
      name prometheus_exporter
      match node_metrics
      host 0.0.0.0
      port 9100
  ```

  It configures prometheuss output plugin to expose metrics endpoint `/metrics` in port 9100.

- Create a manifest file external-node-metrics-service.yml for creating the Kuberentes service pointing to a external server used by Prometheus to scrape External nodes metrics.

  This service. as it happens with k3s-metrics, and Minio must be a headless service and without selector and the endpoints must be defined explicitly.


  The service will be use the Fluentbit metrics endpoint (TCP port 9100) for scraping all metrics.

  ```yml
  ---
  # Headless service for External Node metrics. No Selector
  apiVersion: v1
  kind: Service
  metadata:
    name: external-node-metrics-service
    labels:
      app: prometheus-node-exporter
      release: kube-prometheus-stack
      jobLabel: node-exporter
    namespace: k3s-monitoring
  spec:
    clusterIP: None
    ports:
    - name: http-metrics
      port: 9100
      protocol: TCP
      targetPort: 9100
    type: ClusterIP
  ---
  # Endpoint for the headless service without selector
  apiVersion: v1
  kind: Endpoints
  metadata:
    name: external-node-metrics-servcie
    namespace: k3s-monitoring
  subsets:
  - addresses:
    - ip: 10.0.0.1
    ports:
    - name: http-metrics
      port: 9100
      protocol: TCP
  ```
  
  The service has been configured with specific labels so it matches the discovery rules configured in the Node-Exporter ServiceMonitoring Object (part of the kube-prometheus installation) and no new service monitoring need to be configured and the new nodes will appear in the corresponing Grafana dashboards.

  
      app: prometheus-node-exporter
      release: kube-prometheus-stack
      jobLabel: node-exporter


  Prometheus-Node-Exporter Service Monitor is the following:
  ```yml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    annotations:
      meta.helm.sh/release-name: kube-prometheus-stack
      meta.helm.sh/release-namespace: k3s-monitoring
    generation: 1
    labels:
      app: prometheus-node-exporter
      app.kubernetes.io/managed-by: Helm
      chart: prometheus-node-exporter-3.3.1
      heritage: Helm
      jobLabel: node-exporter
      release: kube-prometheus-stack
    name: kube-prometheus-stack-prometheus-node-exporter
    namespace: k3s-monitoring
    resourceVersion: "6369"
  spec:
    endpoints:
    - port: http-metrics
      scheme: http
    jobLabel: jobLabel
    selector:
      matchLabels:
        app: prometheus-node-exporter
        release: kube-prometheus-stack
  ```
  
  `spec.selector.matchLabels` configuration specifies which labels values must contain the services in order to be discovered by this ServiceMonitor object.
  ```yml
  app: prometheus-node-exporter
  release: kube-prometheus-stack
  ```

  `jobLabel` configuration specifies the name of a service label which contains the job_label assigned to all the metrics. That is why `jobLabel` label is added to the new service with the corresponding value (`node-exporter`). This jobLabel is used in all configured Grafana's dashboards, so it need to be configured to reuse them for the external nodes.
  ```yml
  jobLabel: node-exporter
  ```

- Apply manifest file
  ```shell
  kubectl apply -f exterlnal-node-metrics-service.yml
  ```
- Check target is automatically discovered in Prometheus UI: `http://prometheus/targets`

### Grafana dashboards

Not need to install additional dashboards. Node-exporter dashboards pre-integrated by kube-stack shows the external nodes metrics.

## Provisioning Dashboards automatically

Grafana dashboards can be provisioned automatically creating ConfigMap resources containing the dashboard json definition. For doing so, a provisioning sidecar container must be enabled.

Check grafana chart [documentation](https://github.com/grafana/helm-charts/tree/main/charts/grafana#sidecar-for-dashboards) explaining how to enable/use dashboard provisioning side-car.

`kube-prometheus-stack` configure by default grafana provisioning sidecar to check for new ConfigMaps containing label `grafana_dashboard`

This are the default helm chart values configuring the sidecar:

```yml
grafana:
  sidecar:
    dashboards:
      SCProvider: true
      annotations: {}
      defaultFolderName: null
      enabled: true
      folder: /tmp/dashboards
      folderAnnotation: null
      label: grafana_dashboard
      labelValue: null
```

For provision automatically a new dashboard, a new `ConfigMap` resource must be created, labeled with `grafana_dashboard: 1` and containing as `data` the json file content.

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-grafana-dashboard
  labels:
     grafana_dashboard: "1"
data:
  dashboard.json: |-
  [json_file_content]

```

{{site.data.alerts.important}}

Most of [Grafana community dashboards available](https://grafana.com/grafana/dashboards/) have been exported from a running Grafana and so they include a input  variable (`DS_PROMETHEUS`) which represent a datasource which is referenced in all dashboard panels (`${DS_PROMETHEUS}`). See details in [Grafana export/import documentation](https://grafana.com/docs/grafana/latest/dashboards/export-import/).

When automatic provisioning those exported dashboards following the procedure described above, an error appear when accessing them in the UI:

```
Datasource named ${DS_PROMETHEUS} was not found
```

There is an open [Grafana´s issue](https://github.com/grafana/grafana/issues/10786), asking for support of dasboard variables in dashboard provisioning.

As a workarround, json files can be modified before inserting them into ConfigMap yaml file, in order to detect DS_PROMETHEUS datasource. See issue [#18](https://github.com/ricsanfre/pi-cluster/issues/18) for more details

Modify each json file, containing `DS_PROMETHEUS` input variable within `__input` json key, adding the following code to `templating.list` key

```json
"templating": {
    "list": [
      {
        "hide": 0,
        "label": "datasource",
        "name": "DS_PROMETHEUS",
        "options": [],
        "query": "prometheus",
        "refresh": 1,
        "regex": "",
        "type": "datasource"
      },
    ...
```
{{site.data.alerts.end}}
