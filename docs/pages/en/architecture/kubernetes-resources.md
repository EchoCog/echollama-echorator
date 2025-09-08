# Kubernetes Resources Architecture

This document details the Kubernetes resources created and managed by the EchoLlama-Echorator operator.

## Custom Resource Definitions (CRDs)

### Model CRD

The Model CRD is the primary interface for users to define and manage Ollama model deployments.

```mermaid
graph TD
    subgraph "Model CRD Structure"
        MODEL[Model Resource] --> SPEC[spec]
        MODEL --> STATUS[status]
        MODEL --> METADATA[metadata]
        
        SPEC --> REPLICAS[replicas: int32]
        SPEC --> IMAGE[image: string]
        SPEC --> PULL_POLICY[imagePullPolicy: string]
        SPEC --> STORAGE_CLASS[storageClassName: string]
        SPEC --> PVC_NAME[persistentVolumeClaim: string]
        SPEC --> PV_SPEC[persistentVolume: PVSpec]
        
        PV_SPEC --> ACCESS_MODE[accessMode: AccessMode]
        
        STATUS --> CONDITIONS[conditions: []Condition]
        STATUS --> READY_REPLICAS[readyReplicas: int32]
        STATUS --> AVAILABLE_REPLICAS[availableReplicas: int32]
        STATUS --> PHASE[phase: string]
        STATUS --> MESSAGE[message: string]
    end
```

### Model Resource Relationships

```mermaid
graph LR
    subgraph "User Input"
        MODEL_YAML[model.yaml]
    end
    
    subgraph "Custom Resources"
        MODEL_CR[Model CR]
    end
    
    subgraph "Storage Resources"
        PVC[PersistentVolumeClaim]
        PV[PersistentVolume]
        SC[StorageClass]
        STATEFUL[StatefulSet<br/>Image Storage]
    end
    
    subgraph "Compute Resources"
        DEPLOY[Deployment<br/>Model Inference]
        RS[ReplicaSet]
        PODS[Pods]
    end
    
    subgraph "Network Resources"
        SVC[Service]
        ENDPOINTS[Endpoints]
        INGRESS[Ingress<br/>Optional]
    end
    
    subgraph "Configuration"
        CM[ConfigMap]
        SECRET[Secret]
    end
    
    MODEL_YAML --> MODEL_CR
    
    MODEL_CR --> PVC
    MODEL_CR --> STATEFUL
    MODEL_CR --> DEPLOY
    MODEL_CR --> SVC
    MODEL_CR --> CM
    
    PVC --> PV
    PV --> SC
    
    DEPLOY --> RS
    RS --> PODS
    
    SVC --> ENDPOINTS
    ENDPOINTS --> PODS
    
    SVC --> INGRESS
    
    CM --> PODS
    SECRET --> PODS
    
    STATEFUL --> PVC
    PODS --> PVC
```

## Resource Creation Sequence

```mermaid
sequenceDiagram
    participant User
    participant API as K8s API Server
    participant Controller as Model Controller
    participant Scheduler as K8s Scheduler
    participant Kubelet as Kubelet
    
    User->>API: Apply Model CR
    API->>Controller: Model CR Created Event
    
    Note over Controller: Reconciliation Loop Starts
    
    Controller->>API: Create ConfigMap
    Controller->>API: Create Secret (if needed)
    Controller->>API: Create PVC
    
    Note over API,Scheduler: Storage Provisioning
    
    Controller->>API: Create StatefulSet (Image Storage)
    API->>Scheduler: Schedule StatefulSet Pod
    Scheduler->>Kubelet: Assign Pod to Node
    Kubelet->>API: Pod Status Update
    
    Controller->>API: Wait for StatefulSet Ready
    
    Controller->>API: Create Deployment (Inference)
    API->>Scheduler: Schedule Deployment Pods
    Scheduler->>Kubelet: Assign Pods to Nodes
    Kubelet->>API: Pod Status Updates
    
    Controller->>API: Create Service
    Controller->>API: Create Ingress (if specified)
    
    Controller->>API: Update Model CR Status
    API->>User: Status Available
```

## Storage Architecture

### Persistent Volume Strategy

```mermaid
graph TD
    subgraph "Storage Layers"
        subgraph "Model Images"
            OLLAMA_MODELS[Ollama Model Files<br/>GGUF Format]
            MODEL_METADATA[Model Metadata<br/>JSON Configuration]
        end
        
        subgraph "Persistent Storage"
            PVC_STORAGE[PVC: Model Storage]
            PV_BACKEND[PV Backend<br/>local-path/nfs/cloud]
        end
        
        subgraph "Pods"
            STORAGE_POD[Storage Pod<br/>StatefulSet]
            INFERENCE_PODS[Inference Pods<br/>Deployment]
        end
    end
    
    OLLAMA_MODELS --> PVC_STORAGE
    MODEL_METADATA --> PVC_STORAGE
    PVC_STORAGE --> PV_BACKEND
    
    PVC_STORAGE --> STORAGE_POD
    PVC_STORAGE --> INFERENCE_PODS
    
    STORAGE_POD --> OLLAMA_MODELS
    INFERENCE_PODS --> OLLAMA_MODELS
```

### Access Patterns

```mermaid
graph LR
    subgraph "Access Modes"
        RWO[ReadWriteOnce<br/>Single Node]
        RWX[ReadWriteMany<br/>Multi Node]
        ROX[ReadOnlyMany<br/>Read Only Multi]
    end
    
    subgraph "Use Cases"
        SINGLE[Single Node<br/>kind/minikube]
        MULTI[Multi Node<br/>Production]
        READONLY[Shared Models<br/>Read Only]
    end
    
    SINGLE --> RWO
    MULTI --> RWX
    READONLY --> ROX
```

## Network Architecture

### Service Types and Exposure

```mermaid
graph TD
    subgraph "Service Strategies"
        subgraph "ClusterIP"
            CLUSTER_IP[ClusterIP Service]
            INTERNAL[Internal Access Only]
        end
        
        subgraph "NodePort"
            NODE_PORT[NodePort Service]
            EXTERNAL[External Access]
            PORT_RANGE[Port Range 30000-32767]
        end
        
        subgraph "LoadBalancer"
            LB[LoadBalancer Service]
            CLOUD_LB[Cloud Load Balancer]
        end
        
        subgraph "Ingress"
            INGRESS_CTRL[Ingress Controller]
            HTTP_HTTPS[HTTP/HTTPS Access]
            TLS[TLS Termination]
        end
    end
    
    CLUSTER_IP --> INTERNAL
    NODE_PORT --> EXTERNAL
    NODE_PORT --> PORT_RANGE
    LB --> CLOUD_LB
    INGRESS_CTRL --> HTTP_HTTPS
    INGRESS_CTRL --> TLS
```

## RBAC and Security

### Permission Model

```mermaid
graph TD
    subgraph "Service Accounts"
        OPERATOR_SA[Operator Service Account]
        MODEL_SA[Model Service Account]
    end
    
    subgraph "Cluster Roles"
        OPERATOR_ROLE[Operator ClusterRole]
        MODEL_ROLE[Model Role]
    end
    
    subgraph "Role Bindings"
        OPERATOR_BINDING[Operator ClusterRoleBinding]
        MODEL_BINDING[Model RoleBinding]
    end
    
    subgraph "Resources"
        CRDS[Custom Resource Definitions]
        NATIVE[Native K8s Resources]
        SECRETS[Secrets & ConfigMaps]
    end
    
    OPERATOR_SA --> OPERATOR_BINDING
    MODEL_SA --> MODEL_BINDING
    
    OPERATOR_ROLE --> OPERATOR_BINDING
    MODEL_ROLE --> MODEL_BINDING
    
    OPERATOR_BINDING --> CRDS
    OPERATOR_BINDING --> NATIVE
    
    MODEL_BINDING --> SECRETS
```

### Security Contexts

```mermaid
graph LR
    subgraph "Pod Security"
        subgraph "Security Context"
            RUNASUSER[runAsUser: 1001]
            RUNASGROUP[runAsGroup: 1001]
            FSGROUP[fsGroup: 1001]
            NONROOT[runAsNonRoot: true]
        end
        
        subgraph "Capabilities"
            DROP_ALL[drop: ALL]
            ADD_NET[add: NET_BIND_SERVICE]
        end
        
        subgraph "Security Policies"
            READ_ONLY[readOnlyRootFilesystem]
            PRIVILEGED[allowPrivilegeEscalation: false]
        end
    end
    
    RUNASUSER --> NONROOT
    DROP_ALL --> ADD_NET
    READ_ONLY --> PRIVILEGED
```

## Resource Scaling and Management

### Horizontal Pod Autoscaler Integration

```mermaid
graph TD
    subgraph "Autoscaling Strategy"
        HPA[HorizontalPodAutoscaler]
        METRICS[Metrics Server]
        CUSTOM_METRICS[Custom Metrics]
        
        subgraph "Scaling Triggers"
            CPU[CPU Utilization]
            MEMORY[Memory Utilization]
            REQUEST_RATE[Request Rate]
            QUEUE_DEPTH[Queue Depth]
        end
        
        subgraph "Actions"
            SCALE_UP[Scale Up Replicas]
            SCALE_DOWN[Scale Down Replicas]
            MAINTAIN[Maintain Current]
        end
    end
    
    METRICS --> HPA
    CUSTOM_METRICS --> HPA
    
    HPA --> CPU
    HPA --> MEMORY
    HPA --> REQUEST_RATE
    HPA --> QUEUE_DEPTH
    
    CPU --> SCALE_UP
    MEMORY --> SCALE_UP
    REQUEST_RATE --> SCALE_DOWN
    QUEUE_DEPTH --> MAINTAIN
```

## Monitoring and Observability Resources

```mermaid
graph LR
    subgraph "Monitoring Stack"
        subgraph "Metrics"
            PROMETHEUS[Prometheus]
            SERVICEMONITOR[ServiceMonitor]
            PODMONITOR[PodMonitor]
        end
        
        subgraph "Logging"
            FLUENTD[Fluentd/Fluent Bit]
            ELASTICSEARCH[Elasticsearch]
            KIBANA[Kibana]
        end
        
        subgraph "Tracing"
            JAEGER[Jaeger]
            OPENTELEMETRY[OpenTelemetry]
        end
        
        subgraph "Alerting"
            ALERTMANAGER[AlertManager]
            PROMETHEUSRULE[PrometheusRule]
        end
    end
    
    PROMETHEUS --> SERVICEMONITOR
    PROMETHEUS --> PODMONITOR
    
    FLUENTD --> ELASTICSEARCH
    ELASTICSEARCH --> KIBANA
    
    JAEGER --> OPENTELEMETRY
    
    PROMETHEUS --> ALERTMANAGER
    ALERTMANAGER --> PROMETHEUSRULE
```

This comprehensive Kubernetes resources architecture ensures proper resource management, security, scalability, and observability for the EchoLlama-Echorator operator ecosystem.