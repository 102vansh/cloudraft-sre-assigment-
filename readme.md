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

To identify the issues with the application, I followed a systematic approach:

1. **Initial Testing**: Tested the `/counter` endpoint to observe its behavior
   ```bash
   # Test the endpoint multiple times to observe patterns
   for i in $(seq 0 3); do curl -s localhost/counter; echo; done

   # Test with timing information to measure response times
   for i in $(seq 0 3); do time curl -s localhost/counter; echo; done
   ```

2. **Log Analysis**: Examined the application logs to understand what was happening
   ```bash
   # Get general logs
   kubectl logs -l app=metrics-app

   # Filter logs for specific requests
   kubectl logs -l app=metrics-app | grep "GET /counter"

   # Look for patterns in timing between requests
   kubectl logs -l app=metrics-app | grep -A 5 -B 5 "GET /counter"
   ```

3. **Resource Monitoring**: Checked resource usage to identify abnormal patterns
   ```bash
   # Monitor CPU usage
   kubectl top pod

   # Check for processes inside the container
   kubectl exec -it <pod-name> -- ps aux

   # Monitor file system changes
   kubectl exec -it <pod-name> -- watch -n 1 "ls -la /tmp"
   ```

4. **Code Inspection**: Examined the application code to identify potential issues
   ```bash
   # Examine main application code
   kubectl exec -it <pod-name> -- cat app.py

   # Look for suspicious imports or functions
   kubectl exec -it <pod-name> -- grep -r "import" --include="*.py" .

   # Check specific files identified as suspicious
   kubectl exec -it <pod-name> -- cat metrics.py
   kubectl exec -it <pod-name> -- cat collector.py
   ```

5. **Deep Inspection**: Investigated suspicious components
   ```bash
   # Decode base64-encoded files
   kubectl exec -it <pod-name> -- cat resources.dat | base64 -d

   # Check for temporary files that might be created
   kubectl exec -it <pod-name> -- find /tmp -type f -mmin -60

   # Examine environment variables
   kubectl exec -it <pod-name> -- env | grep PASSWORD
   ```

6. **Network Analysis**: Checked for unexpected network activity
   ```bash
   # Check for open network connections
   kubectl exec -it <pod-name> -- netstat -tulpn

   # Monitor network traffic
   kubectl exec -it <pod-name> -- tcpdump -i any -n
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

#### Evidence of Issues

**Log Evidence:**
```
# Application logs showing requests to the /counter endpoint
10.244.0.7 - - [09/May/2025 10:49:28] "GET /counter HTTP/1.1" 200 -
10.244.0.7 - - [09/May/2025 10:50:09] "GET /counter HTTP/1.1" 200 -  # Note the 41-second gap between requests
```

**Performance Impact:**
The random delays between 10-60 seconds caused inconsistent response times, making the application unreliable. Additionally, the CPU-hogging function would cause resource contention, potentially affecting other applications running in the cluster.

## Implemented Fixes

### Solution Strategy

After identifying the performance and security issues, I developed a multi-faceted approach to fix the problems:

1. **Replace Malicious Code**: Create safe versions of the problematic files
2. **Use Kubernetes ConfigMaps**: Inject the fixed files into the container
3. **Mount as Individual Files**: Use subPath mounts to replace specific files without disturbing others
4. **Maintain Functionality**: Ensure the counter still works while removing harmful behavior

### Fixed Performance and Security Issues

#### 1. Created a ConfigMap with Safe Implementations

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

Key improvements in the fixed code:
- Removed the random sleep delay (10-60 seconds)
- Eliminated the execution of arbitrary code from base64-encoded files
- Prevented CPU-hogging infinite loops
- Maintained the expected interface so the application continues to work

#### 2. Updated the Deployment to Use the ConfigMap

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

Implementation details:
- Used `subPath` to mount individual files instead of entire directories
- This approach preserves other files in the container while replacing only the problematic ones
- The application continues to function normally but without the malicious behavior
- No container restart is required when updating the ConfigMap (though a restart will apply changes immediately)

#### 3. Testing the Fix

After applying the fix, I verified that:
- The counter endpoint responds immediately without delays
- CPU usage remains stable without spikes
- No suspicious temporary files are created
- The counter increments correctly on each request

## Resource Utilization Metrics

Before the fix, the application would:
- Introduce random delays between 10-60 seconds
- Spawn background processes that consumed CPU resources
- Create temporary files with random names

After the fix:
- No random delays in response times
- No background CPU-hogging processes
- No creation of temporary files
- Consistent and predictable performance

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

## Security Considerations for Future Prevention

To prevent similar issues in the future, the following measures should be implemented:

1. **Code Reviews**: Implement mandatory code reviews for all changes to the application code
2. **Static Analysis Tools**: Use tools like SonarQube or Snyk to scan code for security vulnerabilities
3. **Container Scanning**: Implement container image scanning to detect malicious code
4. **Resource Limits**: Set CPU and memory limits for all containers to prevent resource abuse
5. **Monitoring and Alerting**: Set up monitoring for unusual CPU usage patterns and alert on anomalies
6. **Principle of Least Privilege**: Ensure containers run with minimal required permissions

## Future Improvements

1. **Resource Management**:
   - Implement resource requests and limits for the container to ensure proper resource allocation
   ```yaml
   resources:
     requests:
       cpu: 100m
       memory: 128Mi
     limits:
       cpu: 200m
       memory: 256Mi
   ```

2. **Health Monitoring**:
   - Add liveness and readiness probes to detect application health issues
   ```yaml
   livenessProbe:
     httpGet:
       path: /
       port: http
     initialDelaySeconds: 10
     periodSeconds: 10
   readinessProbe:
     httpGet:
       path: /
       port: http
     initialDelaySeconds: 5
     periodSeconds: 5
   ```

3. **Metrics Collection**:
   - Implement Prometheus metrics for better observability
   - Set up Grafana dashboards for visualizing application performance

4. **Automated Testing**:
   - Implement load testing to detect performance issues before deployment
   - Add security scanning in the CI/CD pipeline

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

### Continuous Monitoring Strategy

For ongoing monitoring of the application, I recommend implementing the following strategy:

#### 1. Basic Kubernetes Monitoring

```bash
# Check pod status and health
kubectl get pods -o wide --watch

# Check detailed pod information
kubectl describe pod <pod-name>

# Check logs with timestamps
kubectl logs -l app=metrics-app --timestamps

# Monitor CPU and memory usage
kubectl top pod --containers
```

#### 2. Advanced Monitoring with Prometheus and Grafana

For production environments, implement a comprehensive monitoring solution:

1. **Install Prometheus Operator**:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm install prometheus prometheus-community/kube-prometheus-stack
   ```

2. **Configure ServiceMonitor for the application**:
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     name: metrics-app-monitor
     labels:
       release: prometheus
   spec:
     selector:
       matchLabels:
         app: metrics-app
     endpoints:
     - port: http
       path: /metrics
       interval: 15s
   ```

3. **Set up alerting rules**:
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: metrics-app-alerts
     labels:
       release: prometheus
   spec:
     groups:
     - name: metrics-app
       rules:
       - alert: HighCpuUsage
         expr: container_cpu_usage_seconds_total{pod=~"metrics-app.*"} > 0.8
         for: 5m
         labels:
           severity: warning
         annotations:
           summary: High CPU usage detected
           description: Pod {{ $labels.pod }} has high CPU usage
   ```

#### 3. Log Aggregation with ELK Stack

For centralized logging:

1. **Install ELK Stack**:
   ```bash
   helm repo add elastic https://helm.elastic.co
   helm install elasticsearch elastic/elasticsearch
   helm install kibana elastic/kibana
   helm install filebeat elastic/filebeat
   ```

2. **Configure Filebeat to collect application logs**:
   ```yaml
   filebeat.inputs:
   - type: container
     paths:
       - /var/log/containers/metrics-app-*.log
   ```

### Debugging Techniques for Future Issues

If similar issues arise in the future, use these debugging techniques:

1. **Performance Analysis**:
   ```bash
   # Check response time patterns
   for i in {1..10}; do time curl -s localhost/counter; sleep 1; done

   # Monitor file system activity
   kubectl exec -it <pod-name> -- strace -f -e trace=file -p 1
   ```

2. **Memory Analysis**:
   ```bash
   # Check memory usage patterns
   kubectl exec -it <pod-name> -- top -b -n 1

   # Look for memory leaks
   kubectl exec -it <pod-name> -- ps -o pid,rss,command ax
   ```

3. **Network Analysis**:
   ```bash
   # Check for unexpected connections
   kubectl exec -it <pod-name> -- netstat -tulpn

   # Monitor network traffic
   kubectl exec -it <pod-name> -- tcpdump -i any -n
   ```

## References

1. [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
2. [Helm Documentation](https://helm.sh/docs/)
3. [ArgoCD Documentation](https://argo-cd.readthedocs.io/en/stable/)
4. [KIND Documentation](https://kind.sigs.k8s.io/docs/user/quick-start/)
5. [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)

The application should now show consistent counter values without any performance issues or delays.
