# TFDS SIMPL-open Data Provider Agent

This repository contains the deployment manifests and configurations for the **SIMPL-Open Data Provider** agent, heavily adapted and optimized for local, single-node Kubernetes (k3s) environments.

## 🚀 Quick Start Deployment Guide

This guide is designed for users who want to quickly deploy the Data Provider agent using **ArgoCD**. It assumes you are deploying to a local, single-node k3s cluster.

### Prerequisites

Before deploying the Data Provider, ensure your environment meets the following requirements:
1. **Running k3s Cluster:** A single-node cluster with `MetalLB` and an `nginx` Ingress Controller.
2. **Common Components Deployed:** The core SIMPL infrastructure (Kafka, PostgreSQL, Elastic, OpenBao) must already be running in the `common` namespace.
3. **DNS Routing:** A wildcard DNS record (e.g., `*.dataprovider.yourdomain.com`) pointing to your MetalLB public IP.
4. **Governance Authority:** You need the domain name of the external Governance Authority cluster you will federate with (e.g., `ds.helsinki.tfds.io`).

---

### Step 1: Configure the ArgoCD Manifest

All configuration for the deployment is managed through a single file: `ArgoCD/data-provider_manifest.yaml`.

Open this file and update the `values` block to match your environment:

1. **Namespace Tags:** Update the identifiers for your namespaces:
   ```yaml
   namespaceTag:
     dataprovider: dataprovider     # Your data provider namespace
     authority: authority           # The governance authority namespace
     common: common                 # Your common components namespace
   ```
2. **Domain Federation:** Update the domain suffixes to ensure secure cross-cluster communication:
   ```yaml
   domainSuffix: idea.helsinki.tfds.io           # Your local cluster's base domain
   authorityDomainSuffix: ds.helsinki.tfds.io    # The external Governance Authority's base domain
   ```
3. **Cloud Provisioning (Crossplane):** By default, this is set to `enabled: false`. **Leave this as false** for local k3s deployments.
   * *What it is:* This module (including Gitea, FluxCD, Argo Events, and `infrastructure-be`) is designed exclusively to auto-provision virtual machines and workloads on external OVH or IONOS clouds.
   * *The Impact:* Enabling this on a vanilla local k3s node will cause catastrophic initialization failures (e.g., `argo-events` crashing due to OS `inotify` limits, and GitOps pipelines failing to authenticate). Disabling it ensures a clean, lightweight deployment of the core data agent.
4. **Monitoring:** By default, this is set to `enabled: false`.
   * *What it is:* This controls whether the agent's Java applications attempt to ship OpenTelemetry metrics and traces to the Elastic stack in the Common Components namespace.
   * *The Impact:* If you are not running the heavy Elastic monitoring stack in your cluster, leaving this disabled is correct. You may still see occasional `[otel.javaagent] WARN` messages in the pod logs complaining about a 404 error when trying to reach the collector. These logs are harmless, fire-and-forget telemetry drops and will not impact the performance or stability of the Data Provider.

---

### Step 2: Pre-Deployment Secrets (OpenBao)

Before you deploy the agent, you **must** manually inject your MinIO (S3) credentials into the OpenBao (Vault) server. The Eclipse Dataspace Connector (EDC) requires these to boot properly.

1. Log in to your OpenBao UI (e.g., `https://secrets.common.yourdomain.com`) using your Root Token.
2. Navigate to the Key/Value secrets engine.
3. Locate or create the secret named `<your-dataprovider-namespace>-simpl-edc` (e.g., `dataprovider-simpl-edc`).
4. Add the following keys with your specific MinIO details:
   * `fr_gxfs_s3_access_key`: Your MinIO Access Key (e.g., `admin`)
   * `fr_gxfs_s3_secret_key`: Your MinIO Secret Key
   * `fr_gxfs_s3_endpoint`: Your MinIO API URL (e.g., `https://s3.yourdomain.com`)

---

### Step 3: Deploy the Agent

Once your configuration is set and your secrets are in OpenBao, you can trigger the deployment using ArgoCD:

```bash
kubectl apply -f ArgoCD/data-provider_manifest.yaml
```

ArgoCD will automatically read the configuration and begin spinning up the Data Provider agent in the specified namespace.

---

### Step 4: Expected Behavior & Participant Onboarding

**⚠️ IMPORTANT: Please read this before troubleshooting!**

When you monitor the deployment in ArgoCD or via `kubectl get pods`, you will notice that three specific components appear to be "stuck" or "crashing":

*   🔴 **`tier2-gateway`:** Stuck in `Init:0/1`
*   🔴 **`tier2-proxy`:** Stuck in `Init:0/1`
*   🔴 **`simpl-edc`:** Crashing (`CrashLoopBackOff` / `NullPointerException`)

**This is 100% expected behavior.**

These components act as the secure trust gateways for the data space. They are intentionally stalling because they are waiting for business-level certificates from the Governance Authority.

**How to fix them:**
You must complete the manual **Participant Onboarding** process through the newly deployed Data Provider UI. Once the agent is officially onboarded and the certificates are generated and synced, these pods will automatically recover, initialize, and turn green.

*Reference the official [SIMPL Open Onboarding Manual](https://code.europa.eu/simpl/simpl-open/development/iaa/documentation/-/blob/main/versioned_docs/2.5.x/user-manual/ONBOARD.md) to complete this final step.*