### README.md

# Helmfile-based Kubernetes Deployment

This repository contains a Helmfile-based configuration to deploy various components in a Kubernetes cluster, including distributed storage using Longhorn, a WordPress blog, and Cloudflare Tunnel management using the Cloudflare Operator.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
   - [Setting up a K3s cluster](#setting-up-a-k3s-cluster)
   - [Installing Helmfile](#installing-helmfile)
   - [Creating Required Secrets](#creating-required-secrets)
   - [Deploying Helmfile Configuration](#deploying-helmfile-configuration)
3. [Repositories](#repositories)
4. [Releases](#releases)
   - [Longhorn](#longhorn)
   - [WordPress Blog](#wordpress-blog)
5. [Hooks](#hooks)
   - [Cloudflare Operator Setup](#cloudflare-operator-setup)
   - [Custom Kustomize Resources](#custom-kustomize-resources)
6. [Cloudflare Secrets](#cloudflare-secrets)
7. [Cloudflared Overview](#cloudflared-overview)
8. [Obtaining Cloudflare API Tokens](#obtaining-cloudflare-api-tokens)

---

## Prerequisites

Before you begin, ensure you have the following installed:

- **K3s** (or any Kubernetes cluster)
- **kubectl**: Command-line tool for interacting with Kubernetes clusters.
- **Helm**: Kubernetes package manager.
- **Helmfile**: A declarative spec for deploying Helm charts.
- **Kustomize**: Kubernetes-native configuration management tool.
- **Cloudflare account** with API tokens (detailed in [Cloudflare Secrets](#cloudflare-secrets)).

---

## Installation

### Setting up a K3s cluster

[K3s](https://k3s.io/) is a lightweight Kubernetes distribution designed for small deployments and IoT devices. You can easily set it up on any Linux machine or virtual environment.

1. **Install K3s**:

   On a Linux server or virtual machine, run the following command to install K3s:

   ```bash
   curl -sfL https://get.k3s.io | sh -
   ```

   This will install K3s and start the Kubernetes control plane.

2. **Verify Installation**:

   After installation, ensure the `kubectl` command is available:

   ```bash
   kubectl get nodes
   ```

   This command should return the list of nodes in your cluster (in this case, your single node).

3. **Access K3s cluster**:

   The `kubeconfig` file for K3s is stored at `/etc/rancher/k3s/k3s.yaml`. You can set the `KUBECONFIG` environment variable to point to this file:

   ```bash
   export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
   ```

### Installing Helmfile

1. **Install Helm**:

   Helmfile requires Helm to deploy charts. If you haven't installed Helm yet, follow these steps:

   - For Linux/macOS:
     ```bash
     curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
     ```

   - For Windows: Download the [Helm binary](https://helm.sh/docs/intro/install/) and follow the installation instructions.

2. **Install Helmfile**:

   You can install Helmfile using a package manager like Homebrew, or by downloading the binary:

   - **Linux/macOS**:
     ```bash
     brew install helmfile
     ```

   - **Manual Installation**:
     Download the binary from the [Helmfile releases](https://github.com/helmfile/helmfile/releases) page and move it to a directory in your `PATH`.

3. **Verify Helmfile Installation**:

   After installation, verify that Helmfile is working:

   ```bash
   helmfile --version
   ```

### Creating Required Secrets

Before you deploy Helmfile, you need to create the following secrets for **Cloudflare**, **Longhorn**, and **WordPress** authentication and configuration.

#### Cloudflare Secrets

Run the following command to create the required secrets for the Cloudflare Operator:

```bash
kubectl -n cloudflare-operator-system create secret generic cloudflare-secrets \
  --from-literal=CLOUDFLARE_API_TOKEN=***** \
  --from-literal=CLOUDFLARE_API_KEY=****
```

- Replace `*****` with your actual **Cloudflare API Token** (used for creating and managing tunnels).
- Replace `****` with your actual **Cloudflare Global API Key** (used for managing DNS entries and deleting tunnels).

#### Longhorn Authentication Secret

If you want to enable authentication on the Longhorn Dashboard, you will need to create a Kubernetes secret to store the admin password:

```bash
kubectl -n longhorn-system create secret generic longhorn-credentials \
  --from-literal=LONGHORN_USERNAME=admin \
  --from-literal=LONGHORN_PASSWORD=<your-password>
```

Replace `<your-password>` with your desired admin password for the Longhorn dashboard.

#### WordPress Secrets

You also need to create a secret for WordPress and MariaDB credentials:

```bash
kubectl create secret generic wordpress-secrets \
  --from-literal=wordpress-password=**** \
  --from-literal=mariadb-root-password=**** \
  --from-literal=mariadb-password=**** \
  --namespace wordpress-blog
```

Replace `****` with the appropriate passwords for WordPress and MariaDB.

### Deploying Helmfile Configuration

Once you have created the required secrets, you can proceed to deploy the Helmfile configuration.

1. Clone the repository:

   ```bash
   git clone <your-repo-url>
   cd <your-repo-directory>
   ```

2. Deploy the Helmfile configuration:

   ```bash
   helmfile sync
   ```

   This command will apply all the Helmfile configurations, including deploying Longhorn, WordPress, and Cloudflare Tunnel bindings.

---

## Repositories

The following Helm repositories are used in this setup:

- **Longhorn**: [https://charts.longhorn.io](https://charts.longhorn.io)
- **Bitnami**: [https://charts.bitnami.com/bitnami](https://charts.bitnami.com/bitnami)
- **Cloudflare**: [https://cloudflare.github.io/helm-charts](https://cloudflare.github.io/helm-charts)

---

## Releases

### Longhorn

**Longhorn** is a distributed block storage system for Kubernetes. It is deployed to the `longhorn-system` namespace using the Longhorn Helm chart.

```yaml
- name: longhorn
  namespace: longhorn-system
  chart: longhorn/longhorn
  version: 1.7.1
```

### WordPress Blog

The **WordPress Blog** is deployed using Bitnami's WordPress Helm chart in the `wordpress-blog` namespace. After the deployment, a Kubernetes hook applies a custom Ingress configuration to expose the blog.

```yaml
- name: wordpress
  namespace: wordpress-blog
  chart: bitnami/wordpress
  version: 23.1.17
  values:
    - values/wordpress-blog.yaml
  hooks:
    - events: ["postSync"]
      showlogs: true
      command: "kubectl"
      args:
        - apply
        - -f
        - "ingress/wordpress-blog.yaml"
```

This hook ensures that the custom Ingress for the WordPress blog is applied after the chart is deployed.

---

## Hooks

### Cloudflare Operator Setup

Before deploying the applications, the **Cloudflare Operator** is set up to manage Cloudflare Tunnel configurations and DNS entries for the Kubernetes services.

```yaml
hooks:
  - events: ["prepare"]
    showlogs: true
    command: kubectl
    args:
      - apply
      - -k
      - "https://github.com/adyanth/cloudflare-operator/config/default"
```

This hook installs the Cloudflare Operator, which handles the interaction between Kubernetes and Cloudflare's infrastructure.

### Custom Kustomize Resources

Additionally, custom Kustomize resources are applied to the cluster using the following hook:

```yaml
hooks:
  - events: ["prepare"]
    showlogs: true
    command: kubectl
    args:
      - apply
      - -k
      - "../kustomize"
```

This hook applies any additional resources from the local Kustomize directory.

---

## Cloudflare Secrets

To allow the Cloudflare Operator to interact with Cloudflare, you need to create secrets that contain your **Cloudflare API Token** and **Global API Key**. Run the following command to create the required secrets:

```bash
kubectl -n cloudflare-operator-system create secret generic cloudflare-secrets \
  --from-literal=CLOUDFLARE_API_TOKEN=***** \
  --from-literal=CLOUDFLARE_API_KEY=****
```

Replace `*****` with your actual API token and API key.

---

## Cloudflared Overview

**Cloudflared** is used to securely expose Kubernetes services to the internet by creating Cloudflare Tunnels. These tunnels allow secure access without exposing public IPs or relying on traditional ingress controllers.

The Cloudflare Operator manages tunnels for services running in Kubernetes, including creating DNS entries for them in Cloudflare and managing the tunnel lifecycle.

- **CLOUDFLARE_API_TOKEN**: Required for creating and managing tunnels.
- **CLOUDFLARE_API_KEY**: Required for deleting DNS entries and tunnels.

---

## Obtaining Cloudflare API Tokens

To allow the Cloudflare Operator to interact with the Cloudflare API, you'll need two tokens: **API Token** and **Global API Key**.

### How to Get the Tokens

1. **CLOUDFLARE_API_TOKEN**:
   - Go to **My Profile > API Tokens** in the Cloudflare Dashboard.
   - Create a new **Custom Token** with the following permissions:
      - **Account > Cloudflare Tunnel > Edit**: To create new tunnels.
      - **Account > Account Settings > Read**: To retrieve the `accountId` and `domainId`.
      - **Zone > DNS > Edit**: To manage DNS entries.
   - In **Account Resources**, include **All accounts**.
   - In **Zone Resources**, include **All zones**.

2. **CLOUDFLARE_API_KEY** (Global API Key):
   - Go to **My Profile > API Tokens**.
   - Copy the **Global API Key** from the bottom of the page. This is used to delete DNS entries and tunnels when resources are deleted.

These tokens ensure that the operator can both create and delete resources in Cloudflare as needed.

