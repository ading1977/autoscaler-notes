# autoscaler-notes
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
