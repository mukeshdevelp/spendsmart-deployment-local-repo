# SpendSmart Login Troubleshooting — Keycloak and CSP

## Overview

This document records how the local SpendSmart login issue on Minikube was diagnosed and fixed.

**Local environment**
- Helm chart repository: `~/clickhouse/helm-charts`
- Helm chart: `charts/spendsmart`
- Values file: `client-a-file.yml`
- Kubernetes namespace: `spendsmart`
- Frontend URL: `http://localhost:8083`
- Keycloak URL: `http://localhost:8081`
- Backend URL: `http://localhost:8000`

> These `localhost` URLs are for local development. They must not be used as the public URLs for an EKS deployment.

## 1. Login issue: Keycloak iframe blocked by CSP

### Browser error

```text
Framing 'http://localhost:8081/' violates the following Content Security Policy directive:
"frame-src 'self' https:".
```

### Root cause

The frontend was returning two Content Security Policy (CSP) policies:

1. The Nginx response header already allowed Keycloak:
   `frame-src 'self' http://localhost:8081 https:;`
2. The frontend HTML's CSP meta tag still blocked the local Keycloak URL:
   `frame-src 'self' https:;`

Both policies are enforced by the browser. Updating the Nginx header alone did not remove the restriction in the HTML meta tag.

### How we verified it

Check the Nginx response header:

```bash
curl -sSI http://localhost:8083/ | grep -i content-security-policy
```

Check the CSP meta tag served in the HTML:

```bash
curl -s http://localhost:8083/ | grep -in "Content-Security-Policy\|frame-src"
```

Inspect the HTML inside the running frontend pod:

```bash
kubectl exec -n spendsmart deployment/spendsmart-frontend -- \
  sh -c "grep -n 'Content-Security-Policy' /usr/share/nginx/html/index.html"
```

The HTML contained a policy similar to:

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' http: https: ws: wss:; font-src 'self' data: https:; frame-src 'self' https:;">
```

The Helm chart's Nginx template was at:

```text
charts/spendsmart/templates/frontend/nginx-configmap.yaml
```

It already generated CSP headers from the values file using:

```text
.Values.frontend.nginx.csp.connectSrc
.Values.frontend.nginx.csp.frameSrc
```

The rendered Nginx header was correct; the remaining restriction came from the HTML meta tag.

### Temporary diagnostic fix used

The running pod was temporarily modified with `sed`:

```bash
kubectl exec -n spendsmart deployment/spendsmart-frontend -- sh -c \
"sed -i \"s/frame-src 'self' https:/frame-src 'self' https: http:\/\/localhost:8081/\" /usr/share/nginx/html/index.html"
```

Verify the change:

```bash
kubectl exec -n spendsmart deployment/spendsmart-frontend -- \
  sh -c "grep -n 'Content-Security-Policy' /usr/share/nginx/html/index.html"
```

After this change, the Keycloak framing error was resolved.

**Important:** This was a temporary diagnostic fix only. Editing a file inside a running pod is not durable; a recreated pod or a new deployment can restore the original file from the container image.

### Permanent fix recommended

Update the CSP meta tag in the frontend source/build so that `frame-src` permits the Keycloak origin used by the environment. For local development, that origin is:

```text
http://localhost:8081
```

Build and push a new frontend image with a new, explicit tag, update the frontend image tag in `client-a-file.yml`, then deploy the updated chart:

```bash
cd ~/clickhouse/helm-charts

helm template spendsmart ./charts/spendsmart \
  -n spendsmart \
  -f client-a-file.yml >/tmp/spendsmart-rendered.yaml

helm upgrade --install spendsmart ./charts/spendsmart \
  -n spendsmart \
  -f client-a-file.yml
```

Run the `helm upgrade` only after `helm template` succeeds. Earlier, an invalid edit in `nginx-configmap.yaml` caused a YAML parse error, so validate first.

## 2. Backend API calls blocked by CSP

After the Keycloak framing issue was fixed, the browser showed a second CSP error:

```text
Connecting to 'http://localhost:8000/api/v1/accounts/...' violates the following Content Security Policy directive:
"connect-src 'self' https://keycloak.client-a.example https://cloud-sentry.client-a.example wss: localhost:8083 localhost:8081".
```

A similar error appeared for:

```text
http://localhost:8000/api/v1/report/summary
```

### Root cause

The frontend was calling the backend at `http://localhost:8000`, but the Nginx CSP `connect-src` list did not allow that origin.

### Values change for local development

In `client-a-file.yml`, find `frontend.nginx.csp.connectSrc` and add the backend origin while preserving the other existing entries:

```yaml
frontend:
  nginx:
    csp:
      connectSrc:
        - "'self'"
        - "https://keycloak.client-a.example"
        - "https://cloud-sentry.client-a.example"
        - "http://localhost:8000"
        - "http://localhost:8081"
        - "http://localhost:8083"
        - "wss:"
```

Use the actual structure and indentation already present in the values file; do not replace unrelated frontend settings.

Validate and deploy:

```bash
helm template spendsmart ./charts/spendsmart \
  -n spendsmart \
  -f client-a-file.yml >/tmp/spendsmart-rendered.yaml

helm upgrade --install spendsmart ./charts/spendsmart \
  -n spendsmart \
  -f client-a-file.yml
```

Verify the active Nginx header:

```bash
curl -sSI http://localhost:8083/ | grep -i content-security-policy
```

The `connect-src` directive should include `http://localhost:8000`.

### Verify backend port-forwarding

In a separate terminal, keep this command running:

```bash
kubectl port-forward -n spendsmart svc/spendsmart-backend 8000:80
```

Then test from another terminal:

```bash
curl -i http://localhost:8000/api/v1/accounts/
```

An HTTP error such as `401` or `404` can still mean the request reached the backend; check the route and authentication requirements. The goal of this test is to distinguish application responses from browser CSP blocking.

## 3. Check the frontend and backend

Check pod status:

```bash
kubectl get pods -n spendsmart
```

Check frontend logs:

```bash
kubectl logs -n spendsmart deployment/spendsmart-frontend --tail=100
```

Check backend logs:

```bash
kubectl logs -n spendsmart deployment/spendsmart-backend --tail=100
```

Check the active Nginx ConfigMap:

```bash
kubectl get configmap spendsmart-frontend-nginx -n spendsmart -o yaml
```

Port-forward the frontend if needed:

```bash
kubectl port-forward -n spendsmart svc/spendsmart-frontend 8083:80
```

After changes, hard-refresh the browser with **Ctrl+Shift+R**.

## 4. EKS deployment considerations

Do not carry local `localhost` URLs into an EKS environment. In a browser, `localhost` refers to the end user's own machine, not the EKS cluster.

For EKS:

1. Use the real HTTPS frontend, Keycloak, and backend API URLs.
2. Add those actual origins to the appropriate CSP directives (`frame-src` for framing and `connect-src` for API/WebSocket requests).
3. Configure the Keycloak client's valid redirect URIs and web origins to match the deployed frontend URL.
4. Make sure the CSP meta tag in the built frontend image and the Nginx CSP response header are compatible.
5. Prefer a versioned frontend image tag instead of relying on `latest`, so rollbacks and deployments are predictable.
6. Validate the chart before applying it:

   ```bash
   helm template spendsmart ./charts/spendsmart \
     -n spendsmart \
     -f client-a-file.yml >/tmp/spendsmart-rendered.yaml
   ```

## Summary

- **Keycloak login error:** the frontend HTML meta CSP blocked `http://localhost:8081`, even though the Nginx header allowed it.
- **Temporary fix:** update the meta tag in the running pod with `sed`; this does not persist across pod recreation.
- **Permanent fix:** update the frontend source/build, publish a new image, and deploy it.
- **Backend API CSP error:** add the local backend origin `http://localhost:8000` to `connect-src` for local development.
- **For EKS:** use real HTTPS service URLs, not localhost.
