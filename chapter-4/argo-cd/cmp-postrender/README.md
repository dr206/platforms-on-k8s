# ArgoCD CMP Sidecar with Helm Post-Render Script

This guide adds a Config Management Plugin (CMP) sidecar to an existing ArgoCD installation so that Helm charts can be passed through a post-render shell script before ArgoCD applies them.

<!-- TOC -->
* [ArgoCD CMP Sidecar with Helm Post-Render Script](#argocd-cmp-sidecar-with-helm-post-render-script)
  * [Prerequisites](#prerequisites)
  * [Step 1: Check Your ArgoCD Version](#step-1-check-your-argocd-version)
  * [Step 2: Create the Plugin ConfigMap](#step-2-create-the-plugin-configmap)
  * [Step 3: Create the Post-Render Script ConfigMap](#step-3-create-the-post-render-script-configmap)
  * [Step 4: Patch argocd-repo-server](#step-4-patch-argocd-repo-server)
  * [Step 5: Verify the Sidecar is Running](#step-5-verify-the-sidecar-is-running)
  * [Step 6: Verify the Plugin is Being Used](#step-6-verify-the-plugin-is-being-used)
  * [Troubleshooting](#troubleshooting)
<!-- TOC -->

## Prerequisites

- ArgoCD already installed via `kubectl apply`
- `kubectl` configured against your cluster
- Your post-render script already exists on disk (referred to as `post-render.sh` below)

---

## Step 1: Check Your ArgoCD Version

The init container that copies `argocd-cmp-server` must match your running ArgoCD version.

```bash
ARGOCD_PATCH_VERSION=$(
  kubectl get deployment argocd-repo-server -n argocd \
    -o jsonpath='{.spec.template.spec.containers[0].image}' \
    | cut -d':' -f2
)
```

Note the tag (e.g. `v2.10.0`). You will use it in Step 3.

---

## Step 2: Create the Plugin ConfigMap

This defines the CMP plugin — what to run when ArgoCD generates manifests for an app using this plugin.

```bash
kubectl apply -n argocd -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: cmp-helm-postrender
  namespace: argocd
data:
  plugin.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: helm-postrender
    spec:
      discover:
        find:
          glob: "**/Chart.yaml"
      init:
        command: [sh, -c]
        args:
          - helm dependency update .
      generate:
        command: [sh, -c]
        args:
          - |
            helm template "$ARGOCD_APP_NAME" . \
              --namespace "$ARGOCD_APP_NAMESPACE" \
              --include-crds \
              -f values.yaml \
              | /usr/local/bin/post-render.sh
EOF
```

---

## Step 3: Create the Post-Render Script ConfigMap

Load your existing `post-render.sh` into a ConfigMap so it can be mounted into the sidecar.

```bash
kubectl create configmap cmp-post-render-script \
  --from-file=post-render.sh=../../../chapter-2/rewrite-bitnami-images.sh \
  -n argocd
```

Verify it was created:

```bash
kubectl get configmap cmp-post-render-script -n argocd
```

---

## Step 4: Patch argocd-repo-server

Add the init container and CMP sidecar to the existing `argocd-repo-server` Deployment.

Replace `v2.10.0` with the version from Step 1.

```bash
kubectl patch deployment argocd-repo-server -n argocd --type strategic --patch "
spec:
  template:
    spec:
      volumes:
        - name: var-files
          emptyDir: {}
        - name: plugins
          emptyDir: {}
        - name: cmp-tmp
          emptyDir: {}
        - name: cmp-plugin-config
          configMap:
            name: cmp-helm-postrender
        - name: cmp-post-render-script
          configMap:
            name: cmp-post-render-script
            defaultMode: 0755
      initContainers:
        - name: copy-cmp-server
          image: quay.io/argoproj/argocd:${ARGOCD_PATCH_VERSION}
          command: [sh, -c]
          args:
            - cp /usr/local/bin/argocd-cmp-server /var/run/argocd/
          volumeMounts:
            - name: var-files
              mountPath: /var/run/argocd
      containers:
        - name: cmp-helm-postrender
          image: alpine/helm:3.14.0
          command: [/var/run/argocd/argocd-cmp-server]
          securityContext:
            runAsNonRoot: true
            runAsUser: 999
          volumeMounts:
            - name: var-files
              mountPath: /var/run/argocd
            - name: plugins
              mountPath: /home/argocd/cmp-server/plugins
            - name: cmp-tmp
              mountPath: /tmp
            - name: cmp-plugin-config
              mountPath: /home/argocd/cmp-server/config/plugin.yaml
              subPath: plugin.yaml
            - name: cmp-post-render-script
              mountPath: /usr/local/bin/post-render.sh
              subPath: post-render.sh
"
```

```shell
 kubectl patch deployment argocd-repo-server -n argocd --type json --patch '
  [
    {
      "op": "remove",
      "path": "/spec/template/spec/initContainers/1"
    }
  ]
'
```

This fixes an issue where the patch command adds a duplicate init container on subsequent runs. The `copy-cmp-server` init container should only be added once.

The index used in the `remove` operation may need to be adjusted if you have other init containers defined:

```shell
kubectl get deployment argocd-repo-server -n argocd \
    -o jsonpath='{range .spec.template.spec.initContainers[*]}{.name}{"\n"}{end}'
```

---

## Step 5: Verify the Sidecar is Running

Wait for the rollout to complete:

```bash
kubectl rollout status deployment/argocd-repo-server -n argocd
```

Confirm the sidecar container is present in the pod:

```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-repo-server \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'
```

You should see `cmp-helm-postrender` listed alongside `argocd-repo-server`.

Check the sidecar logs to confirm it started cleanly:

```bash
kubectl logs -n argocd deployment/argocd-repo-server -c cmp-helm-postrender
```

---

## Step 6: Verify the Plugin is Being Used

Inspect the application and confirm it is using the plugin:

```bash
argocd app get staging-environment
```

Preview the manifests the plugin generates (runs helm + post-render without applying):

```bash
argocd app manifests staging-environment
```

Watch the sidecar logs in real time during a sync to see your post-render script execute:

```bash
kubectl logs -n argocd deployment/argocd-repo-server -c cmp-helm-postrender -f
```

Trigger a manual sync if needed:

```bash
argocd app sync conference-staging
```

---

## Troubleshooting

**Sidecar not starting**
Check init container logs to confirm `argocd-cmp-server` was copied successfully:
```bash
kubectl logs -n argocd deployment/argocd-repo-server -c copy-cmp-server
```

**Plugin not detected**
Confirm the plugin name in the Application matches `metadata.name` in `plugin.yaml` exactly:
```bash
kubectl get configmap cmp-helm-postrender -n argocd -o jsonpath='{.data.plugin\.yaml}'
```

**Post-render script not executable**
Confirm the ConfigMap was created with the correct `defaultMode`:
```bash
kubectl get deployment argocd-repo-server -n argocd \
  -o jsonpath='{.spec.template.spec.volumes[?(@.name=="cmp-post-render-script")]}'
```

**Script output corrupting manifests**
Ensure your `post-render.sh` only writes valid YAML to stdout. Use stderr for any logging:
```bash
echo "debug message" >&2
```
