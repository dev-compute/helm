# Helm — Day 1 Notes

## 1. What is Helm?

Helm is a package manager for Kubernetes. It helps us install, configure, upgrade, and uninstall applications running in a Kubernetes environment.

### Why do we need Helm?

Deploying an individual application and its Kubernetes resources is relatively easy. However, managing an application becomes tedious as the number of resources increases.

For example, a single application might require several Kubernetes resources:

- Deployment
- Service
- ConfigMap
- Secret
- PersistentVolume (PV)
- PersistentVolumeClaim (PVC)
- RBAC resources
- Ingress

Each resource is usually defined in a separate YAML manifest file. As the application grows, managing multiple files becomes increasingly difficult.

The main challenges are:

1. **Managing multiple files:** We need to maintain several YAML files for a single application.
2. **Updating configurations:** If we need to change a common configuration, such as a label, we might have to modify multiple files.
3. **Deploying changes:** After modifying the manifests, we need to apply the changes to the Kubernetes cluster.
4. **Deleting resources:** We need to ensure that all resources associated with the application are removed correctly.
5. **Risk of mistakes:** Managing multiple files manually increases the chances of inconsistencies and human errors.

We could combine all the resource definitions into a single YAML file. However, that file might become very large and difficult to maintain.

Helm helps us solve these problems by packaging the Kubernetes resource definitions into a reusable unit called a **Helm chart**.

## 2. How does Helm help us manage an application?

Kubernetes primarily manages individual resources based on their specifications. It does not automatically understand that a Deployment, ConfigMap, Secret, or PersistentVolume belongs to a particular application simply because they are related.

For example, Kubernetes does not automatically group all resources belonging to an application into one manageable unit. We can use labels, namespaces, and owner references to establish relationships, but these do not replace Helm's application-level release management.

From an application management perspective, however, we want to treat all the resources required by an application as a single unit.

This is where Helm helps us.

Helm groups the Kubernetes manifests required by an application into a **chart** and manages their deployment as a **release**.

### Key benefits of Helm

**1. Centralized configuration**

Helm allows us to define configurable values in a `values.yaml` file.

For example, if multiple Kubernetes resources use the same application name, image tag, or label, we can define the relevant values centrally and reference them from multiple templates.

Instead of manually modifying several YAML files, we can update the appropriate value and apply the change through Helm.

**2. Easy installation and upgrades**

We can install or upgrade an application using a single Helm command.

Helm renders the chart templates using the supplied values and applies the resulting Kubernetes resource definitions. It also tracks the release's revisions and manages subsequent upgrades.

**3. Release history and rollback**

Helm maintains revision history for a release. This allows us to roll back to a previous revision when necessary.

**4. Easy uninstallation**

When we uninstall a Helm release, Helm identifies the resources managed by that release and removes them as appropriate.

We do not have to manually remember every manifest file associated with the application. However, resources retained through policies or resources created independently may require separate handling.

## 3. Common Helm Commands

Helm uses a release name to identify an installed instance of a chart.

### 3.1 Install a chart

**Syntax:**

```bash
helm install [RELEASE_NAME] [CHART_REFERENCE]
```

**Example:**

```bash
helm install wordpress oci://localhost:5000/helm-charts/wordpress --version 0.1.0
```

This command installs version `0.1.0` of the `wordpress` chart and creates a Helm release named `wordpress`.

### 3.2 Upgrade a release

**Syntax:**

```bash
helm upgrade [RELEASE_NAME] [CHART_REFERENCE]
```

**Example:**

```bash
helm upgrade wordpress oci://localhost:5000/helm-charts/wordpress --version 0.1.1
```

This command upgrades the existing `wordpress` release using chart version `0.1.1`.

### 3.3 Roll back a release

**Syntax:**

```bash
helm rollback [RELEASE_NAME] [REVISION]
```

**Example:**

```bash
helm rollback wordpress 1
```

This command rolls back the `wordpress` release to revision `1`, provided that the revision is available.

### 3.4 Uninstall a release

**Syntax:**

```bash
helm uninstall [RELEASE_NAME]
```

**Example:**

```bash
helm uninstall wordpress
```

This command uninstalls the Helm release named `wordpress` and removes the Kubernetes resources managed by that release, subject to applicable resource policies and lifecycle behavior.

---

## 4. Helm Charts and OCI Registries

### 4.1 What is a Helm chart?

A **Helm chart** is a package containing the files required to define and configure a Kubernetes application.

A typical chart contains:

- `Chart.yaml` — Chart metadata, such as its name, version, and description.
- `values.yaml` — Default configuration values.
- `templates/` — Kubernetes manifest templates.
- `charts/` — Optional chart dependencies.
- `README.md` — Optional documentation.

A Helm chart can be packaged into a compressed archive with the `.tgz` extension.

For example:

```text
mychart-0.1.0.tgz
```

This archive contains the chart's files in a packaged format.

### 4.2 Where do we store Helm charts?

We can store and distribute Helm charts using an OCI-compatible container registry.

OCI stands for **Open Container Initiative**. OCI defines specifications for container images and related artifacts, including how they are represented and distributed through registries.

The same registry infrastructure used for container images can also store Helm charts as OCI artifacts.

Examples of registries that support OCI Helm charts include:

- Amazon Elastic Container Registry (Amazon ECR)
- Azure Container Registry (ACR)
- Docker Hub
- JFrog Artifactory

In this approach, we do not need a traditional Helm chart repository hosted on a basic HTTP server. We can publish and retrieve charts using OCI registry operations instead.

### 4.3 How are Helm charts stored in an OCI registry?

Consider the following command:

```bash
helm push mychart-0.1.0.tgz oci://localhost:5000/helm-charts
```

Example output:

```text
Pushed: localhost:5000/helm-charts/mychart:0.1.0
Digest: sha256:ec5f08ee7be8b557cd1fc5ae1a0ac985e8538da7c93f51a51eff4b277509a723
```

Let's understand each component.

| Component | Explanation |
|---|---|
| Registry host | `localhost:5000` — The registry's hostname and port. |
| Repository path | `helm-charts/mychart` — The complete repository path for this chart. The namespace(helm-charts) + chart name(mychart) combined form a single repository path |
| Artifact | The packaged Helm chart, distributed as an OCI artifact.  The underlying content of the mychart Helm chart packaged as a container-like layer. |
| Tag | `0.1.0` — Identifies the chart version. |
| Digest | A content-derived identifier used to identify the artifact's content. |

**Important:** When pushing a chart, the destination argument ends with the parent repository path, such as `helm-charts`. Helm derives the chart name and version from the packaged chart metadata and filename.

For example, if we push another chart named `frontend`:

```bash
helm push frontend-1.0.0.tgz oci://localhost:5000/helm-charts
```

The resulting reference is:

```text
localhost:5000/helm-charts/frontend:1.0.0
```

### 4.4 How does the registry organize multiple charts?

An OCI registry treats each chart's full repository path as a separate repository.

For example, after pushing two charts, the registry contains these repositories:

```text
localhost:5000
└── helm-charts/
    ├── mychart
    │   ├── 0.1.0
    │   ├── 0.1.1
    │   └── 1.0.0
    │
    └── frontend
        └── 1.0.0
```

This is a conceptual representation of the registry's repository and tag structure, not necessarily its exact user interface.

Here:

- `helm-charts/mychart` is one repository containing multiple versions of the `mychart` chart.
- `helm-charts/frontend` is another repository containing versions of the `frontend` chart.
- Each version tag points to a particular artifact manifest, which identifies the associated content by digest.

**Key takeaway:** A registry can store multiple Helm charts, and each chart repository can contain multiple versions.

## 5. What happens behind the scenes?

When a Helm chart is pushed to an OCI registry, the chart is stored using the OCI artifact model rather than simply being uploaded as an ordinary file to a folder.

An OCI artifact includes a manifest that describes its configuration and content.

For a Helm chart, the manifest commonly references the following components:

### OCI Manifest

Identifies the artifact's configuration and content.

**Config layer**

Chart metadata, including the Helm chart configuration media type:

```text
application/vnd.cncf.helm.config.v1+json
```

**Content layer**

The packaged chart content, typically containing the chart's templates and values:

```text
application/vnd.cncf.helm.chart.content.v1.tar+gzip
```

The content layer contains the packaged chart archive in a compressed tar format. The registry stores the content as a blob, referenced by the artifact manifest.

When we run `helm pull`, Helm resolves the requested chart version, retrieves the associated artifact, and downloads the chart content.

For example:

```bash
helm pull oci://localhost:5000/helm-charts/mychart --version 0.1.0
```

This command downloads the chart archive as:

```text
mychart-0.1.0.tgz
```

We can also extract the chart archive:

```bash
tar -xzf mychart-0.1.0.tgz
```

**Note:** The digest identifies content; the version tag is a human-readable reference to a particular chart release. The tag itself is not the content.

## 6. Inspect a Helm Chart Without Installing It

Helm provides the `helm show` command to inspect chart information and files directly from a chart reference, including a chart hosted in an OCI registry.

These commands retrieve the relevant chart information without installing the application into a Kubernetes cluster.

### 6.1 View `values.yaml`

```bash
helm show values oci://localhost:5000/helm-charts/mychart --version 0.1.0
```

Displays the default configuration values defined by the specified chart version.

For example, the output might include:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"

service:
  type: ClusterIP
  port: 80
```

These are illustrative values; the actual output depends on the chart.

### 6.2 View `Chart.yaml` metadata

```bash
helm show chart oci://localhost:5000/helm-charts/mychart --version 0.1.0
```

Displays chart metadata, such as:

- Chart name
- Chart version
- Description
- Application version
- Dependencies and other metadata, when defined

### 6.3 View `README.md`

```bash
helm show readme oci://localhost:5000/helm-charts/mychart --version 0.1.0
```

Displays the chart's README content, if a README is included in the chart.

### 6.4 View all available chart information

```bash
helm show all oci://localhost:5000/helm-charts/mychart --version 0.1.0
```

Displays the supported chart information and file contents, including chart metadata, default values, and README content when present.

### 6.5 Authenticate with the registry

If the registry requires authentication, log in before running these commands:

```bash
helm registry login localhost:5000
```

Helm will prompt for the appropriate credentials unless they are supplied through supported options.

For registries such as Azure Container Registry, Amazon ECR, or Docker Hub, use the authentication method appropriate to that registry.

## 7. Quick Revision

| Term | Meaning |
|---|---|
| Helm | A package manager for Kubernetes. |
| Chart | A package containing Kubernetes templates, configuration, and related files. |
| Release | An installed instance of a Helm chart managed by Helm. |
| `Chart.yaml` | Contains chart metadata. |
| `values.yaml` | Defines default configuration values for chart templates. |
| `templates/` | Contains templates that Helm renders into Kubernetes manifests. |
| `.tgz` | A compressed archive used to package a Helm chart. |
| OCI registry | A registry that stores and distributes OCI artifacts, including Helm charts. |
| Repository | A named path(namespace(helm-charts) + chart name(mychart) combined form a single repository path) in the registry containing artifact manifests and tags. |
| Tag | A human-readable reference, commonly used for a chart version. |
| Digest | A content-derived identifier for an artifact. |
| Revision | A numbered version of a Helm release's history. |

### The overall workflow

1. **Create a Helm chart** — `Chart.yaml`, `values.yaml`, and `templates/`
2. **Package the chart** — For example, `mychart-0.1.0.tgz`
3. **Push to an OCI registry** — Use `helm push`
4. **Install or upgrade the release** — Use `helm install` or `helm upgrade`
5. **Manage the release** — Inspect, upgrade, roll back, or uninstall

The most important distinction to remember is that **a chart is the package/artifact, a release is an installed instance of that package/artifact, and an OCI registry is where the packaged chart can be stored and distributed.**
