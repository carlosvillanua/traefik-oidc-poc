# Traefik OIDC Workshop

This workshop demonstrates how to secure applications with OIDC authentication using Traefik Hub and session storage with Valkey.

## Prerequisites

- k3d installed
- kubectl installed
- helm installed

## Step 1: Create k3d Cluster

Create a local Kubernetes cluster with k3d:

```bash
k3d cluster create traefik-oidc \
  --port 80:80@loadbalancer \
  --port 443:443@loadbalancer \
  --port 8000:8000@loadbalancer \
  --k3s-arg "--disable=traefik@server:0"
```

## Step 2: Add Traefik Helm Repository

```bash
helm repo add --force-update traefik https://traefik.github.io/charts
```

## Step 3: Create Traefik Namespace and License

**Note:** Get your Traefik Hub token from the hub.traefik.io website

```bash
kubectl create namespace traefik

# Replace <TRAEFIK_HUB_TOKEN> with the token from hub.traefik.io
kubectl create secret generic traefik-hub-license --namespace traefik --from-literal=token=<TRAEFIK_HUB_TOKEN>
```

## Step 4: Deploy Valkey for Session Storage

Deploy Valkey (Redis-compatible) to store OIDC sessions:

```bash
kubectl apply -f valkey-deployment.yaml
```

## Step 5: Install Traefik Hub

Install Traefik Hub with API Management enabled:

```bash
helm upgrade --install --namespace traefik traefik traefik/traefik \
  --set hub.token=traefik-hub-license \
  --set hub.apimanagement.enabled=true \
  --set ingressRoute.dashboard.enabled=true \
  --set ingressRoute.dashboard.matchRule='Host(`dashboard.docker.localhost`)' \
  --set ingressRoute.dashboard.entryPoints={web} \
  --set image.registry=ghcr.io --set image.repository=traefik/traefik-hub --set image.tag=v3.18.0
```

## Step 6: Configure Traefik Plugins

Add the token transformer plugin:

**Note:** Get your GitHub token from your Traefik Solutions Architect

```bash
# Replace <GITHUB_TOKEN> with the token
helm upgrade traefik traefik/traefik -n traefik --wait \
  --reset-then-reuse-values \
  --set experimental.plugins.traefikforwardauthpost.moduleName=github.com/carlosvillanua/traefikforwardauthpost \
  --set experimental.plugins.traefikforwardauthpost.version=v1.0.0 \
  --set experimental.plugins.traefikforwardauthpost.settings.useUnsafe=true \
  --set 'additionalArguments[0]=--hub.pluginRegistry.sources.github.baseModuleName=github.com' \
  --set 'additionalArguments[1]=--hub.pluginRegistry.sources.github.github.token=<GITHUB_TOKEN>'
```

## Step 7: Create Application Namespace

```bash
kubectl create namespace apps
```

## Step 8: Deploy HTTPBin Application with OIDC

**Note:** Before deploying, edit `httpbin-oidc/1-middlewares.yaml` and configure your Azure AD credentials:
- Replace `<TENANT_ID>` with your Azure AD tenant ID
- Replace `<CLIENT_ID>` with your application client ID
- Replace `<CLIENT_SECRET>` with your application client secret

Deploy the HTTPBin application with OIDC middlewares and routes:

```bash
kubectl apply -f httpbin-oidc/
```

This deploys:
- **Middlewares** (1-middlewares.yaml):
  - `oidc` - OIDC authentication middleware with Valkey session storage
  - `token-transformer` - Transforms tokens in requests
  - `strip-prefix-secured` - Strips `/oidc` prefix from paths
- **HTTPBin Deployment & Service** (2-httpbin-deployment.yaml)
- **IngressRoutes** (2-route.yaml):
  - `/oidc` - Protected endpoint with OIDC authentication
  - `/callback` - OIDC callback handler
  - `/logout` - OIDC logout handler

## Step 9: Test the Application

Access the OIDC-protected endpoint:

```bash
# Open in browser
open http://localhost/oidc
```

You should be redirected to the Azure AD login page. After authentication, you'll be redirected back to HTTPBin.

## Monitoring Valkey Sessions

### View Current Sessions

Check what sessions are currently stored in Valkey:

```bash
# List all session keys
kubectl exec -n traefik $(kubectl get pod -n traefik -l app=valkey -o jsonpath='{.items[0].metadata.name}') -- redis-cli --user traefik --pass traefik --no-auth-warning KEYS "*"

# Count active sessions
kubectl exec -n traefik $(kubectl get pod -n traefik -l app=valkey -o jsonpath='{.items[0].metadata.name}') -- redis-cli --user traefik --pass traefik --no-auth-warning DBSIZE

# Get session with TTL
kubectl exec -n traefik $(kubectl get pod -n traefik -l app=valkey -o jsonpath='{.items[0].metadata.name}') -- sh -c 'redis-cli --user traefik --pass traefik --no-auth-warning KEYS "*" | while read key; do ttl=$(redis-cli --user traefik --pass traefik --no-auth-warning TTL "$key"); echo "Session: $key (TTL: ${ttl}s)"; done'
```

### Real-Time Session Monitoring

Monitor session operations in real-time (shows when sessions are created, accessed, or deleted):

```bash
# Simple real-time monitoring - shows all session operations
kubectl exec -n traefik $(kubectl get pod -n traefik -l app=valkey -o jsonpath='{.items[0].metadata.name}') -- redis-cli --user traefik --pass traefik MONITOR 2>/dev/null | grep --line-buffered -E '"(get|set|del)"' | grep -v "AUTH"
```

**Tip:** Keep this running in a separate terminal while you test authentication to see sessions being created and accessed in real-time!

### Watch Sessions (Auto-Refresh)

Continuously monitor active sessions with auto-refresh:

```bash
# Watch session count and details (refreshes every 2 seconds)
while true; do
  clear
  kubectl exec -n traefik $(kubectl get pod -n traefik -l app=valkey -o jsonpath="{.items[0].metadata.name}") -- sh -c "
    echo \"=== Active Sessions: \$(redis-cli --user traefik --pass traefik --no-auth-warning DBSIZE) ===\"
    echo
    redis-cli --user traefik --pass traefik --no-auth-warning KEYS \"*\" | while read key; do
      ttl=\$(redis-cli --user traefik --pass traefik --no-auth-warning TTL \"\$key\")
      echo \"Session: \$key (TTL: \${ttl}s)\"
    done
    echo
    echo \"=== Stats ===\"
    redis-cli --user traefik --pass traefik --no-auth-warning INFO stats | grep -E \"total_commands_processed|instantaneous_ops_per_sec\"
  "
  sleep 2
done
```

## Cleanup

To remove everything:

```bash
k3d cluster delete traefik-oidc
```
