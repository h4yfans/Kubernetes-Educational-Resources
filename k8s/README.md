# Kubernetes Educational Resources

This directory contains a comprehensive collection of educational resources about Kubernetes concepts, components, and best practices. Each markdown file provides an in-depth explanation of a specific Kubernetes resource or concept, including definitions, examples, and practical usage patterns.

## Available Resources

### Core Kubernetes Resources
- [Namespaces](kubernetes-namespaces.md) - Logical partitions within a cluster
- [Pods](kubernetes-pods.md) - The smallest deployable units in Kubernetes
- [ReplicaSets](kubernetes-replicasets.md) - Ensures a specified number of pod replicas are running
- [Deployments](kubernetes-deployments.md) - Declarative updates for Pods and ReplicaSets
- [Services](kubernetes-services.md) - Network abstraction for pod access
- [ConfigMaps](kubernetes-configmaps.md) - External configuration storage
- [Secrets](kubernetes-secrets.md) - Sensitive information storage

### Workload Resources
- [Jobs](kubernetes-jobs.md) - Run-to-completion workloads
- [CronJobs](kubernetes-cronjobs.md) - Scheduled Jobs
- [DaemonSets](kubernetes-daemonsets.md) - Ensure pods run on all or specific nodes
- [StatefulSets](kubernetes-statefulsets.md) - Manage stateful applications

### Storage
- [PersistentVolumes and PersistentVolumeClaims](kubernetes-persistent-volumes.md) - Storage abstraction and management

### Networking and Ingress
- [Ingress](kubernetes-ingress.md) - HTTP/HTTPS routing to services
- [Gateway API](kubernetes-gateway-api.md) - Next-generation traffic routing

### CI/CD and DevOps
- [CI with GitHub Actions](kubernetes-ci-github-actions.md) - Continuous Integration for Kubernetes

## Resource Structure

Each resource guide typically includes:

1. **Definition and Purpose** - What the resource is and why it's used
2. **Key Characteristics** - Important features and capabilities
3. **How It Works** - Technical details on implementation
4. **YAML Examples** - Sample configurations
5. **Common Use Cases** - Practical applications
6. **Best Practices** - Recommendations for effective use
7. **Real-World Examples** - Practical implementation scenarios
8. **Troubleshooting Tips** - Common issues and solutions

## Recommended Learning Path

### For Beginners
If you're new to Kubernetes, we recommend studying these resources in order:

1. [Namespaces](kubernetes-namespaces.md)
2. [Pods](kubernetes-pods.md)
3. [ReplicaSets](kubernetes-replicasets.md)
4. [Deployments](kubernetes-deployments.md)
5. [Services](kubernetes-services.md)
6. [ConfigMaps](kubernetes-configmaps.md) and [Secrets](kubernetes-secrets.md)

### Intermediate Topics
After understanding the basics, proceed to:

1. [Jobs](kubernetes-jobs.md) and [CronJobs](kubernetes-cronjobs.md)
2. [DaemonSets](kubernetes-daemonsets.md)
3. [PersistentVolumes and PersistentVolumeClaims](kubernetes-persistent-volumes.md)
4. [Ingress](kubernetes-ingress.md)

### Advanced Topics
For more complex scenarios and advanced usage:

1. [StatefulSets](kubernetes-statefulsets.md)
2. [Gateway API](kubernetes-gateway-api.md)
3. [CI with GitHub Actions](kubernetes-ci-github-actions.md)

## Resource Relationships

Understanding how Kubernetes resources relate to each other is crucial:

- **Namespaces** provide isolation for all other resources
- **Pods** are managed by controllers like **ReplicaSets**, **Deployments**, **StatefulSets**, **DaemonSets**, **Jobs**, and **CronJobs**
- **Services** provide network access to **Pods**
- **Ingress** and **Gateway API** resources route external traffic to **Services**
- **ConfigMaps** and **Secrets** provide configuration to **Pods**
- **PersistentVolumes** and **PersistentVolumeClaims** provide storage to **Pods**

## Common Kubernetes Patterns

These guides cover several common patterns in Kubernetes:

1. **Stateless Applications** - Using Deployments and Services
2. **Stateful Applications** - Using StatefulSets with stable network identity and storage
3. **Background Processing** - Using Jobs and CronJobs
4. **Node-level Operations** - Using DaemonSets
5. **External Access** - Using Services, Ingress, and Gateway API
6. **Configuration Management** - Using ConfigMaps and Secrets
7. **Persistent Storage** - Using PersistentVolumes and PersistentVolumeClaims
8. **Continuous Integration** - Using GitHub Actions with Kubernetes