# Open WebUI on OpenShift (simple admin guide)

This repository is a **small, ready-to-apply set of OpenShift instructions** (YAML files) that run **[Open WebUI](https://openwebui.com/)**—a web interface your team can use to chat with AI models.

## What you are actually deploying?

Think of Open WebUI as a **browser window** that sits in front of your AI services. It is **not** the brain of the AI itself. Your real models and inference still live on the OpenShift AI (or other) endpoints you already operate. Open WebUI helps people pick a model, upload documents for search (RAG), and keep a familiar chat history—with a persistent disk so settings survive restarts.

## What gets created on the cluster?

| Piece | Plain-English purpose |
| --- | --- |
| **Deployment** | Runs the official `ghcr.io/open-webui/open-webui:main` container as a normal user (UID **1001**) so it fits typical **restricted-v2** security profiles. |
| **Service** | Internal network path from the cluster to the app (port **8080**). |
| **Route** | A public **https://…apps.&lt;your-cluster&gt;…** address with **edge** TLS so browsers get a padlock without extra certificate work inside the pod. |
| **PVC (10 Gi)** | Disk space for the app’s database and files so **nothing important disappears** when the pod restarts. |
| **Secret (template)** | Placeholder storage for **model server URLs** and **API keys** (including support for **several** endpoints at once). |
| **ConsoleLink (optional)** | A **shortcut inside the OpenShift web console** on the **`open-webui` project overview** that opens the app in a new tab (see below). |

## Before you start (one-time)

1. **Create or choose a project (namespace)** named `open-webui`, or let the GitHub Action create it for you the first time it runs.
2. **Edit the Secret values** in `openshift/secrets.yaml` *before* you rely on them, or update them later in the OpenShift Console (see below). Never share real keys in email or chat.

### Multiple model backends (OpenShift AI 3.3 and friends)

Open WebUI reads **`OPENAI_API_BASE_URLS`** as a **semicolon-separated** list of OpenAI-compatible base URLs (for example, several OpenShift AI inference endpoints). Pair them with **`OPENAI_API_KEYS`** in the **same order**, also separated by semicolons. Example shape:

- `https://first-server.../v1;https://second-server.../v1`
- `key-for-first;key-for-second`

### Document search (RAG) and web search

This deployment turns on **`ENABLE_RAG_WEB_SEARCH=true`** and sets **`RAG_EMBEDDING_ENGINE=openai`** so embeddings can use your configured OpenAI-compatible stack. If you instead run **Ollama** for embeddings, switch the embedding engine to **`ollama`** in `openshift/deployment.yaml` and set **`OLLAMA_BASE_URL`** in the Secret (see the commented example in `openshift/secrets.yaml`).

## Install from your workstation (no GitHub required)

You need the `oc` command and permission to create Deployments, Services, Routes, PVCs, and Secrets in your project.

```bash
oc new-project open-webui   # skip if the project already exists
oc project open-webui
# Apply project resources; skip ConsoleLink here unless you use a cluster-admin account (see “Console shortcut” below).
for f in openshift/*.yaml; do
  grep -q 'kind: ConsoleLink' "$f" && continue
  oc apply -f "$f"
done
oc rollout status deployment/open-webui --timeout=10m
```

Open the **Route** URL under **Networking → Routes** in the OpenShift Console (copy **Location**), or use the **console shortcut** once it is configured.

## Console shortcut

This is a **link inside the OpenShift web console** (not a second URL for the app itself). OpenShift can show **“Open Web UI”** on the **namespace / project overview** when you select the **`open-webui`** project. That comes from **`openshift/consolelink.yaml`** (`ConsoleLink`).

1. **Who can install it?** Creating a `ConsoleLink` is usually a **cluster-wide** permission. If `oc apply -f openshift/consolelink.yaml` returns **Forbidden**, ask a **cluster administrator** to apply that file (or grant you the right to create `consolelinks.console.openshift.io`).
2. **Set the real URL.** The template uses a placeholder host. After the Route exists, point the shortcut at it—either edit **`spec.href`** in the YAML and re-apply, or run (from a machine with `oc`):

   ```bash
   ROUTE_HOST=$(oc get route open-webui -n open-webui -o jsonpath='{.spec.host}')
   oc patch consolelink open-webui-launch --type=merge -p "{\"spec\":{\"href\":\"https://${ROUTE_HOST}\"}}"
   ```

3. **Where it appears.** In the web console, open the **Administrator** perspective, choose project **`open-webui`**, and open **Home → Projects → open-webui** (or the project **Overview**). You should see **Open Web UI** linking to your Route.

If the link never appears, confirm the namespace is named exactly **`open-webui`** (the ConsoleLink matches the standard label **`kubernetes.io/metadata.name=open-webui`**, which OpenShift 4.12+ sets automatically).

## How to update the model URL (OpenShift Console)

You do **not** need to edit code—only the configuration OpenShift passes into the container.

1. Log in to the **OpenShift Web Console**.
2. Switch to the **`open-webui`** project (namespace) using the project dropdown at the top.
3. Open **Workloads → Secrets** and select **`open-webui-secrets`**.
4. Choose **Actions → Edit Secret** (or **Add key/value** depending on your console version).
5. Update **`OPENAI_API_BASE_URLS`** (semicolons between multiple URLs) and matching **`OPENAI_API_KEYS`**.
6. Open **Workloads → Deployments → open-webui** and click **Restart rollout** (or scale down to 0 and back up) so running pods reload environment variables from the Secret.

> **Note:** Open WebUI can store some settings in its database. If something does not change after a restart, check the in-app admin settings or temporarily consult Open WebUI’s documentation on persistent configuration.

## GitHub Actions automation (optional)

The workflow **`.github/workflows/deploy.yml`** installs `oc`, logs in, runs **`oc apply -f openshift/`**, and waits for the rollout. Configure these **repository secrets** in GitHub (**Settings → Secrets and variables → Actions**):

| Secret | What to put there |
| --- | --- |
| `OPENSHIFT_SERVER` | API URL, for example `https://api.my-cluster.domain:6443` |
| `OPENSHIFT_TOKEN` | A service account or user token with rights to apply these objects in `open-webui` |

Pushes to **`main`** and manual **“Run workflow”** both trigger a deploy.

## Troubleshooting (common OpenShift messages)

### “Forbidden” or “cannot create resource … User cannot …”

Your user or service account **lacks RBAC permissions** in the namespace (or cluster). Ask your cluster administrator to grant **edit** or a tailored role that allows **Deployments, Services, Routes, PVCs, and Secrets** in `open-webui`.

**Only for `ConsoleLink`:** If everything else applied but **`consolelink.yaml`** failed, that is normal for a **namespace-only** role. Either skip the console shortcut or have a **cluster admin** apply **`openshift/consolelink.yaml`** and patch **`spec.href`** as described in [Console shortcut](#console-shortcut).

### “unable to validate against any security context constraint”

The pod’s **securityContext** does not match any **SCC** you are allowed to use. This repo targets **non-root UID 1001** and **restricted-v2**-friendly settings. If it still fails, the administrator may need to allow the service account (often `default` in the namespace) to use **restricted-v2**, or adjust SCC bindings—share the exact error text with them.

### PVC stays **Pending**

Usually **storage class** or **quota** issues. In **Storage → PersistentVolumeClaims**, read the **Events** tab. If your cluster requires a specific `storageClassName`, edit `openshift/pvc.yaml`, uncomment and set **`storageClassName`**, then re-apply.

### Route shows the page but login or models fail

- Re-check **`OPENAI_API_BASE_URLS`** and **`OPENAI_API_KEYS`** (order and semicolons matter).
- From a debug pod or **Networking** tools, confirm the app can **reach** your model endpoints (DNS, TLS, and corporate proxies all matter).
- Look at **Workloads → Pods → open-webui → Logs** for connection or authentication errors.

### Image pull errors (`ImagePullBackOff`)

The cluster must be allowed to pull from **`ghcr.io`**. Your platform team may need to permit egress or configure a **pull secret** for GitHub Container Registry.

### Pod keeps restarting with “permission denied” on `/root` or cache paths

The manifests set **`runAsUser: 1001`** and point **`HOME`** at **`/app/backend/data`** so caches land on the persistent disk. If errors persist, the container image your cluster pulled may expect a different user; ask your platform team to confirm **SCC** settings, or build an image with **`UID=1001`** using the upstream Dockerfile’s build arguments.

## Files in this repo

- `openshift/deployment.yaml` — pod definition, security context, RAG-related environment variables, volume mount.
- `openshift/service.yaml` — internal Service on port 8080.
- `openshift/route.yaml` — HTTPS Route with edge termination.
- `openshift/pvc.yaml` — 10 Gi claim for `/app/backend/data`.
- `openshift/secrets.yaml` — template Secret for URLs and keys.
- `openshift/consolelink.yaml` — optional `ConsoleLink` shortcut in the web console (cluster scope; update `spec.href` after the Route exists).
- `.github/workflows/deploy.yml` — optional CI deploy using `oc apply` (namespace manifests first; ConsoleLink step is best-effort).

## License and upstream

Open WebUI is an upstream open-source project; see their site and license for product details. This repository only packages opinionated OpenShift manifests for easier operations.
