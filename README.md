# Open WebUI on OpenShift

Open WebUI is an open-source, user-friendly web interface for LLMs. This deployment runs Open WebUI on OpenShift with OAuth integration and optional RHOAI (Red Hat OpenShift AI) backend support.

## Overview

This deployment includes:

- **Open WebUI** – Chat UI for LLMs (OpenAI-compatible API)
- **OAuth Proxy** – OpenShift authentication in front of the app
- **RHOAI Integration** – Use RHOAI InferenceServices as OpenAI-compatible endpoints

## Prerequisites

- OpenShift 4.19+ with admin access
- RHOAI with at least one InferenceService (optional; for RHOAI-backed chat)
- Helm 3 (for direct deployment)
- OpenShift GitOps/Argo CD (for GitOps deployment)

### Install OpenShift GitOps (optional)

If GitOps is not already installed:

```bash
./bootstrap.sh
```

If GitOps is already installed (e.g. from another repo), the script will detect it and skip installation.

## Deployment Options

### Option 1: GitOps deployment with Argo CD (recommended)

**Step 1: Point Argo CD at this repo**

Edit `gitops/openwebui.yaml` and set `spec.source.repoURL` to your fork or copy of this repo, then apply:

```bash
oc apply -f gitops/openwebui.yaml
```

**Step 2: Configure RHOAI endpoints (optional)**

After the app is running, configure Open WebUI to use your RHOAI InferenceService:

1. Get the route:
   ```bash
   oc get route openwebui -n demo-apps -o jsonpath='{.spec.host}'
   ```

2. Open the UI and complete the setup wizard.

3. In **Settings → Connections**, add an OpenAI-compatible connection:
   - **API URL**: `http://<model>-predictor.<namespace>.svc.cluster.local/v1`  
     Example: `http://granite-7b-predictor.demo-models.svc.cluster.local/v1`
   - **API Key**: leave empty for in-cluster access

**Step 3: Monitor**

```bash
oc get application openwebui -n openshift-gitops -w
oc get pods -n demo-apps
```

### Option 2: Direct Helm deployment

**Step 1: Deploy with Helm**

```bash
helm install openwebui ./helm \
  -f ./helm/values-rhoai.yaml \
  -n demo-apps --create-namespace
```

**Step 2: Access Open WebUI**

```bash
oc get route openwebui -n demo-apps -o jsonpath='{.spec.host}'
# Open https://<that-host> in a browser (OpenShift OAuth login)
```

## Configuration

### Logout (OpenShift OAuth)

To allow sign-out and re-login:

1. Set the OAuth proxy logout URL in your values (e.g. `helm/values.yaml` or Argo CD overrides):

   ```yaml
   oauthProxy:
     logoutUrl: "https://oauth-openshift.apps.<your-cluster-domain>/logout"
   ```

   To get the OAuth host:  
   `oc get route oauth-openshift -n openshift-authentication -o jsonpath='{.spec.host}'`

2. To log out, open:  
   `https://<your-openwebui-route>/oauth/sign_out`

### Storage

Default: 10Gi PVC for app data. Override in `helm/values-rhoai.yaml` (or your value file):

```yaml
storage:
  size: 20Gi
```

### RHOAI model namespace (cross-namespace RBAC)

If Open WebUI needs to discover InferenceServices in another namespace, set:

```yaml
rhoai:
  modelNamespace: "demo-models"
```

This creates a Role and RoleBinding in that namespace so the Open WebUI service account can list/get InferenceServices.

### Resources

Adjust in your values file:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
```

## Namespace

The chart deploys into the **demo-apps** namespace by default (same as the AnythingLLM pattern). You can override with the `namespace` value.

## Uninstall

**Helm:**

```bash
helm uninstall openwebui -n demo-apps
```

**GitOps:**

```bash
oc delete -f gitops/openwebui.yaml
```

**Optional – remove data:**

```bash
oc delete pvc openwebui-storage -n demo-apps
```

## References

- [Open WebUI](https://github.com/open-webui/open-webui)
- [Red Hat OpenShift AI (RHOAI)](https://access.redhat.com/products/red-hat-openshift-ai)
