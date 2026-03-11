# jumpstarter-operator

The Jumpstarter Operator is a Kubernetes operator that automates the deployment
and lifecycle management of the Jumpstarter service. It uses a single
`Jumpstarter` Custom Resource (CR) to declaratively configure and manage
controller pods, router pods, TLS certificates (via cert-manager), networking
endpoints, RBAC, and authentication — all within a target namespace.

## Description

The operator watches for `Jumpstarter` custom resources
(API group `operator.jumpstarter.dev/v1alpha1`) and reconciles the full
Jumpstarter stack into the namespace where the CR resides. A single CR drives
the creation of:

- **Controller Deployment** – the main API server that manages clients,
  exporters, and leases through gRPC and an optional REST API.
- **Router Deployments** – one Deployment per replica, each responsible for
  routing gRPC traffic between clients and exporters.
- **TLS Certificates** – optionally managed by cert-manager with self-signed CA
  or an external Issuer/ClusterIssuer.
- **Networking Resources** – OpenShift Routes, Kubernetes Ingresses, NodePort
  services, or LoadBalancer services depending on the cluster environment.
- **RBAC** – ServiceAccounts, Roles, and RoleBindings required by the managed
  controller and router pods.
- **Authentication** – internal token-based auth, Kubernetes service-account
  auth, and external JWT/OIDC providers.
- **ConfigMaps & Secrets** – controller configuration, router configuration,
  and CA certificate bundles.

### Custom Resource Definitions

| CRD | API Group | Description |
|-----|-----------|-------------|
| `Jumpstarter` | `operator.jumpstarter.dev/v1alpha1` | Top-level CR that defines a full Jumpstarter deployment |
| `Client` | `jumpstarter.dev/v1alpha1` | Represents an authenticated client |
| `Exporter` | `jumpstarter.dev/v1alpha1` | Represents a device exporter |
| `Lease` | `jumpstarter.dev/v1alpha1` | Exclusive access grant from a client to an exporter |
| `ExporterAccessPolicy` | `jumpstarter.dev/v1alpha1` | Label-based access control between clients and exporters |

### Status Conditions

The operator reports readiness through standard Kubernetes conditions on the
`Jumpstarter` resource:

| Condition | Description |
|-----------|-------------|
| `CertManagerAvailable` | cert-manager CRDs are installed in the cluster |
| `IssuerReady` | The configured cert-manager Issuer is ready |
| `ControllerCertificateReady` | Controller TLS certificate secret exists |
| `RouterCertificatesReady` | All router TLS certificate secrets exist |
| `ControllerDeploymentReady` | Controller Deployment is available |
| `RouterDeploymentsReady` | All router Deployments are available |
| `Ready` | Aggregate condition — true when all components are ready |

## Getting Started

### Prerequisites
- go version v1.24.0+
- docker version 17.03+.
- kubectl version v1.11.3+.
- Access to a Kubernetes v1.11.3+ cluster.

### To Deploy on the cluster
**Build and push your image to the location specified by `IMG`:**

```sh
make docker-build docker-push IMG=<some-registry>/jumpstarter-operator:tag
```

**NOTE:** This image ought to be published in the personal registry you specified.
And it is required to have access to pull the image from the working environment.
Make sure you have the proper permission to the registry if the above commands don’t work.

**Install the CRDs into the cluster:**

```sh
make install
```

**Deploy the Manager to the cluster with the image specified by `IMG`:**

```sh
make deploy IMG=<some-registry>/jumpstarter-operator:tag
```

> **NOTE**: If you encounter RBAC errors, you may need to grant yourself cluster-admin
privileges or be logged in as admin.

**Create instances of your solution**
You can apply the samples (examples) from the config/sample:

```sh
kubectl apply -k config/samples/
```

>**NOTE**: Ensure that the samples has default values to test it out.

### To Uninstall
**Delete the instances (CRs) from the cluster:**

```sh
kubectl delete -k config/samples/
```

**Delete the APIs(CRDs) from the cluster:**

```sh
make uninstall
```

**UnDeploy the controller from the cluster:**

```sh
make undeploy
```

## Project Distribution

Following the options to release and provide this solution to the users.

### By providing a bundle with all YAML files

1. Build the installer for the image built and published in the registry:

```sh
make build-installer IMG=<some-registry>/jumpstarter-operator:tag
```

**NOTE:** The makefile target mentioned above generates an 'install.yaml'
file in the dist directory. This file contains all the resources built
with Kustomize, which are necessary to install this project without its
dependencies.

2. Using the installer

Users can just run 'kubectl apply -f <URL for YAML BUNDLE>' to install
the project, i.e.:

```sh
kubectl apply -f https://raw.githubusercontent.com/<org>/jumpstarter-operator/<tag or branch>/dist/install.yaml
```

### By providing a Helm Chart

1. Build the chart using the optional helm plugin

```sh
operator-sdk edit --plugins=helm/v1-alpha
```

2. See that a chart was generated under 'dist/chart', and users
can obtain this solution from there.

**NOTE:** If you change the project, you need to update the Helm Chart
using the same command above to sync the latest changes. Furthermore,
if you create webhooks, you need to use the above command with
the '--force' flag and manually ensure that any custom configuration
previously added to 'dist/chart/values.yaml' or 'dist/chart/manager/manager.yaml'
is manually re-applied afterwards.

## Contributing

Contributions are welcome! To get started:

1. Fork the repository and create a feature branch.
2. Ensure your changes compile and pass existing tests:
   ```sh
   make test
   ```
3. Follow the [Kubebuilder conventions](https://book.kubebuilder.io/introduction.html)
   for controller and API changes.
4. If you modify CRD types in `api/v1alpha1/`, regenerate manifests:
   ```sh
   make manifests generate
   ```
5. Open a pull request with a clear description of the change.

**NOTE:** Run `make help` for more information on all potential `make` targets

More information can be found via the [Kubebuilder Documentation](https://book.kubebuilder.io/introduction.html)

## License

Copyright 2025.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

