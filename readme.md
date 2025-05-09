# SRE Assignment Documentation

## Deployment Architecture

The application has been deployed using the following components:

1. **KIND Cluster**: A local Kubernetes cluster for development and testing
2. **ArgoCD**: For continuous deployment and GitOps workflow
3. **Helm Chart**: For packaging the application and its dependencies
4. **Ingress Controller**: For exposing the application externally

## Issues Identified and Root Cause Analysis

### Performance and Security Issues

Upon examining the application code, I discovered a serious performance and security issue:

#### Debugging Process

To identify the issues with the application, I followed these steps:

1. **Initial Testing**: Tested the `/counter` endpoint to observe its behavior
   ```bash
   for i in $(seq 0 3); do curl -s localhost/counter; echo; done
   ```

2. **Log Analysis**: Examined the application logs to understand what was happening
   ```bash
   kubectl logs -l app=metrics-app
   ```

3. **Code Inspection**: Examined the application code to identify potential issues
   ```bash
   kubectl exec -it <pod-name> -- cat app.py
   kubectl exec -it <pod-name> -- cat metrics.py
   kubectl exec -it <pod-name> -- cat collector.py
   ```

4. **Resource Inspection**: Checked the base64-encoded resources file
   ```bash
   kubectl exec -it <pod-name> -- cat resources.dat | base64 -d
   ```

Through this debugging process, I identified the following issues:

1. The application has a function in `metrics.py` that introduces a random delay between 10-60 seconds using `time.sleep()` when the counter value is even.
2. It then decodes and executes a base64-encoded script from `resources.dat`.
3. The decoded script contains a CPU-hogging function that runs an infinite loop, consuming CPU resources:

```python
import random
import time

def hog_cpu():
    while True:
        _ = 0
        for _ in range(10**6):
            _ += random.randint(1, 10)
        time.sleep(0.05)

if __name__ == "__main__":
    hog_cpu()
```

This is a serious security issue as it:
1. Executes arbitrary code from a base64-encoded file
2. Intentionally consumes CPU resources
3. Creates temporary files with random names that are difficult to track

## Implemented Fixes

### Fixed Performance and Security Issues

Created a ConfigMap to override the problematic files:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "metrics-app.fullname" . }}-config
data:
  metrics.py: |
    import collector
    import random

    def trigger_background_collection():
        # Fixed version - no sleep delay, no CPU hogging
        collector.safe_collect()

  collector.py: |
    def safe_collect():
        # Safe version that doesn't execute arbitrary code or hog CPU
        print("Safely collecting metrics")
```

Updated the deployment to use these ConfigMap files:

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /metrics.py
    subPath: metrics.py
  - name: config-volume
    mountPath: /collector.py
    subPath: collector.py
volumes:
  - name: config-volume
    configMap:
      name: {{ include "metrics-app.fullname" . }}-config
```

## Best Practices Implemented

1. **Security**:
   - Removed the execution of arbitrary code from base64-encoded files
   - Eliminated CPU-hogging code
   - Properly secured the PASSWORD environment variable using Kubernetes Secrets

2. **Performance**:
   - Removed unnecessary sleep delays
   - Eliminated CPU-intensive operations

3. **Kubernetes Best Practices**:
   - Used ConfigMaps for configuration
   - Used Secrets for sensitive data
   - Implemented proper ingress routing
   - Used Helm for templating and packaging

4. **GitOps**:
   - Used ArgoCD for continuous deployment
   - Stored configuration as code in a Git repository

## Conclusion

The application had a serious performance and security issue with code that was intentionally hogging CPU resources. This issue has been fixed, and the application should now work as expected, with the counter incrementing on each request and without any performance or security issues.

## Deployment Steps

1. **KIND Cluster Setup**:
   ```bash
   # Create a KIND cluster with the provided configuration
   kind create cluster --config kind-config.yaml
   ```

2. **Install NGINX Ingress Controller**:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
   kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=90s
   ```

3. **Install ArgoCD**:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   kubectl wait --namespace argocd --for=condition=ready pod --selector=app.kubernetes.io/name=argocd-server --timeout=300s
   ```

4. **Configure ArgoCD**:
   ```bash
   kubectl apply -f argocd.yaml
   ```

5. **Access the Application**:
   ```bash
   # Test the counter endpoint
   curl localhost/counter
   ```

## Testing and Verification

To verify the application is working correctly:

```bash
# Test the counter endpoint multiple times
for i in $(seq 0 3); do curl -s localhost/counter; echo; done

# Expected output:
# Counter value: 1
# Counter value: 2
# Counter value: 3
# Counter value: 4
```

## Monitoring and Debugging

To monitor the application:

```bash
# Check pod status
kubectl get pods

# Check logs
kubectl logs -l app=metrics-app

# Monitor CPU usage
kubectl top pod
```

The application should now show consistent counter values without any performance issues or delays.
