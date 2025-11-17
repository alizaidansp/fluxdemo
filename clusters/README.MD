# FluxCD Complete Guide

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Installing Flux CLI](#2-installing-flux-cli)
3. [Setting Up Kubernetes Cluster](#3-setting-up-kubernetes-cluster)
4. [Installing Flux Components](#4-installing-flux-components)
5. [Bootstrapping Flux](#5-bootstrapping-flux)
6. [Health Checks and Verification](#6-health-checks-and-verification)
7. [Monitoring Flux Sources and Kustomizations](#7-monitoring-flux-sources-and-kustomizations)
8. [Deploying Applications with Flux](#8-deploying-applications-with-flux)
9. [Working with Git Sources and Kustomizations](#9-working-with-git-sources-and-kustomizations)
10. [Syncing with Private GitHub Repositories](#10-syncing-with-private-github-repositories)
11. [Suspend and Resume Operations](#11-suspend-and-resume-operations)
12. [Understanding Prune Feature](#12-understanding-prune-feature)
13. [Deploying Artifacts to GitHub Container Registry](#13-deploying-artifacts-to-github-container-registry)
14. [Image Reflector and Automation Controllers](#14-image-reflector-and-automation-controllers)
15. [Troubleshooting Common Issues](#15-troubleshooting-common-issues)

---

## 1. Prerequisites

Before installing Flux, ensure you have:

- **Kubernetes cluster** (version 1.31.0 or higher for Flux 2.6.4+)
- **kubectl** configured to access your cluster
- **Git** installed and configured
- **GitHub account** (for bootstrapping with GitHub)
- **Personal Access Token (PAT)** with appropriate permissions (if working with private repos)

### Checking Prerequisites

```sh
# Check Kubernetes version
kubectl version --short

# Check if kubectl can access your cluster
kubectl cluster-info

# Check Git installation
git --version
```

---

## 2. Installing Flux CLI

The Flux CLI is a command-line tool that helps you interact with Flux controllers and manage your GitOps workflows.

### Installation Method

Install Flux CLI version 2.6.4 using the official installation script:

```sh
curl -s https://fluxcd.io/install.sh | sudo FLUX_VERSION=2.6.4 bash
```

**What this does:**

- Downloads the Flux CLI binary for your system
- Installs it to `/usr/local/bin/flux` (requires sudo)
- Makes it available system-wide

### Verify Installation

```sh
# Check Flux version
flux --version

# Get help
flux --help

# Check available commands
flux --help | grep -A 5 "Available Commands"
```

**Expected output:**

```
flux version 2.6.4
```

---

## 3. Setting Up Kubernetes Cluster

Flux requires a Kubernetes cluster. This guide uses Minikube for local development.

### Pre-installation Checks

Before installing Flux, verify your cluster meets the requirements:

```sh
flux check --pre
```

**What this checks:**

- Kubernetes version compatibility
- Cluster connectivity
- Required Kubernetes APIs availability
- RBAC permissions

**Example output (successful):**

```
✓ Kubernetes 1.31.0 >= 1.31.0-0
✓ kubectl 1.31.0 >= 1.31.0-0
✓ Kubernetes cluster "minikube" is ready
```

**Example output (if Kubernetes version is too old):**

```
✗ Kubernetes version v1.24.1 does not match >=1.31.0-0
```

### Upgrading Minikube Cluster

If your Kubernetes version is not supported, you need to upgrade your Minikube cluster:

**Step 1: Stop the current cluster**

```sh
minikube stop
```

**Step 2: Delete the old cluster**

```sh
minikube delete
```

**Step 3: Start a new cluster with compatible Kubernetes version**

```sh
minikube start --kubernetes-version=v1.31.0 --nodes=3 --container-runtime=containerd
```

**Parameters explained:**

- `--kubernetes-version=v1.31.0`: Specifies the Kubernetes version (must be >= 1.31.0 for Flux 2.6.4+)
- `--nodes=3`: Creates a multi-node cluster (optional, default is 1)
- `--container-runtime=containerd`: Uses containerd instead of Docker (required for newer K8s versions)

**Note:** Make sure to stop and delete the current Minikube cluster before creating a new one with a different Kubernetes version.

### Verify Cluster is Ready

```sh
# Check nodes
kubectl get nodes

# Expected output:
# NAME           STATUS   ROLES           AGE   VERSION
# minikube       Ready    control-plane   5m    v1.31.0
# minikube-m02   Ready    <none>          4m    v1.31.0
# minikube-m03   Ready    <none>          4m    v1.31.0
```

---

## 4. Installing Flux Components

Flux consists of several controllers that work together. You can install all components or select specific ones.

### Installing Specific Components

For a minimal setup, you might only need the source controller:

```sh
flux install --components=source-controller --export > flux-system.yaml
```

**What this does:**

- `--components=source-controller`: Installs only the source controller (for Git repositories)
- `--export`: Outputs the manifests to stdout instead of applying them
- `> flux-system.yaml`: Saves the manifests to a file

**Available components:**

- `source-controller`: Manages Git, Helm, and OCI sources
- `kustomize-controller`: Applies Kustomize overlays
- `helm-controller`: Manages Helm releases
- `notification-controller`: Sends notifications about Flux events
- `image-reflector-controller`: Scans container registries
- `image-automation-controller`: Updates image references

### Installing All Components

To install all Flux components:

```sh
flux install
```

This will:

- Create the `flux-system` namespace
- Install all Flux controllers as deployments
- Set up necessary RBAC resources
- Configure the controllers to work together

### Applying Exported Manifests

If you exported the manifests to a file:

```sh
kubectl apply -f flux-system.yaml
```

### Verify Components Installation

```sh
# Check if Flux pods are running
kubectl get pods -n flux-system

# Expected output:
# NAME                                      READY   STATUS    RESTARTS   AGE
# source-controller-xxxxxxxxxx-xxxxx        1/1     Running   0          2m
# kustomize-controller-xxxxxxxxxx-xxxxx     1/1     Running   0          2m
# helm-controller-xxxxxxxxxx-xxxxx          1/1     Running   0          2m
```

---

## 5. Bootstrapping Flux

Bootstrapping is the process of installing Flux and configuring it to manage itself from a Git repository. This creates a "GitOps" setup where Flux syncs its own configuration from Git.

### Bootstrap with GitHub

```sh
flux bootstrap github \
  --owner=alizaidansp \
  --repository=fluxdemo \
  --branch=main \
  --path=./clusters/fluxcluster \
  --personal=true \
  --private=false
```

**Parameters explained:**

- `--owner=alizaidansp`: Your GitHub username or organization
- `--repository=fluxdemo`: The repository name (will be created if it doesn't exist)
- `--branch=main`: The Git branch to use
- `--path=./clusters/fluxcluster`: Path within the repo where cluster configs are stored
- `--personal=true`: Use personal GitHub account (not organization)
- `--private=false`: Repository will be public (set to `true` for private repos)

**What bootstrap does:**

1. Creates the GitHub repository if it doesn't exist
2. Generates SSH keys for Flux to authenticate with GitHub
3. Adds the public key as a deploy key to the repository
4. Creates a GitRepository source pointing to your repo
5. Creates a Kustomization that applies manifests from the specified path
6. Commits the Flux configuration to the repository

**After bootstrapping:**

- Flux will continuously sync from your Git repository
- Any changes you push to the repo will be automatically applied to the cluster
- Flux manages itself - updates to Flux configs in Git will update Flux in the cluster

### Verify Bootstrap Success

```sh
# Check if Flux is syncing from Git
flux get sources git

# Check Flux system status
flux check
```

---

## 6. Health Checks and Verification

Regular health checks ensure Flux is working correctly and your cluster is in sync with Git.

### Basic Health Check

```sh
flux check
```

**What it checks:**

- All Flux controllers are running
- GitRepository sources are ready
- Kustomizations are applied successfully
- Cluster is in sync with Git

**Example output:**

```
► checking prerequisites
✗ flux 2.6.4 <2.7.3 (new CLI version is available, please upgrade)
✓ Kubernetes 1.31.0 >= 1.31.0-0
✓ kubectl 1.31.0 >= 1.31.0-0
✓ Kubernetes cluster "minikube" is ready

► checking controllers
✓ source-controller: deployment ready
✓ kustomize-controller: deployment ready

► checking crds
✓ all custom resource definitions are installed
```

### Pre-installation Check

Before installing Flux, check if prerequisites are met:

```sh
flux check --pre
```

This verifies:

- Kubernetes version compatibility
- Cluster connectivity
- Required APIs availability

### Viewing Flux Logs

Monitor Flux operations in real-time:

```sh
flux logs -f
```

**What you'll see:**

- Git repository reconciliation events
- Kustomization apply operations
- Errors and warnings
- Successful sync confirmations

**Use cases:**

- Debugging sync issues
- Monitoring deployment progress
- Understanding what Flux is doing

---

## 7. Monitoring Flux Sources and Kustomizations

### Checking Git Sources

Use `flux get sources git` to list all GitRepository sources and verify if new sources have been added:

```sh
flux get sources git
```

**What it shows:**

- All GitRepository sources in your cluster
- Their status (Ready/Not Ready)
- The last reconciled revision (commit hash)
- The branch being tracked
- Last reconciliation time

**Example output:**

```
NAME                      REVISION                              SUSPENDED   READY   MESSAGE
demo-3-source             main@sha1:fd11bc4e                    False       True     stored artifact for revision 'main@sha1:fd11bc4e'
flux-system               main@sha1:73e88108                    False       True     stored artifact for revision 'main@sha1:73e88108'
voting-app-source-demo    master@sha1:18e14da4                  False       True     stored artifact for revision 'master@sha1:18e14da4'
```

**Use cases:**

- Verify if a new GitRepository source has been successfully added after creating it
- Check if the source is syncing correctly with the latest commit
- Troubleshoot authentication issues (will show "Not Ready" status if there are problems)
- Verify commit hash matches between Git and Flux (ensures Flux has latest changes)

### Monitoring Kustomizations in Real-Time

Use `flux get kustomizations` (or the shorter alias `flux get ks`) to monitor if Flux is updating your cluster manifests in real-time:

```sh
flux get kustomizations
# or use the shorter alias:
flux get ks
```

**What it shows:**

- All Kustomization resources in your cluster
- Their reconciliation status (Ready/Not Ready/Progressing)
- The last applied revision
- Health status of the applied resources
- Last reconciliation time and next scheduled reconciliation

**Example output:**

```
NAME                          READY   MESSAGE                         REVISION                              SUSPENDED
flux-system                   True    Applied revision: main@sha1:8826fd28d3eb9f6f54172b9302ca876f01fbaef9   main@sha1:8826fd28d3eb9f6f54172b9302ca876f01fbaef9   False
voting-app-kustomization-demo True    Applied revision: master@sha1:8e7759aad41f9ed753d4bd067d051cd424065af3   master@sha1:8e7759aad41f9ed753d4bd067d051cd424065af3   False
```

**Use cases:**

- Monitor if Flux is applying changes from your Git repository to the cluster
- Verify that a new Kustomization has been created and is working
- Check the health status of deployed resources
- See when the next reconciliation will occur

**Watch mode (real-time updates):**

```sh
flux get ks -w
# or
watch flux get ks
```

This will continuously refresh the output, showing you updates as Flux reconciles your kustomizations.

---

## 8. Deploying Applications with Flux

Once Flux is bootstrapped, you can deploy applications by adding manifests to your Git repository.

### Workflow Overview

1. **Make changes in Git repository:**

   - Add new directories with Kubernetes manifests
   - Modify existing manifests
   - The path should match what's configured in your `gotk-sync.yaml` file (typically `./clusters/fluxcluster`)

2. **Commit and push changes:**

   ```sh
   git add .
   git commit -m "Add demo app manifests"
   git push origin main
   ```

3. **Flux automatically syncs:**

   - Flux detects the new commit
   - Downloads the updated manifests
   - Applies them to the cluster

4. **Verify deployment:**

   ```sh
   # Check if Flux detected the changes
   flux get sources git

   # Check if Kustomization applied the changes
   flux get ks

   # View Flux logs
   flux logs -f

   # Verify resources were created
   kubectl get all -n demoapp-ns
   ```

### Example: Deploying a Demo App

**Step 1: Create manifest files in your Git repo**

Structure:

```
clusters/fluxcluster/
└── demo1/
    ├── 1-demoapp-ns.yml
    ├── 2-demoapp-dep.yml
    └── 3-demoapp-svc.yml
```

**Step 2: Commit and push**

```sh
git add clusters/fluxcluster/demo1/
git commit -m "Add demo app"
git push
```

**Step 3: Monitor the deployment**

```sh
# Watch Flux logs
flux logs -f

# Check if commit hash matches
flux get sources git

# Verify resources
kubectl get all -n demoapp-ns
```

**Expected output:**

```
NAME                          READY   STATUS    RESTARTS   AGE
pod/demodep-7f6b6cd9d-22vdx   1/1     Running   0          5m59s
pod/demodep-7f6b6cd9d-f4cjk   1/1     Running   0          5m59s
pod/demodep-7f6b6cd9d-svpgs   1/1     Running   0          5m59s

NAME              TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/demosvc  NodePort   10.105.116.225  <none>        80:31000/TCP   5m59s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demodep   3/3     3            3           5m59s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/demodep-7f6b6cd9d   3         3         3       5m59s
```

**Important:** Always verify that the commit hash shown by `flux get sources git` matches the latest commit in your GitHub repository. This ensures Flux has the latest changes.

---

## 9. Working with Git Sources and Kustomizations

### Creating a Git Source

A GitRepository source tells Flux where to find your Kubernetes manifests.

```sh
flux create source git voting-app-source-demo \
  --url=https://github.com/yogeshraheja/KubernetesVotingApp.git \
  --branch=main \
  --export > ./fluxdemo/clusters/fluxcluster/demo2/voting-app-source-demo.yaml
```

**Parameters:**

- `--url`: The Git repository URL
- `--branch`: Branch to track (default: main)
- `--export`: Output to file instead of applying directly
- `> filename.yaml`: Save to file

**Common error: Directory doesn't exist**

If you see:

```
zsh: no such file or directory: ./fluxdemo/clusters/fluxcluster/demo2/voting-app-source-demo.yaml
```

**Solution:** Create the directory first:

```sh
mkdir -p ./fluxdemo/clusters/fluxcluster/demo2
```

Then run the create command again.

### Creating a Kustomization

A Kustomization tells Flux which path in the Git repository to apply to the cluster.

```sh
flux create kustomization voting-app-kustomization-demo \
  --source=GitRepository/voting-app-source-demo \
  --path="./" \
  --prune=true \
  --export > ./fluxdemo/clusters/fluxcluster/demo2/voting-app-kustomization-demo.yaml
```

**Parameters:**

- `--source`: Reference to the GitRepository (format: `Kind/name`)
- `--path`: Path within the Git repo to apply (relative to repo root)
- `--prune`: Delete resources removed from Git (see [Understanding Prune Feature](#12-understanding-prune-feature))
- `--export`: Output to file

**After creating:**

1. Commit both files to your Git repository
2. Push to trigger Flux sync
3. Verify with `flux get sources git` and `flux get ks`

---

## 10. Syncing with Private GitHub Repositories

To sync Flux with a private GitHub repository, you need to provide authentication credentials via a Kubernetes secret.

### Step 1: Create a GitHub Personal Access Token (PAT)

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token (classic)
3. Select scopes:
   - `repo` (full control of private repositories)
   - Or at minimum: `public_repo` for public repos
4. Copy the token (you won't see it again!)

### Step 2: Check Existing Secrets

```sh
kubectl get secrets -n flux-system
```

### Step 3: Create the Secret

```sh
kubectl create secret generic devopsinaction-secret \
  --from-literal=username=alizaidansp \
  --from-literal=password=<gh-pat-token> \
  -n flux-system
```

**Replace:**

- `alizaidansp` with your GitHub username
- `<gh-pat-token>` with your actual PAT

**Verify the secret:**

```sh
kubectl get secrets -n flux-system
```

**Expected output:**

```
NAME                    TYPE     DATA   AGE
devopsinaction-secret   Opaque   2      3s
flux-system             Opaque   3      21h
```

### Step 4: Reference the Secret in GitRepository

In your GitRepository manifest, add the `secretRef`:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: voting-app-source-demo
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: master
  url: https://github.com/alizaidansp/KubernetesVotingApp.git
  secretRef:
    name: devopsinaction-secret # Reference the secret here
```

### Step 5: Apply and Verify

```sh
# Apply the GitRepository
kubectl apply -f voting-app-source-demo.yaml

# Check if authentication works
flux get sources git

# Watch logs for any auth errors
flux logs -f
```

**Troubleshooting auth errors:**

- If you see "authentication required: Repository not found", check:
  - Secret name matches `secretRef.name`
  - PAT has correct permissions
  - Username is correct
  - Repository URL is correct

**Note:** After fixing authentication issues, Flux will automatically retry. You might need to wait for the next reconciliation interval or trigger a manual reconciliation.

---

## 11. Suspend and Resume Operations

You can temporarily pause Flux operations without deleting resources.

### Suspend a Source

Stop Flux from reconciling a specific GitRepository:

```sh
flux suspend source git flux-system
```

**Effect:**

- Flux will not fetch updates from the Git repository
- Existing resources remain in the cluster
- If you push changes to Git, Flux won't reconcile them
- Useful for maintenance or troubleshooting

**Verify suspension:**

```sh
flux get sources git
# The SUSPENDED column will show "True"
```

### Resume a Source

Re-enable reconciliation:

```sh
flux resume source git flux-system
```

**Effect:**

- Flux resumes fetching and applying changes from Git
- Next reconciliation happens at the configured interval

### Suspend a Kustomization

Stop Flux from applying a specific Kustomization:

```sh
flux suspend kustomization voting-app-kustomization-demo
```

### Resume a Kustomization

```sh
flux resume kustomization voting-app-kustomization-demo
```

---

## 12. Understanding Prune Feature

The `prune` feature ensures your cluster matches your Git repository exactly.

### What is Prune?

**Without `prune: true`:**

- Flux applies resources from Git to the cluster
- If you remove a resource from Git, Flux **does NOT** delete it from the cluster
- This can cause drift between Git (desired state) and cluster (actual state)

**With `prune: true`:**

- Flux applies resources from Git
- If you remove a resource from Git, Flux **will delete** it from the cluster
- Keeps cluster in sync with Git (desired state)

### Example Scenario

1. **Initial state:** You push a deployment manifest to Git

   - Flux creates the deployment in the cluster ✓

2. **Remove from Git:** You delete the deployment manifest from Git

   - **Without prune:** Deployment still exists in cluster ✗
   - **With prune:** Flux deletes the deployment from cluster ✓

3. **Result:**
   - Without prune: Cluster has resources not in Git (drift)
   - With prune: Cluster matches Git exactly (no drift)

### Setting Prune in Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-kustomization
  namespace: flux-system
spec:
  prune: true # Enable pruning
  interval: 1m0s
  path: ./
  sourceRef:
    kind: GitRepository
    name: my-source
```

**Important:** Prune only affects resources managed by that specific Kustomization. Resources created outside Flux or by other Kustomizations are not affected.

---

## 13. Deploying Artifacts to GitHub Container Registry

You can package Kubernetes manifests as OCI artifacts and push them to GitHub Container Registry (ghcr.io).

### Step 1: Ensure Prerequisites

**Check Helm (optional, for charts):**

```sh
helm version
```

**Login to GitHub Container Registry:**

```sh
docker login ghcr.io
```

- **Username:** Your GitHub username (e.g., `alizaidansp`)
- **Password:** Your GitHub Personal Access Token (PAT)

**Note:** Docker stores credentials in `~/.docker/config.json`, possibly unencrypted. Consider using a credential helper for better security. See [Docker credential store docs](https://docs.docker.com/go/credential-store/).

### Step 2: Clone and Prepare Repository

```sh
git clone https://github.com/yogeshraheja/ocidemo.git
cd ocidemo
ls   # You should see: charts  manifests  README.md
cd manifests
ls   # You should see: 01_demoapp_ns.yaml  02_demoapp_dep.yaml  03_demoapp_svc.yaml
```

### Step 3: Push Manifests as OCI Artifact

**Basic command:**

```sh
flux push artifact oci://ghcr.io/alizaidansp/ocidemo-manifests:$(git rev-parse --short HEAD) \
  --path=./ \
  --revision=$(git branch --show-current)@sha1:$(git rev-parse HEAD)
```

**If you see: `✗ --source is required`**

Add the `--source` flag:

```sh
flux push artifact oci://ghcr.io/alizaidansp/ocidemo-manifests:$(git rev-parse --short HEAD) \
  --path=./ \
  --source=$(git config --get remote.origin.url) \
  --revision=$(git branch --show-current)@sha1:$(git rev-parse HEAD)
```

**Parameters explained:**

- `oci://ghcr.io/alizaidansp/ocidemo-manifests:...`: OCI registry URL with tag
- `$(git rev-parse --short HEAD)`: Uses short commit hash as tag
- `--path=./`: Path to manifests directory
- `--source`: Git repository URL (required)
- `--revision`: Full revision reference

### Troubleshooting Permission Errors

**Error:**

```
✗ pushing artifact failed: POST https://ghcr.io/v2/alizaidansp/ocidemo-manifests/blobs/uploads/: DENIED: permission_denied: The token provided does not match expected scopes.
```

**Solution:** Your PAT lacks required permissions. Ensure it has:

- `write:packages` – Upload packages to GitHub Package Registry
- `read:packages` – Download packages from GitHub Package Registry
- `delete:packages` – Delete packages from GitHub Package Registry

**Steps:**

1. Go to GitHub → Settings → Developer settings → Personal access tokens
2. Edit or create a new token with the above scopes
3. Update your Docker login with the new token:
   ```sh
   docker login ghcr.io
   ```
4. Retry the push command

---

## 14. Troubleshooting Common Issues

### Issue: Kubernetes Version Not Supported

**Error:**

```
✗ Kubernetes version v1.24.1 does not match >=1.31.0-0
```

**Solution:** Upgrade your cluster (see [Setting Up Kubernetes Cluster](#3-setting-up-kubernetes-cluster))

### Issue: Directory Not Found When Creating Sources

**Error:**

```
zsh: no such file or directory: ./fluxdemo/clusters/fluxcluster/demo2/voting-app-source-demo.yaml
```

**Solution:**

```sh
mkdir -p ./fluxdemo/clusters/fluxcluster/demo2
```

### Issue: Authentication Required for Private Repos

**Error in logs:**

```
error GitRepository/voting-app-source-demo.flux-system - Reconciler error failed to checkout and determine revision: unable to list remote for 'https://github.com/...': authentication required: Repository not found.
```

**Solution:** See [Syncing with Private GitHub Repositories](#10-syncing-with-private-github-repositories)

### Issue: Flux Not Detecting New Commits

**Symptoms:**

- You push changes to Git
- `flux get sources git` shows old commit hash
- Resources not updating in cluster

**Solutions:**

1. Check if source is suspended:

   ```sh
   flux get sources git
   # If SUSPENDED is True, resume it:
   flux resume source git <source-name>
   ```

2. Check reconciliation interval (might be long):

   ```sh
   kubectl get gitrepository <source-name> -n flux-system -o yaml
   # Look for spec.interval
   ```

3. Force reconciliation:
   ```sh
   flux reconcile source git <source-name>
   ```

### Issue: Resources Not Being Deleted (Prune Not Working)

**Symptoms:**

- Removed resources from Git
- Resources still exist in cluster

**Solutions:**

1. Verify prune is enabled:

   ```sh
   kubectl get kustomization <name> -n flux-system -o yaml | grep prune
   ```

2. Ensure resource was created by Flux (has proper labels/annotations)

3. Check Kustomization status:
   ```sh
   flux get ks <name>
   ```

### Issue: Notification Controller Connection Refused

**Error in logs:**

```
error Kustomization/... - unable to record event POST http://notification-controller.flux-system.svc.cluster.local./ giving up after 5 attempt(s): Post "http://notification-controller.flux-system.svc.cluster.local./": dial tcp 10.101.97.136:80: connect: connection refused
```

**Note:** This is a warning, not a critical error. Flux operations continue normally. The notification controller is optional and only used for sending alerts/notifications.

**Solution (optional):** Install notification controller if you need notifications:

```sh
flux install --components=notification-controller
```

---

## Quick Reference Commands

```sh
# Installation
flux --version
flux check --pre
flux install

# Bootstrap
flux bootstrap github --owner=<user> --repository=<repo> --path=./clusters/<cluster-name>

# Health Checks
flux check
flux get sources git
flux get ks
flux logs -f

# Create Resources
flux create source git <name> --url=<url> --branch=<branch> --export > <file>.yaml
flux create kustomization <name> --source=GitRepository/<source> --path=./ --prune=true --export > <file>.yaml

# Control Operations
flux suspend source git <name>
flux resume source git <name>
flux suspend kustomization <name>
flux resume kustomization <name>

# Image Management
flux get image all
flux get image policy
flux get image update
flux create image repository <name> --image=<image> --interval=1m0s --export > <file>.yaml
flux create image policy <name> --image-ref=<repo> --select-semver=1.x.x --export > <file>.yaml

# Troubleshooting
kubectl get pods -n flux-system
kubectl get gitrepository -n flux-system
kubectl get kustomization -n flux-system
kubectl get imagerepository -n flux-system
kubectl get imagepolicy -n flux-system
```

---

## 14. Image Reflector and Automation Controllers

Flux includes two controllers for automating container image updates:

- **Image Reflector Controller**: Scans container registries and discovers available image tags
- **Image Automation Controller**: Automatically updates image references in your Git repository

This enables fully automated image updates based on policies you define.

### Prerequisites

Ensure the image controllers are installed:

```sh
# Check if controllers are running
kubectl get pods -n flux-system | grep image

# Expected output:
# image-reflector-controller-xxx    1/1     Running
# image-automation-controller-xxx  1/1     Running
```

If not installed, add them:

```sh
flux install --components=image-reflector-controller,image-automation-controller
```

### Workflow Overview

1. Create Kubernetes manifests with image references
2. Create an ImageRepository to scan the container registry
3. Create an ImagePolicy to define which tags to select
4. (Optional) Create ImageUpdateAutomation to automatically update Git

### Step-by-Step Guide

#### Step 1: Create Application Manifests

First, create your application manifests in your Git repository. The deployment manifest should specify the image tag you want to use:

**Example: `demo9/manifests/2-demoapp-dep.yml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demoapp
  name: demodep
  namespace: demoapp-9-ns
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demoapp
  template:
    metadata:
      labels:
        app: demoapp
    spec:
      containers:
        - image: yogeshraheja/fluxdemo:1.0.0 # This tag will be updated automatically
          name: demoapp
```

**Commit and push to Git:**

```sh
git add demo9/manifests/
git commit -m "Add demo9 application manifests"
git push origin main
```

#### Step 2: Verify Git Sync

Ensure Flux has synced the latest changes:

```sh
# Check if commit hash matches
flux get sources git

# Watch logs to confirm sync
flux logs -f
```

**Verify the commit hash shown by Flux matches your latest commit in GitHub.**

#### Step 3: Create ImageRepository

An ImageRepository tells Flux which container registry to scan for available image tags.

```sh
flux create image repository demoapp-9-image \
  --image=yogeshraheja/fluxdemo \
  --interval=1m0s \
  --export > demo9/demo-9-image.yaml
```

**Parameters:**

- `--image`: Container image name (without tag)
- `--interval`: How often to scan for new tags (default: 1h)
- `--export`: Output to file instead of applying directly

**Example output file: `demo9/demo-9-image.yaml`**

```yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: demoapp-9-image
  namespace: flux-system
spec:
  image: yogeshraheja/fluxdemo
  interval: 1m0s
```

**Commit and apply:**

```sh
git add demo9/demo-9-image.yaml
git commit -m "Add ImageRepository for demoapp"
git push origin main

# Force reconciliation to apply immediately
flux reconcile source git flux-system
```

#### Step 4: Verify ImageRepository is Working

Check if the ImageRepository is scanning successfully:

```sh
# List all ImageRepositories
flux get image all

# Or check specific repository
kubectl get imagerepository demoapp-9-image -n flux-system
```

**Expected output:**

```
NAME                            LAST SCAN               SUSPENDED   READY   MESSAGE
imagerepository/demoapp-9-image 2025-11-17T09:40:14Z    False       True    successful scan: found 2 tags
```

**View discovered tags:**

```sh
kubectl describe imagerepository demoapp-9-image -n flux-system
```

This shows all available tags found in the container registry.

#### Step 5: Create ImagePolicy

An ImagePolicy defines which image tags should be selected from the ImageRepository.

```sh
flux create image policy demoapp-9-image-policy \
  --image-ref=demoapp-9-image \
  --select-semver=1.x.x \
  --export > demo9/demo-9-image-policy.yaml
```

**Parameters:**

- `--image-ref`: Reference to the ImageRepository
- `--select-semver`: Semantic version range (e.g., `1.x.x`, `>=1.0.0 <2.0.0`)
- `--export`: Output to file

**Semantic version examples:**

- `1.x.x` - Latest 1.x version
- `>=1.0.0 <2.0.0` - Any 1.x version
- `>=1.0.0` - Version 1.0.0 or higher
- `latest` - Always use the latest tag

**Example output file: `demo9/demo-9-image-policy.yaml`**

```yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: demoapp-9-image-policy
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: demoapp-9-image
  policy:
    semver:
      range: 1.x.x
```

**Commit and apply:**

```sh
git add demo9/demo-9-image-policy.yaml
git commit -m "Add ImagePolicy for demoapp"
git push origin main

# Force reconciliation
flux reconcile source git flux-system
```

#### Step 6: Verify ImagePolicy

Check the selected image:

```sh
# List all ImagePolicies
flux get image policy

# Check specific policy
kubectl get imagepolicy demoapp-9-image-policy -n flux-system -o yaml
```

**The ImagePolicy will show the latest tag that matches your semver range.**

### Using ImageUpdateAutomation (Optional)

To automatically update your Git repository when new images are available, create an ImageUpdateAutomation:

```sh
flux create image update demoapp9-image-update \
  --git-repo-ref=flux-system \
  --git-repo-path=./clusters/fluxcluster \
  --checkout-branch=main \
  --author-name=alizaidansp \
  --author-email=alizaidansp@gmail.com \
  --push-branch=main \
  --export > demo9/demo-9-automation.yaml
```

**Parameters:**

- `--git-repo-ref`: Reference to the GitRepository (usually `flux-system`)
- `--git-repo-path`: Path within the repo where manifests are located
- `--checkout-branch`: Branch to checkout and update
- `--author-name`: Git commit author name
- `--author-email`: Git commit author email
- `--push-branch`: Branch to push changes to
- `--export`: Output to file

**Commit and apply:**

```sh
git add demo9/demo-9-automation.yaml
git commit -m "Add ImageUpdateAutomation for demoapp"
git push origin main

# Force reconciliation to apply immediately
flux reconcile source git flux-system
```

**This will:**

- Monitor the ImagePolicy for new images
- Automatically update the image tag in your deployment manifests (those with `{"$imagepolicy": "demoapp-9-image-policy"}` annotation)
- Commit and push changes to Git
- Trigger Flux to apply the updated manifests

### Complete Automation Flow

Here's the complete flow showing how all components work together:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. ImageRepository                                          │
│    Scans: yogeshraheja/fluxdemo                             │
│    Finds: [1.0.0, 1.0.1, 1.1.0, 2.0.0, latest]           │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. ImagePolicy                                               │
│    Pattern: 1.x.x                                           │
│    Selects: 1.1.0 (latest 1.x version)                       │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. ImageUpdateAutomation                                    │
│    Detects: Policy selected 1.1.0 (newer than current 1.0.0)│
│    Updates Git: Changes manifest from 1.0.0 → 1.1.0         │
│    Commits & Pushes: "Update image to 1.1.0"                 │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Flux Kustomization                                       │
│    Detects: Git commit with new manifest                    │
│    Applies: Updated deployment to cluster                   │
│    Result: Pods restart with new image 1.1.0               │
└─────────────────────────────────────────────────────────────┘
```

**Step-by-step what happens:**

1. **ImageRepository** scans the container registry every minute and discovers all available tags
2. **ImagePolicy** evaluates the discovered tags against the semver pattern (`1.x.x`) and selects the latest matching tag
3. **ImageUpdateAutomation** detects when the policy selects a new tag that's different from what's currently in Git, then:
   - Checks out your Git repository
   - Finds manifests with the `{"$imagepolicy": "demoapp-9-image-policy"}` annotation
   - Updates the image tag in those manifests
   - Commits the change with your configured author info
   - Pushes to the repository
4. **Flux Kustomization** detects the new Git commit, downloads the updated manifests, and applies them to the cluster, causing pods to restart with the new image

### Verification Commands

```sh
# Check ImageRepository status
flux get image all
kubectl get imagerepository -n flux-system

# Check ImagePolicy status
flux get image policy
kubectl get imagepolicy -n flux-system

# Check ImageUpdateAutomation status
flux get image update
kubectl get imageupdateautomation -n flux-system

# View discovered tags
kubectl describe imagerepository <name> -n flux-system

# View selected image
kubectl describe imagepolicy <name> -n flux-system
```

### Troubleshooting Image Controllers

**Issue: ImageRepository shows "waiting to be reconciled"**

```sh
# Check if image-reflector-controller is running
kubectl get pods -n flux-system | grep image-reflector

# If pod is crashing, check logs
kubectl logs -n flux-system -l app.kubernetes.io/name=image-reflector-controller

# Restart the pod if needed
kubectl delete pod -n flux-system -l app.kubernetes.io/name=image-reflector-controller
```

**Issue: ImageRepository can't find tags**

- Verify the image name is correct (check Docker Hub)
- Ensure the image is public or provide authentication
- Check controller logs for authentication errors

**Issue: ImagePolicy not selecting expected tag**

- Verify semver range matches available tags
- Check ImageRepository has scanned successfully
- View ImagePolicy status to see selected image

---
