
Splunk Operator Deployment with Helmfile
========================================

This project provides a structured approach to manage Splunk deployments across different environments using Helmfile. By leveraging Helmfile, we can maintain consistency, simplify complex deployments, and handle environment-specific configurations efficiently.

Table of Contents
-----------------

* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Project Structure](#project-structure)
* [Configuration Files](#configuration-files)
  * [helmfile.yaml](#helmfileyaml)
  * [repos.yaml](#reposyaml)
  * [Templates](#templates)
  * [Environments](#environments)
  * [.helmfile\-local](#helmfile-local)
* [Usage](#usage)
  * [Deploying to Development Environment](#deploying-to-development-environment)
  * [Deploying to Staging Environment](#deploying-to-staging-environment)
  * [Deploying to Production Environment](#deploying-to-production-environment)
  * [Using Personal Configurations](#using-personal-configurations)
* [Customization](#customization)
  * [Adding New Releases](#adding-new-releases)
  * [Overriding Values and Secrets](#overriding-values-and-secrets)
* [Troubleshooting](#troubleshooting)
* [Contributing](#contributing)
* [License](#license)

---

Architecture
------------

The deployment consists of two main components:

* **Splunk Operator**: Manages the lifecycle of Splunk deployments in Kubernetes.
* **Splunk Standalone Instance**: A standalone Splunk Enterprise instance for log aggregation and analysis.

Prerequisites
-------------

Before you begin, ensure you have the following installed:

* **Kubernetes Cluster**: A running Kubernetes cluster (v1.18+).
* **Helm**: Package manager for Kubernetes (v3.0+).
* **Helmfile**: Declaratively manage your Helm charts (v0.139.9+).

Project Structure
-----------------

```php
helmfile.d/
├── helmfile.yaml            # Main Helmfile configuration
├── repos.yaml               # Shared Helm repositories
├── templates/
│   └── default-template.yaml # Shared templates for releases
├── environments/
│   ├── global-env-defaults.yaml  # Global environment defaults
│   ├── dev-values.yaml           # Development environment values
│   ├── staging-values.yaml       # Staging environment values
│   └── prod-values.yaml          # Production environment values
└── .helmfile-local/          # Personal configurations (gitignored)
```

Configuration Files
-------------------

### helmfile.yaml

The main configuration file that orchestrates the deployment:

```yaml
helmDefaults:
  cleanupOnFail: true
  verify: false
  wait: false
  waitForJobs: true
  timeout: 600
  recreatePods: false
  force: true
  historyMax: 10
  createNamespace: true
  devel: false
  reuseValues: false
  skipDeps: false
  cascade: background
  insecureSkipTLSVerify: false

bases:
  - repos.yaml
  - templates/default-template.yaml
  - .helmfile-local/helmfile.yaml

missingFileHandler: Warn

environments:
  dev:
    values:
      - environments/global-env-defaults.yaml
      - environments/dev-values.yaml

---

releases:
  - name: splunk-operator-{{.Environment.Name}}
    namespace: "{{.StateValues.SPLUNK_OPERATOR_NAMESPACE}}"
    chart: splunk/splunk-operator
    version: "{{.StateValues.SPLUNK_OPERATOR_VERSION}}"
    inherit:
      - template: default

  - name: splunk-standalone-{{.Environment.Name}}
    namespace: "{{.StateValues.SPLUNK_NAMESPACE}}"
    chart: splunk/splunk-enterprise
    inherit:
      - template: default
```

### repos.yaml

Defines shared Helm repositories:

```yaml
repositories:
  - name: splunk
    url: https://splunk.github.io/splunk-operator/charts
```

### Templates

#### templates/default-template.yaml

A shared template that standardizes release configurations:

```yaml
templates:
  default:
    namespace: kube-system
    chart: stable/{{`{{.Release.Name}}`}}
    values:
      - releases/{{`{{.Release.Name}}`}}/values.yaml
      - releases/{{`{{.Release.Name}}`}}/{{`{{.Environment.Name}}`}}-values.yaml
    secrets:
      - releases/{{`{{.Release.Name}}`}}/secrets.yaml
      - releases/{{`{{.Release.Name}}`}}/{{`{{.Environment.Name}}`}}-secrets.yaml
    labels:
      environment: "{{`{{.Environment.Name}}`}}"
    missingFileHandler: Warn
```

### Environments

Environment-specific values and configurations are stored here:

* **global-env-defaults.yaml**: Global defaults applied to all environments.
* **dev-values.yaml**: Development environment overrides.
* **staging-values.yaml**: Staging environment overrides.
* **prod-values.yaml**: Production environment overrides.

### .helmfile-local

Personal configurations for individual developers or operators. This directory is ignored by version control (e.g., Git) and allows for local overrides.

#### Example .helmfile-local/helmfile.yaml

```yaml
environments:
  mymac:
    values:
      - environments/dev-values.yaml
      - .helmfile-local/mymac-values.yaml
    kubeContext: "devs"
```

Usage
-----

### Deploying to Development Environment

```bash
helmfile -e dev sync
```

### Deploying to Staging Environment

Uncomment the staging environment in `helmfile.yaml`:

```yaml
environments:
  staging:
    values:
      - environments/global-env-defaults.yaml
      - environments/staging-values.yaml
```

Then run:

```bash
helmfile -e staging sync
```

### Deploying to Production Environment

Uncomment the production environment in `helmfile.yaml`:

```yaml
environments:
  prod:
    values:
      - environments/global-env-defaults.yaml
      - environments/prod-values.yaml
```

Then run:

```bash
helmfile -e prod sync
```

### Using Personal Configurations

For personal overrides, add your configurations to `.helmfile-local/helmfile.yaml`:

```yaml
environments:
  your-environment-name:
    values:
      - environments/dev-values.yaml
      - .helmfile-local/your-values.yaml
    kubeContext: "your-kube-context"
```

Deploy using:

```bash
helmfile -e your-environment-name sync
```

Customization
-------------

### Adding New Releases

To add a new release, update the `releases` section in `helmfile.yaml`:

```yaml
- name: new-release-{{.Environment.Name}}
  namespace: "{{.StateValues.NEW_RELEASE_NAMESPACE}}"
  chart: your-chart-repo/your-chart-name
  version: "{{.StateValues.YOUR_CHART_VERSION}}"
  inherit:
    - template: default
```

### Overriding Values and Secrets

Place your custom values and secrets in the appropriate directories:

* Base values: `releases/<release-name>/values.yaml`
* Environment-specific values: `releases/<release-name>/<environment>-values.yaml`
* Base secrets: `releases/<release-name>/secrets.yaml`
* Environment-specific secrets: `releases/<release-name>/<environment>-secrets.yaml`

Troubleshooting
---------------

* **Missing Files**: If you encounter warnings about missing files, ensure that all referenced files exist or adjust the `missingFileHandler` setting.
* **Deployment Failures**: Check the Helm and Kubernetes logs for detailed error messages.
* **Timeouts**: If deployments time out, consider increasing the `timeout` value in `helmfile.yaml`.

Contributing
------------

Contributions are welcome! Please fork the repository and submit a pull request.

License
-------

This project is licensed under the MIT License.
