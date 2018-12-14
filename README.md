# autoscaler-notes
## VPA
Follow https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler.
Things to change
* metrics server deployment customization:
```
--- a/deploy/1.8+/metrics-server-deployment.yaml
+++ b/deploy/1.8+/metrics-server-deployment.yaml
@@ -31,6 +31,10 @@ spec:
       - name: metrics-server
         image: k8s.gcr.io/metrics-server-amd64:v0.3.1
         imagePullPolicy: Always
+        command:
+        - /metrics-server
+        - --kubelet-insecure-tls
+        - --kubelet-preferred-address-types=InternalIP        
         volumeMounts:
         - name: tmp-dir
           mountPath: /tmp

```

## Cluster Autoscaler
```
-- a/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
+++ b/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
@@ -121,7 +121,7 @@ spec:
     spec:
       serviceAccountName: cluster-autoscaler
       containers:
-        - image: k8s.gcr.io/cluster-autoscaler:v1.2.2
+        - image: gcr.io/google-containers/cluster-autoscaler:v1.12.1
           name: cluster-autoscaler
           resources:
             limits:
@@ -137,7 +137,10 @@ spec:
             - --cloud-provider=aws
             - --skip-nodes-with-local-storage=false
             - --expander=least-waste
-            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
+            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/pending-pods
+          env:
+            - name: AWS_REGION
+              value: us-west-2
           volumeMounts:
             - name: ssl-certs
               mountPath: /etc/ssl/certs/ca-certificates.crt

```
