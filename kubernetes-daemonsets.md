# Understanding Kubernetes DaemonSets

## What Is a DaemonSet?

A DaemonSet in Kubernetes is a controller that ensures that a copy of a Pod runs on all (or a subset of) nodes in a cluster. As nodes are added to the cluster, Pods are automatically added to them. As nodes are removed from the cluster, those Pods are garbage collected.

Think of a DaemonSet as a "node-level daemon" that provides node-specific functionality. It's similar to system daemons like `syslogd` or `journald` that run on every machine in a traditional system.

## Key Characteristics of DaemonSets

- **Node Coverage**: Runs one Pod on each node (or selected nodes)
- **Automatic Scheduling**: Automatically places Pods on new nodes as they join the cluster
- **Automatic Cleanup**: Removes Pods when nodes are drained or removed
- **Node Affinity**: Can target specific nodes using node selectors or affinity rules
- **Guaranteed Scheduling**: Bypasses normal scheduling constraints to ensure coverage
- **Update Strategies**: Supports rolling updates and OnDelete update strategies

## How DaemonSets Work

1. You define a DaemonSet with a Pod template and optional node selection criteria
2. The DaemonSet controller creates a Pod on each node that matches the selection criteria
3. When a new node joins the cluster, the DaemonSet automatically creates a Pod on that node
4. When a node is removed from the cluster, the corresponding Pod is garbage collected
5. Updates to the DaemonSet can be rolled out to all Pods using a rolling update strategy

## Anatomy of a DaemonSet

Here's a basic DaemonSet definition:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

Key components:
- **apiVersion/kind**: Specifies the API version and resource type
- **metadata**: Information about the DaemonSet (name, namespace, labels)
- **spec**: The desired state of the DaemonSet
  - **selector**: Labels used to identify which Pods belong to the DaemonSet
  - **updateStrategy**: How to update the DaemonSet (RollingUpdate or OnDelete)
  - **template**: Pod template used to create Pods
    - **metadata**: Pod metadata (labels)
    - **spec**: The Pod specification
      - **tolerations**: Allow Pods to be scheduled on nodes with matching taints
      - **containers**: List of containers to run
      - **volumes**: Volumes to mount in the Pod

## Node Selection in DaemonSets

DaemonSets can be configured to run on specific nodes using various selection mechanisms:

### 1. Running on All Nodes (Default)

By default, a DaemonSet runs on all nodes:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.3.1
```

### 2. Node Selector

Use `nodeSelector` to run on nodes with specific labels:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-driver-installer
spec:
  selector:
    matchLabels:
      name: gpu-driver-installer
  template:
    metadata:
      labels:
        name: gpu-driver-installer
    spec:
      nodeSelector:
        hardware: gpu
      containers:
      - name: installer
        image: gpu-driver-installer:latest
```

### 3. Node Affinity

Use `nodeAffinity` for more complex node selection:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
spec:
  selector:
    matchLabels:
      name: monitoring-agent
  template:
    metadata:
      labels:
        name: monitoring-agent
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: environment
                operator: In
                values:
                - production
              - key: disk-type
                operator: In
                values:
                - ssd
      containers:
      - name: agent
        image: monitoring-agent:v1.2
```

### 4. Taints and Tolerations

Use `tolerations` to allow scheduling on nodes with taints:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: critical-daemon
spec:
  selector:
    matchLabels:
      name: critical-daemon
  template:
    metadata:
      labels:
        name: critical-daemon
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: daemon
        image: critical-daemon:v1.0
```

## Update Strategies

DaemonSets support two update strategies:

### 1. RollingUpdate (Default)

Updates Pods one at a time in a controlled manner:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Maximum number of Pods that can be unavailable during the update
  # ... rest of the DaemonSet definition
```

### 2. OnDelete

Updates Pods only when they are manually deleted:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: legacy-daemon
spec:
  updateStrategy:
    type: OnDelete
  # ... rest of the DaemonSet definition
```

## Working with DaemonSets

### Creating a DaemonSet

```bash
# Using a YAML file
kubectl apply -f daemonset.yaml
```

### Viewing DaemonSets

```bash
# List all DaemonSets in the current namespace
kubectl get daemonsets

# List all DaemonSets in all namespaces
kubectl get daemonsets --all-namespaces

# Get detailed information about a DaemonSet
kubectl describe daemonset fluentd-elasticsearch -n kube-system

# View the Pods created by a DaemonSet
kubectl get pods -l name=fluentd-elasticsearch -n kube-system
```

### Updating a DaemonSet

```bash
# Update a DaemonSet
kubectl apply -f updated-daemonset.yaml

# Edit a DaemonSet directly
kubectl edit daemonset fluentd-elasticsearch -n kube-system

# Update the image of a DaemonSet
kubectl set image daemonset/fluentd-elasticsearch fluentd-elasticsearch=quay.io/fluentd_elasticsearch/fluentd:v2.6.0 -n kube-system
```

### Deleting a DaemonSet

```bash
# Delete a DaemonSet
kubectl delete daemonset fluentd-elasticsearch -n kube-system

# Delete a DaemonSet without deleting its Pods
kubectl delete daemonset fluentd-elasticsearch -n kube-system --cascade=orphan
```

## DaemonSet vs. Other Kubernetes Resources

### DaemonSet vs. Deployment

- A **Deployment** manages a set of identical Pods, typically running across multiple nodes
- A **DaemonSet** ensures that a single Pod runs on each node in the cluster

### DaemonSet vs. StatefulSet

- A **StatefulSet** manages stateful applications with stable network identities and persistent storage
- A **DaemonSet** manages node-level daemons that provide node-specific functionality

### DaemonSet vs. Job

- A **Job** creates Pods that run until completion and then terminate
- A **DaemonSet** creates Pods that run continuously on each node

## DaemonSet Use Cases

### 1. Cluster-Level Logging

Collect logs from every node:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

### 2. Node Monitoring

Run monitoring agents on every node:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.3.1
        ports:
        - containerPort: 9100
          hostPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

### 3. Storage Plugins

Install storage drivers on nodes:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: csi-node-driver
  namespace: storage
spec:
  selector:
    matchLabels:
      app: csi-node-driver
  template:
    metadata:
      labels:
        app: csi-node-driver
    spec:
      containers:
      - name: driver-registrar
        image: k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.3.0
      - name: csi-driver
        image: example.com/csi-driver:v1.0.0
        securityContext:
          privileged: true
        volumeMounts:
        - name: plugin-dir
          mountPath: /csi
        - name: registration-dir
          mountPath: /registration
      volumes:
      - name: plugin-dir
        hostPath:
          path: /var/lib/kubelet/plugins/csi-driver
          type: DirectoryOrCreate
      - name: registration-dir
        hostPath:
          path: /var/lib/kubelet/plugins_registry
          type: Directory
```

### 4. Network Plugins

Deploy network agents on each node:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      hostNetwork: true
      containers:
      - name: calico-node
        image: calico/node:v3.22.0
        env:
        - name: DATASTORE_TYPE
          value: "kubernetes"
        - name: CALICO_IPV4POOL_CIDR
          value: "192.168.0.0/16"
        securityContext:
          privileged: true
        volumeMounts:
        - name: lib-modules
          mountPath: /lib/modules
          readOnly: true
        - name: var-run-calico
          mountPath: /var/run/calico
        - name: var-lib-calico
          mountPath: /var/lib/calico
      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: var-run-calico
        hostPath:
          path: /var/run/calico
      - name: var-lib-calico
        hostPath:
          path: /var/lib/calico
```

### 5. Security Agents

Run security monitoring on all nodes:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: security
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      containers:
      - name: falco
        image: falcosecurity/falco:0.31.1
        securityContext:
          privileged: true
        volumeMounts:
        - name: dev-fs
          mountPath: /host/dev
          readOnly: true
        - name: proc-fs
          mountPath: /host/proc
          readOnly: true
        - name: boot-fs
          mountPath: /host/boot
          readOnly: true
        - name: lib-modules
          mountPath: /host/lib/modules
          readOnly: true
        - name: usr-fs
          mountPath: /host/usr
          readOnly: true
        - name: etc-fs
          mountPath: /host/etc
          readOnly: true
      volumes:
      - name: dev-fs
        hostPath:
          path: /dev
      - name: proc-fs
        hostPath:
          path: /proc
      - name: boot-fs
        hostPath:
          path: /boot
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: usr-fs
        hostPath:
          path: /usr
      - name: etc-fs
        hostPath:
          path: /etc
```

## Advanced DaemonSet Features

### Host Networking

Use the host's network namespace:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: host-network-daemon
spec:
  selector:
    matchLabels:
      name: host-network-daemon
  template:
    metadata:
      labels:
        name: host-network-daemon
    spec:
      hostNetwork: true  # Use the host's network namespace
      containers:
      - name: daemon
        image: example/daemon:v1.0
        ports:
        - containerPort: 8080
          hostPort: 8080  # Expose on the host's port 8080
```

### Host PID and IPC Namespaces

Access the host's process and IPC namespaces:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: system-daemon
spec:
  selector:
    matchLabels:
      name: system-daemon
  template:
    metadata:
      labels:
        name: system-daemon
    spec:
      hostPID: true   # Use the host's PID namespace
      hostIPC: true   # Use the host's IPC namespace
      containers:
      - name: daemon
        image: example/system-daemon:v1.0
        securityContext:
          privileged: true
```

### Scheduling with Critical Pod Annotation

Mark Pods as critical to prevent eviction:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: critical-daemon
spec:
  selector:
    matchLabels:
      name: critical-daemon
  template:
    metadata:
      labels:
        name: critical-daemon
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
    spec:
      priorityClassName: system-node-critical  # High priority
      containers:
      - name: daemon
        image: example/critical-daemon:v1.0
```

### Init Containers

Use init containers for setup tasks:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemon-with-init
spec:
  selector:
    matchLabels:
      name: daemon-with-init
  template:
    metadata:
      labels:
        name: daemon-with-init
    spec:
      initContainers:
      - name: init-daemon
        image: example/init-daemon:v1.0
        command: ["sh", "-c", "echo 'Initializing daemon...' && setup-daemon.sh"]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/daemon
      containers:
      - name: daemon
        image: example/daemon:v1.0
        volumeMounts:
        - name: config-volume
          mountPath: /etc/daemon
      volumes:
      - name: config-volume
        emptyDir: {}
```

## Best Practices for DaemonSets

1. **Resource Limits**: Set appropriate resource requests and limits for DaemonSet Pods
2. **Tolerations**: Use tolerations to ensure DaemonSets run on all required nodes, including control plane nodes if necessary
3. **Node Affinity**: Use node affinity for more complex node selection requirements
4. **Update Strategy**: Choose the appropriate update strategy based on your application requirements
5. **Security Context**: Configure security contexts to provide only the necessary privileges
6. **Health Checks**: Implement liveness and readiness probes for reliable operation
7. **Labels and Annotations**: Use meaningful labels and annotations for organization and management
8. **Host Resource Access**: Be cautious when mounting host paths or using host namespaces
9. **Minimal Images**: Use minimal container images to reduce resource usage and attack surface
10. **Monitoring**: Set up monitoring for DaemonSet Pods to ensure they're running correctly

## Real-World Examples

### Comprehensive Logging Agent

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: logging-agent
  namespace: logging
  labels:
    app: logging-agent
    component: logging
spec:
  selector:
    matchLabels:
      app: logging-agent
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: "10%"
  template:
    metadata:
      labels:
        app: logging-agent
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9103"
    spec:
      serviceAccountName: logging-agent
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.14-debian-elasticsearch7-1
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch-service"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_ELASTICSEARCH_SCHEME
          value: "http"
        - name: FLUENTD_SYSTEMD_CONF
          value: "disable"
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config-volume
          mountPath: /fluentd/etc/conf.d
        ports:
        - containerPort: 24224
          name: forward
          protocol: TCP
        - containerPort: 9103
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /metrics
            port: metrics
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /metrics
            port: metrics
          initialDelaySeconds: 5
          periodSeconds: 15
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config-volume
        configMap:
          name: fluentd-config
```

### Node Problem Detector

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-problem-detector
  namespace: kube-system
  labels:
    app: node-problem-detector
spec:
  selector:
    matchLabels:
      app: node-problem-detector
  template:
    metadata:
      labels:
        app: node-problem-detector
    spec:
      containers:
      - name: node-problem-detector
        image: k8s.gcr.io/node-problem-detector/node-problem-detector:v0.8.10
        securityContext:
          privileged: true
        resources:
          limits:
            cpu: 10m
            memory: 80Mi
          requests:
            cpu: 10m
            memory: 80Mi
        volumeMounts:
        - name: log
          mountPath: /var/log
          readOnly: true
        - name: kmsg
          mountPath: /dev/kmsg
          readOnly: true
      volumes:
      - name: log
        hostPath:
          path: /var/log
      - name: kmsg
        hostPath:
          path: /dev/kmsg
```

## Debugging DaemonSets

### Common Issues

1. **Pods Not Scheduled**: Check node selectors, taints, and tolerations
   ```bash
   kubectl describe daemonset my-daemonset
   kubectl get nodes --show-labels
   kubectl describe node <node-name>
   ```

2. **Pods Failing**: Check Pod logs and events
   ```bash
   kubectl logs -l app=my-daemonset
   kubectl describe pod -l app=my-daemonset
   ```

3. **Update Issues**: Check update strategy and Pod status
   ```bash
   kubectl rollout status daemonset/my-daemonset
   kubectl get pods -l app=my-daemonset
   ```

### Useful Commands

```bash
# Check DaemonSet status
kubectl get daemonset my-daemonset -o wide

# Check which nodes have DaemonSet Pods
kubectl get pods -l app=my-daemonset -o wide

# View DaemonSet events
kubectl describe daemonset my-daemonset

# Check logs from all DaemonSet Pods
kubectl logs -l app=my-daemonset --all-containers

# Restart a DaemonSet (by deleting all its Pods)
kubectl delete pods -l app=my-daemonset

# Pause a DaemonSet rollout
kubectl rollout pause daemonset/my-daemonset

# Resume a DaemonSet rollout
kubectl rollout resume daemonset/my-daemonset

# Roll back a DaemonSet update
kubectl rollout undo daemonset/my-daemonset
```

## Conclusion

DaemonSets are a powerful feature in Kubernetes for running node-level daemons across your cluster. They ensure that every node (or a subset of nodes) runs a copy of a Pod, making them ideal for infrastructure-level tasks like logging, monitoring, networking, and storage.

By understanding how to effectively use DaemonSets, you can implement reliable cluster-wide services that operate at the node level, ensuring consistent functionality across your entire Kubernetes infrastructure.