## Home Assignment 5 - Working with Helm
###### This assignment focuses on understanding Helm and its components, managing Kubernetes apps, and remote hosting of Helm chart repos.

### 1. Conceptual Questions

#### a. Helm's role in Kubernetes

- Why Helm is preferred over managing plain Kubernetes YAML files?
  - **Simplified** Deployment: Helm packages related Kubernetes YAML files into a **single** unit called a **chart**, making it much easier to install, upgrade, and uninstall applications.
  - **Configuration** Management: Helm allows to define configurable parameters using **values.yaml** to customize deployments across different environments (e.g., dev, staging, prod) **without modifying** the core YAML templates.
  - **Version Control** and **Rollbacks**: Helm makes it easy to upgrade to new versions and, to rollback to previous stable versions. 
  - **Reusability and Sharing**: Helm charts can be easily shared and reused. There's a large ecosystem of pre-built, community-maintained charts for popular applications, saving the effort of writing everything from scratch.  
  - **Dependency** Management: Helm allows to define dependencies between charts. If the app relies on other services (like a database), Helm can help manage their deployment as well.
  - **Templating**: Helm uses a templating engine, allowing for dynamic generation of Kubernetes manifests based on values and logic. This reduces redundancy and makes configurations more flexible.
---
- Key components of a Helm chart.
  - `Chart.yaml`- contains **metadata** about the chart: apiVersion, name, version, appVersion, type, description, dependencies, and other metadata
  - `values.yaml`- contains the default config values for the chart; it separates config from chart's templates
  - `templates/` directory - contains template files. These are YAML manifests for Kubernetes objects (like Deployments, Services, Pods, etc.) written using the Go template language
  - `charts/` directory (optional) - contains other Helm charts(sub-charts) that this chart depends on.
  - `NOTES.txt`(optional) - contains **short** usage **notes** about app access, or important config details.
  - `README.md` - human-readable intro to the chart, its purpose, how to install and config details
  - `LICENSE`(optional) - contains the license under which the chart is distributed.
---
#### b. Environment-specific config.
- **Multiple `values.yaml` files**: each environment has its own `values.yaml`(e.g., values-**dev**.yaml, values-**staging**.yaml, values-**prod**.yaml).
These files contain the specific configurations for each environment, like database URLs, API keys, replica counts, etc.
- **Conditional Logic in Templates**: enable specific features based on an environment value
  - Note: the `environment` value is not included by default 
    in a basic chart and would be added to `values.yaml` file
```yaml
# values-prod.yaml
environment: "prod"
```
```yaml
# templates/deployment.yaml
{{- if eq .Values.environment "prod" }}
strategy:
  type: RollingUpdate
{{- else }}
strategy:
  type: Recreate
{{- end }}
```
- --set **Flag**: allows to override env values on the command line during installation or upgrade
  - Good for quick testing: avoid modifying the original `values.yaml` file.
```
helm upgrade -i <release-name> ./<chart-name> --set environment="prod"
```
---
#### c. Helm Chart Repos
- They are **servers** that **store** and **serve** packaged Helm charts over **HTTP(S)**.
- There are 2 main types of repos:
  - **Traditional** repos: These are **HTTP(S) servers** that rely on an `index.yaml` file to list and describe Helm charts. They primarily focus on serving Helm charts.
  - **OCI-based registry** repos: These leverage standard **HTTP(S) container registries** and do **not rely** on a central `index.yaml` file for Helm chart discovery. Instead, they use the **OCI Distribution Specification API** built on top of HTTP(S). These registries can store and serve various **OCI artifacts**, including Helm charts and container images.
- **Traditional** repos:
  - Simple Web Servers - host packaged charts (.tgz) and an `index.yaml` file on a standard HTTP(S) server (e.g., Nginx, Apache). Require manual management of the `index.yaml`.
  - **Cloud** Storage - utilize services like Amazon S3, GC Storage, or Azure Blob Storage as HTTP(S) servers to store charts. Often requires tools or plugins to manage the `index.yaml`.
  -  **Helm Repo Servers** - Use specialized apps (i.e., JFrog Artifactory) for chart hosting.
  - **GitHub Pages** - A free option for hosting public Helm charts directly from a GitHub repo. Requires a specific repo structure and a workflow to generate and update the `index.yaml`.
- **OCI-based registry** repos:
  - Standard container registries that support the OCI Distribution Specification to store Helm charts as **OCI artifacts**.
  - Examples: Docker Registry, Amazon ECR, Google Artifact Registry, Azure Container Registry.

---
#### d. CI/CD Integration (Simplified workflow)
- Steps in **CI** pipeline:
  - **Checkout** code
  - **Set up** Helm
  - **Lint** Helm Chart
  - **Package** Helm Chart
  - **Upload** Chart Artifact (for OCI-based: login + push)
- Steps in **CD** pipeline:
  - **Checkout** code
  - **Download** Chart Artifact
  - **Set up** Helm
  - **Authenticate** to Kubernetes (configure access to the cluster)
  - **Deploy** Helm Chart (for OCI-based: pull + deploy)
---
### 2. Practical Tasks

#### a. Helm installation
- Check if Helm already installed
```bash
   helm version
```
- If Helm not installed, install it with Homebrew, and check installed version again
```bash
   brew install helm
   helm version
```
#### b. Creating a Helm Chart
- Create chart named **my-chart**
  - Important: the current working directory is the project **root**
```bash
   helm create my-chart
```
- Verify the creation of `my-chart` directory in the project **root**
```tree
my-chart/
├── Chart.yaml          # Info about the chart
├── templates/          # Kubernetes manifest templates
│   ├── NOTES.txt       # User instructions
│   ├── _helpers.tpl    # Template helpers
│   ├── deployment.yaml # Example deployment
│   ├── service.yaml    # Example service
│   └── serviceaccount.yaml # Example service account
└── values.yaml         # Default config values
```
- Review and modify the default `values.yaml`:
  - replicaCount: 2
  - image.repository: nginx
  - image.tag: latest

#### c. Deploy the updated Helm chart (upgrade the chart)
- The command structure: ` helm upgrade -i <release-name> ./<chart-dir>`
  -  Note 1: `./<chart-dir>` represents the **relative** path from the project's **root** directory to the Helm chart directory.
  -  Note 2: Verify that `template/deployments.yaml` has: 
     - `spec.replicas: {{ .Values.replicaCount }}`
  -  Note 3: The flag `-i` tells Helm to **install-or-upgrade**. Very useful in CI/CD pipelines to ensure the app deployed without fail
```bash
   helm upgrade -i my-release ./my-chart
```
- Verify the deployment by listing the running Pods:
  - Expected output: **two** appropriate Pods in the **running** status
```bash
   kubectl get pods
```
- Modify in the `values.yaml`:
  - replicaCount: 3
- Monitor by the same command: `kubectl get pods`
  - Expected output: **three** appropriate Pods in the **running** status
  
### 3. Advanced Task
#### a. Creating a Helm chart repo
- Packaging the chart: 
  - The command structure is `helm package ./<chart-dir>`
  - Note: add `<chart-dir>/.helmignore` file to determine which files and directories to **exclude** from the resulting `.tgz` archive.
```bash
   helm package ./my-chart
```

#### b. Creating Helm Chart repo using Docker Hub OCI Registry
- Log in to Docker Hub via Helm: 
  - `helm registry login docker.io -u <dockerhub_username>` 
  - fill password after prompt
- Push the chart to Docker Hub as an OCI Artifact
  - `helm chart push ./<package-name.tgz> oci://registry-1.docker.io/<dockerhub_username>`
```bash
   helm chart push ./my-chart-0.1.0.tgz oci://registry-1.docker.io/alkon100
```
- Verify on Docker Hub:
  - Navigate to `https://hub.docker.com/r/<dockerhub_username>/my-chart/tags`
- (Re-)Install the created OCI Registry Repo:
  - `helm upgrade -i <release-name> oci://registry-1.docker.io/<dockerhub_username>/<chart-name>:v<version-number>`  

 
 


