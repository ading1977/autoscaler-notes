# HPA with microservices based application
This benchmarking is to test the performance of upstream [`Horizontal Pod Autoscaler`](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) against the microservices based application [`turbonomic demoapp`](https://github.com/turbonomic/demoapp/). Certain modifications to the original [`deploy instructions`](https://github.com/turbonomic/demoapp/tree/master/deploy) are required to make upstream Horizontal Pod Autoscaler with custom metrics work.
## Download and deploy Istio 
* Follow the [`instructions`]( https://istio.io/docs/setup/kubernetes/quick-start/) to install Istio.
* If you have Helm/Tiller enabled, you can install with [`helm`](https://istio.io/docs/setup/kubernetes/helm-install/). For exampe, you can use the following command to install Istio in an AKS cluster:
```
  helm install install/kubernetes/helm/istio --name istio --namespace istio-system \
    --set global.controlPlaneSecurityEnabled=true \
    --set grafana.enabled=true \
    --set tracing.enabled=true \
    --set kiali.enabled=true
```
* Do **NOT** enable automatic injection of istio-proxy in any namespace.

## Deploy Cassandra cluster, and Cassandra exporter for Prometheus
* Follow Step 2 in the original [`deploy instructions`](https://github.com/turbonomic/demoapp/tree/master/deploy).

## Deploy Twitter app
* Use the [`yaml file`](./app/deploy-with-istio.yaml) to deploy the twitter application with container pods injected with istio-proxy.
* Use the [`yaml file`](./app/demoapp-gateway.yaml) to configure a gateway and virtualservice such that client traffic to the twitter application can be load balanced between multiple twitter-cass-api instances.
* The external IP for the twitter application can be obtained by running:
  ```
  meng@Mengs-MBP$ kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
  40.76.50.131
  ```
  The port is default HTTP port 80.

## Update the prometheus configuration
Run `kubectl -n istio-system edit configmap prometheus`
* Add `cassandra-nodes` target 
```
    - job_name: 'cassandra-nodes'
      static_configs:
      - targets: ['10.244.3.17:8080','10.244.1.23:8080']
```

* Update the `kubernetes-apiservers` target to replace the aipserver address with a DNS name (i.e., `kubernetes.default.svc`)
```
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - default
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: kubernetes;https
      - target_label: __address__
        replacement: kubernetes.default.svc:443
```

## Configure custom metrics to be collected by Envoy, Istio and Prometheus
* Deploy the [`yaml file`](./metrics/ip.turbo.metric.yaml). Compared with the original [`yaml file`](https://github.com/turbonomic/demoapp/blob/master/deploy/metrics/ip.turbo.metric.yaml), this file adds a few dimensions to the metric which is needed for the `k8s-prometheus-adapter` label matching rule to work for custom metrics.

  **turbopodlatency**
  ```
    destination_name: destination.name | "unknown"
    destination_namespace: destination.namespace | "unknown"
  ```
  **turbosvclatency**
  ```
    destination_service_name: destination.service.name | "unknown"
    destination_service_namespace: destination.service.namespace | "unknown"
  ```
## Launch Locust test to simulate user workload
* Deploy the Locust cluster on a seperate kubernetes cluster (preferrably in a different geographic region) with the [`sample yaml files`](./locust) after configuring the IP fields inside the file. For example:
```
meng@Mengs-MBP$ kubectl --kubeconfig='/Users/meng/.kube/kubeconfig-aws' get po -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP           NODE                                     
locust-master-b9f69c9fc-crx7w    1/1     Running   0          8m25s   10.2.1.82    ip-172-23-1-89.us-west-2.compute.internal
locust-worker-57947bd484-dgpbr   1/1     Running   0          7m2s    10.2.2.123   ip-172-23-1-79.us-west-2.compute.internal
locust-worker-57947bd484-mpf52   1/1     Running   0          8m25s   10.2.1.84    ip-172-23-1-89.us-west-2.compute.internal
locust-worker-57947bd484-qsxpc   1/1     Running   0          7m2s    10.2.2.122   ip-172-23-1-79.us-west-2.compute.internal
locust-worker-57947bd484-zvwt7   1/1     Running   0          8m25s   10.2.1.83    ip-172-23-1-89.us-west-2.compute.internal
```

## Configure CPU based HPA
CPU utilization based HPA can be enabled relatively easily
* Install [`metric server`](https://github.com/kubernetes-incubator/metrics-server). Heapster based metric collection has been deprecated.
* Run the following command to enable HPA:
  ```
  kubectl autoscale deployment twitter-cass-api --max=10 --cpu-percent=75
  kubectl autoscale deployment twitter-cass-tweet --max=10 --cpu-percent=75
  kubectl autoscale deployment twitter-cass-user --max=10 --cpu-percent=75
  ```
## Configure custom metrics based HPA
Kubernetes Horizontal Pod Autoscaler retrieves custom metrics from `custom.metrics.k8s.io` API. It’s provided by “adapter” API servers provided by metrics solution vendors. The most popular opensource adapter API server solution for promethus is [`k8s-prometheus-adapter`](https://github.com/directxman12/k8s-prometheus-adapter). It acts as a delegation between kubernetes and prometheus. Queries made to the kubernetes `custom.metrics.k8s.io` API endpoints are directed to this adapter server, where metrics values are dynamically obtained from prometheus server.
* Create `custom-metrics` namespace
  ```
  kubectl create namespace custom-metrics
  ```
* Deploy `k8s-promethues-adapter` without TLS enabled
  ```
  kubectl create -f k8s-promethues-adapter/manifests/
  ```
  
  Verify the custom resources are registered:
  ```
  meng@Mengs-MBP$ kubectl get --raw '/apis/custom.metrics.k8s.io/v1beta1' | jq .
  {
    "kind": "APIResourceList",
    "apiVersion": "v1",
    "groupVersion": "custom.metrics.k8s.io/v1beta1",
    "resources": [
      {
        "name": "namespaces/average-response-time",
        "singularName": "",
        "namespaced": false,
        "kind": "MetricValueList",
        "verbs": [
          "get"
        ]
      },
      {
        "name": "services/average-response-time",
        "singularName": "",
        "namespaced": true,
        "kind": "MetricValueList",
        "verbs": [
          "get"
        ]
      }
    ]
  }
  ```
  
  You can retrieve the resource by doing:
  ```
  meng@Mengs-MBP$ kubectl get --raw '/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/service/twitter-cass-api/average-response-time' | jq .items[].value
  "1236m"
  ```
  
  
* Create hpa based on custom metrics:
  ```
  kubectl create -f hpa/
  ```
* View hpa in action
  ```
  meng@Mengs-MBP$ kubectl get hpa
  NAME                 REFERENCE                       TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
  istio-pilot          Deployment/istio-pilot          <unknown>/80%   1         5         0          110d
  twitter-cass-api     Deployment/twitter-cass-api     265m/300m       1         15        13         21h
  twitter-cass-tweet   Deployment/twitter-cass-tweet   244m/300m       1         15        5          21h
  twitter-cass-user    Deployment/twitter-cass-user    11m/300m        1         10        1          20h
  ```
* Reference: Changes made to the original deployment yaml:
  - Remove TLS
  - Allow generation of certificates into /tmp directory
  - Modify prometheus server URL
  - Change log level to 6
  ```
  @@ -22,28 +22,23 @@ spec:
           image: directxman12/k8s-prometheus-adapter-amd64
           args:
           - --secure-port=6443
  -        - --tls-cert-file=/var/run/serving-cert/serving.crt
  -        - --tls-private-key-file=/var/run/serving-cert/serving.key
  +        - --cert-dir=/tmp/cert
           - --logtostderr=true
  -        - --prometheus-url=http://prometheus.prom.svc:9090/
  +        - --prometheus-url=http://prometheus.istio-system.svc:9090/
           - --metrics-relist-interval=1m
  -        - --v=10
  +        - --v=6
           - --config=/etc/adapter/config.yaml
           ports:
           - containerPort: 6443
  +        securityContext:
  +          readOnlyRootFilesystem: true
           volumeMounts:
  -        - mountPath: /var/run/serving-cert
  -          name: volume-serving-cert
  -          readOnly: true
           - mountPath: /etc/adapter/
             name: config
             readOnly: true
           - mountPath: /tmp
             name: tmp-vol
         volumes:
  -      - name: volume-serving-cert
  -        secret:
  -          secretName: cm-adapter-serving-certs
         - name: config
           configMap:
             name: adapter-config
   ```
 * Reference: The [`configuration`](./k8s-prometheus-adapter/manifests/custom-metrics-config-map-rt.yaml) to retrieve the average response time custom metrics.

 * Reference: Sample hpa configuration for [`twitter-cass-api`](./hpa/twitter-cass-api-hpa.yaml)
   ```
    apiVersion: autoscaling/v2beta1
    kind: HorizontalPodAutoscaler
    metadata:
      creationTimestamp: "2018-12-23T16:44:53Z"
      name: twitter-cass-api
      namespace: default
      resourceVersion: "25993995"
      selfLink: /apis/autoscaling/v2beta1/namespaces/default/horizontalpodautoscalers/twitter-cass-api
      uid: 128b7be8-06d2-11e9-94c0-0a58ac1f1fb8
    spec:
      minReplicas: 1
      maxReplicas: 10
      metrics:
      - type: Object
        object:
          target:
            kind: Service
            name: twitter-cass-api
          metricName: average-response-time
          targetValue: 300m
      scaleTargetRef:
        apiVersion: extensions/v1beta1
        kind: Deployment
        name: twitter-cass-api
     ```
 
 
 
 
 
