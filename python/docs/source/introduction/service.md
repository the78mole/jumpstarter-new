# Service

When building a lab with many devices under test, it quickly becomes difficult
to keep track of devices, schedule access for automated tests, and perform
routine maintenance such as batch updates.

Jumpstarter provides a service that can be installed in any
[Kubernetes](https://kubernetes.io/) cluster to manage connected clients and
exporters.

If you're already using a Kubernetes-native CI tool such as
[Tekton](https://tekton.dev/), [Jenkins X](https://jenkins-x.io/),
[Konflux](https://konflux-ci.dev), or [GitLab
CI](https://docs.gitlab.com/user/clusters/agent/ci_cd_workflow/), Jumpstarter
can integrate directly into your existing cloud or on-premises cluster.

```{mermaid}
:config: {"theme":"base","themeVariables":{"primaryColor":"#f8f8f8","primaryTextColor":"#000","primaryBorderColor":"#e5e5e5","lineColor":"#3d94ff","secondaryColor":"#f8f8f8","tertiaryColor":"#fff"}}
flowchart TB
    subgraph "Kubernetes Cluster"
        Controller["Controller<br/>(CRDs, Auth, Leases)"]
        Router1["Router 1"]
        Router2["Router 2"]
    end

    subgraph "Clients"
        CI["CI Pipeline"]
        Dev["Developer"]
    end

    subgraph "Lab"
        Exp1["Exporter 1<br/>(Host A)"]
        Exp2["Exporter 2<br/>(Host B)"]
        DUT1["DUT 1"]
        DUT2["DUT 2"]
    end

    CI -- "Request Lease" --> Controller
    Dev -- "Request Lease" --> Controller
    Controller -- "Assign" --> Router1
    Controller -- "Assign" --> Router2
    CI <-- "gRPC Tunnel" --> Router1
    Dev <-- "gRPC Tunnel" --> Router2
    Router1 <-- "gRPC" --> Exp1
    Router2 <-- "gRPC" --> Exp2
    Exp1 --> DUT1
    Exp2 --> DUT2
```

## Controller

The core of the Service is the Controller, which manages access to devices,
authenticates clients/exporters, and maintains a set of labels to easily
identify specific devices.

The Controller is implemented as a Kubernetes
[controller](https://github.com/jumpstarter-dev/jumpstarter-controller) using
[Custom Resource Definitions
(CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
to store information about clients, exporters, leases, and other resources.

### Leases

When a client requests access to an exporter and a matching instance is found, a
Lease is created. The lease ensures that each lessee (client) has exclusive
access to a specific device/exporter.

Clients can be scheduled to access a specific exporter or any exporter that
matches a set of requested labels, similar to [node
selection](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)
in Kubernetes. This enables flexible CI-driven testing even when physical
resources are limited.

## Router

The Router service is used by the controller to route messages between clients
and exporters through a gRPC tunnel. This enables remote access to exported
interfaces via the client.

Once a lease is established, all traffic between the client and the exporter
flows through a router instance. While there may only be one controller, the
router can be scaled with multiple instances to handle traffic between many
clients and exporters simultaneously.