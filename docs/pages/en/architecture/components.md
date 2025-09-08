# Component Architecture

This document details the internal components and their interactions within the EchoLlama-Echorator system.

## Core Components Overview

```mermaid
graph LR
    subgraph "EchoLlama-Echorator Operator"
        subgraph "Controllers"
            MC[Model Controller]
            RC[Reconcile Controller]
            SC[Status Controller]
        end
        
        subgraph "CLI Tools"
            KOLLAMA[kollama CLI]
            KUBECTL[kubectl integration]
        end
        
        subgraph "APIs"
            CRDAPI[CRD API v1]
            WEBHOOK[Admission Webhooks]
            METRICS[Metrics API]
        end
    end
    
    subgraph "Kubernetes Resources"
        subgraph "Custom Resources"
            MODEL[Model CRD]
        end
        
        subgraph "Native Resources"
            DEPLOY[Deployments]
            SVC[Services]
            PVC[PersistentVolumeClaims]
            STATEFUL[StatefulSets]
        end
    end
    
    subgraph "EchoLlama Engine"
        CORE[Core Engine]
        API_SERVER[API Server]
        MODEL_MGR[Model Manager]
        INFERENCE[Inference Engine]
    end
    
    subgraph "Deep Tree Echo"
        COGNITIVE[Cognitive Engine]
        MEMORY[Memory System]
        LEARNING[Learning System]
        VISUALIZATION[Visualization]
    end
    
    MC --> MODEL
    MC --> DEPLOY
    MC --> SVC
    MC --> PVC
    MC --> STATEFUL
    
    RC --> MC
    SC --> METRICS
    
    KOLLAMA --> CRDAPI
    KUBECTL --> CRDAPI
    
    WEBHOOK --> MODEL
    
    DEPLOY --> CORE
    SVC --> API_SERVER
    
    CORE --> MODEL_MGR
    CORE --> INFERENCE
    
    MODEL_MGR --> COGNITIVE
    INFERENCE --> MEMORY
    API_SERVER --> LEARNING
    METRICS --> VISUALIZATION
```

## Model Controller Detail

The Model Controller is the heart of the operator, responsible for managing the complete lifecycle of Ollama models.

```mermaid
stateDiagram-v2
    [*] --> Pending: Create Model Resource
    
    Pending --> ImagePulling: Validate Spec
    ImagePulling --> ImageReady: Pull Success
    ImagePulling --> Failed: Pull Error
    
    ImageReady --> StorageProvisioning: Image Available
    StorageProvisioning --> StorageReady: PVC Created
    StorageProvisioning --> Failed: Storage Error
    
    StorageReady --> ModelDeploying: Storage Available
    ModelDeploying --> ModelRunning: Deployment Ready
    ModelDeploying --> Failed: Deployment Error
    
    ModelRunning --> EchoLlamaIntegration: Model Healthy
    EchoLlamaIntegration --> DeepTreeEcho: Engine Connected
    DeepTreeEcho --> Ready: Cognitive Layer Active
    
    Ready --> ModelRunning: Health Check
    Ready --> Scaling: Replica Change
    Scaling --> Ready: Scale Complete
    
    Ready --> Terminating: Delete Request
    ModelRunning --> Terminating: Delete Request
    Failed --> Terminating: Delete Request
    
    Terminating --> [*]: Cleanup Complete
    
    Failed --> Pending: Manual Retry
```

## Resource Management Architecture

```mermaid
graph TD
    subgraph "Model Instance Lifecycle"
        MODEL_CR[Model Custom Resource]
        CONTROLLER[Model Controller]
        
        subgraph "Image Management"
            IMAGE_PULLER[Image Puller]
            IMAGE_STORAGE[Model Image Storage]
            STATEFULSET[StatefulSet for Storage]
        end
        
        subgraph "Inference Management"
            DEPLOYMENT[Deployment]
            PODS[Model Pods]
            SERVICE[Service]
        end
        
        subgraph "Persistence"
            PVC[PersistentVolumeClaim]
            PV[PersistentVolume]
            STORAGE_CLASS[StorageClass]
        end
    end
    
    MODEL_CR --> CONTROLLER
    CONTROLLER --> IMAGE_PULLER
    IMAGE_PULLER --> IMAGE_STORAGE
    IMAGE_STORAGE --> STATEFULSET
    
    CONTROLLER --> DEPLOYMENT
    DEPLOYMENT --> PODS
    PODS --> SERVICE
    
    CONTROLLER --> PVC
    PVC --> PV
    PV --> STORAGE_CLASS
    
    STATEFULSET --> PVC
    PODS --> IMAGE_STORAGE
```

## Component Responsibilities

### Model Controller
- **Primary Function**: Reconcile Model custom resources
- **Key Operations**:
  - Validate model specifications
  - Coordinate image pulling and storage
  - Manage deployment lifecycle
  - Handle scaling operations
  - Monitor health and status
  - Integrate with EchoLlama engine
  - Coordinate with Deep Tree Echo system

### Reconcile Controller
- **Primary Function**: Ensure desired state matches actual state
- **Key Operations**:
  - Continuous reconciliation loops
  - Error recovery mechanisms
  - State drift detection
  - Resource cleanup

### Status Controller
- **Primary Function**: Maintain accurate status information
- **Key Operations**:
  - Health monitoring
  - Metrics collection
  - Status reporting
  - Event generation

### CLI Tools Integration
- **kollama CLI**: User-friendly command-line interface
- **kubectl integration**: Native Kubernetes tooling support

## Inter-Component Communication

```mermaid
sequenceDiagram
    participant User
    participant CLI as kollama CLI
    participant API as Kubernetes API
    participant Controller as Model Controller
    participant Engine as EchoLlama Engine
    participant Echo as Deep Tree Echo
    
    User->>CLI: kollama deploy phi
    CLI->>API: Create Model Resource
    API->>Controller: Watch Event
    
    Controller->>Controller: Validate Spec
    Controller->>API: Create StatefulSet (Image Storage)
    Controller->>API: Create Deployment (Inference)
    Controller->>API: Create Service
    Controller->>API: Create PVC
    
    Controller->>Engine: Register Model Instance
    Engine->>Echo: Initialize Cognitive Layer
    Echo->>Engine: Cognitive Layer Ready
    Engine->>Controller: Registration Complete
    
    Controller->>API: Update Status (Ready)
    API->>CLI: Status Update
    CLI->>User: Deployment Complete
```

## Error Handling and Recovery

```mermaid
graph TD
    ERROR[Error Detected] --> CLASSIFY{Classify Error}
    
    CLASSIFY -->|Transient| RETRY[Exponential Backoff Retry]
    CLASSIFY -->|Configuration| CONFIG_ERROR[Update Status: Configuration Error]
    CLASSIFY -->|Resource| RESOURCE_ERROR[Update Status: Resource Error]
    CLASSIFY -->|System| SYSTEM_ERROR[Update Status: System Error]
    
    RETRY --> SUCCESS{Retry Success?}
    SUCCESS -->|Yes| RESOLVED[Mark as Resolved]
    SUCCESS -->|No| MAX_RETRIES{Max Retries?}
    
    MAX_RETRIES -->|No| RETRY
    MAX_RETRIES -->|Yes| PERMANENT_ERROR[Mark as Failed]
    
    CONFIG_ERROR --> MANUAL_INTERVENTION[Require Manual Fix]
    RESOURCE_ERROR --> AUTO_CLEANUP[Attempt Auto Cleanup]
    SYSTEM_ERROR --> ESCALATE[Escalate to Admin]
    
    AUTO_CLEANUP --> RETRY_PROVISION[Retry Provisioning]
    RETRY_PROVISION --> SUCCESS
```

This component architecture ensures robust, scalable, and maintainable operations while providing deep integration with the EchoLlama ecosystem and Deep Tree Echo cognitive capabilities.