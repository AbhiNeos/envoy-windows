# Apigee X + Envoy Adapter: Complete Step-by-Step Setup Guide (Windows + Docker)

This guide walks you through setting up Envoy proxy with the Apigee X Remote Service adapter on a Windows machine using Docker. Every step has been tested and verified to work.

---

## Architecture

```
Client Request --> Envoy Proxy (:8080) --> httpbin (target backend)
                       |
                       | (ext_authz gRPC)
                       v
              Apigee Adapter (:5000)
                       |
                       v
              Apigee X Management Plane
```

- Envoy intercepts all incoming API traffic
- The `ext_authz` filter calls the Apigee adapter for every request
- The adapter validates API keys, enforces quotas, and pushes analytics to Apigee X
- If authorized, the request is forwarded to the target service (httpbin)

---

## Prerequisites

| Requirement | How to install |
|-------------|----------------|
| Docker Desktop for Windows | https://www.docker.com/products/docker-desktop |
| Google Cloud SDK (gcloud) | https://cloud.google.com/sdk/docs/install |
| Apigee X organization | Must be provisioned in your GCP project |
| Apigee environment | At least one environment (e.g., `sandbox`) |

Ensure you are authenticated:
```powershell
gcloud auth login
gcloud config set project YOUR_GCP_PROJECT_ID
```

---

## Variables (replace with your values)

Throughout this guide, replace these placeholders:

| Variable | Description |
|----------|-------------|
| `YOUR_ORG` | Your Apigee X org name (usually GCP project ID) |
| `YOUR_ENV` | Your Apigee environment name |
| `YOUR_RUNTIME` | Your Apigee runtime hostname |

---

## Step 1: Create Project Directory Structure

Open PowerShell and run:

```powershell
mkdir C:\apigee-envoy-docker\config\policy-secret
mkdir C:\apigee-envoy-docker\config\analytics-secret
cd C:\apigee-envoy-docker
```

---

## Step 2: Create Envoy Configuration

Create file `config\envoy.yaml`:

```yaml
admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901

static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                access_log:
                  - name: envoy.access_loggers.stdout
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains:
                        - "*"
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            cluster: target_service
                http_filters:
                  - name: envoy.filters.http.ext_authz
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
                      transport_api_version: V3
                      grpc_service:
                        envoy_grpc:
                          cluster_name: apigee-remote-service-envoy
                        timeout: 1s
                      metadata_context_namespaces:
                        - envoy.filters.http.ext_authz
                      failure_mode_allow: false
                      with_request_body:
                        max_request_bytes: 4096
                        allow_partial_message: true
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: apigee-remote-service-envoy
      type: STRICT_DNS
      connect_timeout: 2s
      typed_extension_protocol_options:
        envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
          "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
          explicit_http_config:
            http2_protocol_options: {}
      load_assignment:
        cluster_name: apigee-remote-service-envoy
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: apigee-adapter
                      port_value: 5000

    - name: target_service
      type: STRICT_DNS
      connect_timeout: 2s
      load_assignment:
        cluster_name: target_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: httpbin
                      port_value: 80
```

---

## Step 3: Create Docker Compose File

Create file `docker-compose.yml`:

```yaml
services:
  envoy:
    image: envoyproxy/envoy:v1.28-latest
    container_name: envoy-proxy
    ports:
      - "8080:8080"
      - "9901:9901"
    volumes:
      - ./config/envoy.yaml:/etc/envoy/envoy.yaml:ro
    depends_on:
      - apigee-adapter
      - httpbin
    networks:
      - apigee-mesh
    restart: unless-stopped

  apigee-adapter:
    image: google/apigee-envoy-adapter:v2.1.1
    container_name: apigee-adapter
    ports:
      - "5000:5000"
      - "5001:5001"
    volumes:
      - ./config/config.yaml:/config/config.yaml:ro
      - ./config/policy-secret:/policy-secret:ro
      - ./config/analytics-secret:/analytics-secret:ro
    command: ["--config", "/config/config.yaml", "--policy-secret", "/policy-secret", "--analytics-secret", "/analytics-secret"]
    networks:
      - apigee-mesh
    restart: unless-stopped

  httpbin:
    image: kennethreitz/httpbin
    container_name: httpbin-target
    ports:
      - "8000:80"
    networks:
      - apigee-mesh
    restart: unless-stopped

networks:
  apigee-mesh:
    driver: bridge
```

---

## Step 4: Create Analytics Service Account

The adapter needs a GCP service account with `roles/apigee.analyticsAgent` to push analytics.

```powershell
# Create service account
gcloud iam service-accounts create apigee-envoy-analytics --project=YOUR_ORG --display-name="Apigee Envoy Adapter Analytics"

# Grant Analytics Agent role
gcloud projects add-iam-policy-binding YOUR_ORG --member="serviceAccount:apigee-envoy-analytics@YOUR_ORG.iam.gserviceaccount.com" --role="roles/apigee.analyticsAgent"

# Download the key
gcloud iam service-accounts keys create config\analytics-sa.json --iam-account=apigee-envoy-analytics@YOUR_ORG.iam.gserviceaccount.com --project=YOUR_ORG
```

---

## Step 5: Provision the Apigee Remote Service

> **IMPORTANT:** The Windows `.exe` version of the CLI has a known bug that causes:
> `fs.WalkDir: copyToTempDir: open proxies\remote-service-gcp: file does not exist`
>
> **Workaround:** Download the Linux binary and run it inside a Docker container.

### 5a. Download the Linux CLI

```powershell
mkdir cli
Invoke-WebRequest -Uri "https://github.com/apigee/apigee-remote-service-cli/releases/download/v2.1.5/apigee-remote-service-cli_2.1.5_linux_64-bit.tar.gz" -OutFile cli\cli.tar.gz -UseBasicParsing
tar -xzf cli\cli.tar.gz -C cli
Remove-Item cli\cli.tar.gz
```

### 5b. Run provisioning via Docker

```powershell
$TOKEN = gcloud auth print-access-token

docker run --rm -v "${PWD}\cli:/cli" -v "${PWD}\config:/output" alpine:latest sh -c "chmod +x /cli/apigee-remote-service-cli && /cli/apigee-remote-service-cli provision --organization 'YOUR_ORG' --environment 'YOUR_ENV' --runtime 'https://YOUR_RUNTIME' --analytics-sa /output/analytics-sa.json --token '$TOKEN' > /output/provision-output.yaml"
```

This creates `config\provision-output.yaml` containing a Kubernetes manifest with base64-encoded secrets.

---

## Step 6: Extract Secrets from Provisioning Output

The provisioning output is a Kubernetes manifest. You need to extract the actual config and secrets from it.

### 6a. Create the flat config.yaml

Create file `config\config.yaml` (use values from the `data.config.yaml` section in the provisioning output):

```yaml
tenant:
  remote_service_api: https://YOUR_RUNTIME/remote-service
  org_name: YOUR_ORG
  env_name: YOUR_ENV
analytics:
  collection_interval: 10s
auth:
  jwt_provider_key: https://YOUR_RUNTIME/remote-token/token
  append_metadata_headers: true
```

### 6b. Decode the policy secrets

Open the provisioning output and find the `Secret` named `YOUR_ORG-YOUR_ENV-policy-secret`. Decode each base64 field:

Create file `decode-secrets.ps1`:

```powershell
# Replace these base64 values with the ones from YOUR provisioning output
# Found in: metadata.name: YOUR_ORG-YOUR_ENV-policy-secret

$crtBase64 = "PASTE_remote-service.crt_BASE64_HERE"
$keyBase64 = "PASTE_remote-service.key_BASE64_HERE"
$propsBase64 = "PASTE_remote-service.properties_BASE64_HERE"

# Decode and write policy secrets
$bytes = [System.Convert]::FromBase64String($crtBase64)
[System.IO.File]::WriteAllBytes("$PWD\config\policy-secret\remote-service.crt", $bytes)
Write-Host "  remote-service.crt: $($bytes.Length) bytes"

$bytes = [System.Convert]::FromBase64String($keyBase64)
[System.IO.File]::WriteAllBytes("$PWD\config\policy-secret\remote-service.key", $bytes)
Write-Host "  remote-service.key: $($bytes.Length) bytes"

$bytes = [System.Convert]::FromBase64String($propsBase64)
[System.IO.File]::WriteAllBytes("$PWD\config\policy-secret\remote-service.properties", $bytes)
Write-Host "  remote-service.properties: $($bytes.Length) bytes"

# Copy analytics SA to analytics-secret directory
Copy-Item ".\config\analytics-sa.json" ".\config\analytics-secret\client_secret.json" -Force
Write-Host "  client_secret.json: copied"

Write-Host "`nDone! Secrets decoded."
```

Run it:
```powershell
powershell -ExecutionPolicy Bypass -File decode-secrets.ps1
```

---

## Step 7: Start All Services

```powershell
docker compose up -d
```

Verify all 3 containers are running:
```powershell
docker compose ps
```

Expected output:
```
NAME             IMAGE                                STATUS          PORTS
apigee-adapter   google/apigee-envoy-adapter:v2.1.1   Up              0.0.0.0:5000-5001->5000-5001/tcp
envoy-proxy      envoyproxy/envoy:v1.28-latest        Up              0.0.0.0:8080->8080/tcp, 0.0.0.0:9901->9901/tcp
httpbin-target   kennethreitz/httpbin                  Up              0.0.0.0:8000->80/tcp
```

Check adapter logs for successful startup:
```powershell
docker compose logs apigee-adapter --tail 10
```

You should see:
```
INFO  product/manager.go:174  started product manager
INFO  quota/manager.go:117    started quota manager with 10 workers
INFO  analytics/manager.go:196 started analytics manager
INFO  app/main.go:171         listening: :5000
INFO  app/main.go:204         listening: :5001
```

---

## Step 8: Create API Product, Developer, and App (Get API Key)

The adapter needs an API Product with the correct target binding and an App to generate the API key.

### 8a. Create API Product

```powershell
$TOKEN = gcloud auth print-access-token
$HEADERS = @{ "Authorization" = "Bearer $TOKEN"; "Content-Type" = "application/json" }

# IMPORTANT: apiSource and apigee-remote-service-targets must match the Host header
# that the adapter sees. For local Docker, this is "localhost:8080".
$BODY = @'
{
  "name": "envoy-httpbin-product",
  "displayName": "Envoy httpbin Product",
  "description": "API Product for httpbin via Envoy adapter",
  "environments": ["YOUR_ENV"],
  "approvalType": "auto",
  "operationGroup": {
    "operationConfigs": [
      {
        "apiSource": "localhost:8080",
        "operations": [
          {
            "resource": "/**",
            "methods": ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS", "HEAD"]
          }
        ]
      }
    ],
    "operationConfigType": "remoteservice"
  },
  "attributes": [
    {"name": "access", "value": "public"},
    {"name": "apigee-remote-service-targets", "value": "localhost:8080"}
  ]
}
'@

Invoke-WebRequest -Uri "https://apigee.googleapis.com/v1/organizations/YOUR_ORG/apiproducts" -Method Post -Headers $HEADERS -Body $BODY -UseBasicParsing
```

> **Key points:**
> - Do NOT include the `"access"` field at the root level (it's an Apigee Edge-only field and causes 400 errors on Apigee X)
> - `operationConfigType` must be `"remoteservice"` for the Envoy adapter
> - `apiSource` and `apigee-remote-service-targets` must match the `:authority` (Host) header that Envoy sees - for local Docker this is `localhost:8080`
> - `resource: "/**"` allows all paths; only one operation per config is allowed

### 8b. Create Developer

```powershell
$BODY = @'
{
  "email": "envoy-test@example.com",
  "firstName": "Envoy",
  "lastName": "Test",
  "userName": "envoy-test"
}
'@

Invoke-WebRequest -Uri "https://apigee.googleapis.com/v1/organizations/YOUR_ORG/developers" -Method Post -Headers $HEADERS -Body $BODY -UseBasicParsing
```

### 8c. Create Developer App (generates the API key)

```powershell
$BODY = @'
{
  "name": "envoy-httpbin-app",
  "apiProducts": ["envoy-httpbin-product"]
}
'@

Invoke-WebRequest -Uri "https://apigee.googleapis.com/v1/organizations/YOUR_ORG/developers/envoy-test@example.com/apps" -Method Post -Headers $HEADERS -Body $BODY -UseBasicParsing
```

### 8d. Retrieve the API Key

```powershell
$APP = Invoke-RestMethod -Uri "https://apigee.googleapis.com/v1/organizations/YOUR_ORG/developers/envoy-test@example.com/apps/envoy-httpbin-app" -Method Get -Headers $HEADERS
$API_KEY = $APP.credentials[0].consumerKey
Write-Host "Your API Key: $API_KEY"
```

---

## Step 9: Restart the Adapter (Force Product Sync)

The adapter syncs products every ~2 minutes. To force immediate sync, restart it:

```powershell
docker compose restart apigee-adapter
```

Wait 10-15 seconds for it to start and fetch products.

---

## Step 10: Test

### Without API key (should return 403):
```powershell
curl http://localhost:8080/headers
```

### With API key (should return 200):
```powershell
curl http://localhost:8080/headers -H "x-api-key: YOUR_API_KEY"
```

### Expected successful response:
```json
{
  "headers": {
    "Accept": "*/*",
    "Host": "localhost:8080",
    "User-Agent": "curl/8.19.0",
    "X-Api-Key": "YOUR_API_KEY",
    "X-Apigee-Accesstoken": "",
    "X-Apigee-Api": "localhost:8080",
    "X-Apigee-Apiproducts": "envoy-httpbin-product",
    "X-Apigee-Application": "envoy-httpbin-app",
    "X-Apigee-Authorized": "true",
    "X-Apigee-Clientid": "YOUR_API_KEY",
    "X-Apigee-Developeremail": "envoy-test@example.com",
    "X-Apigee-Environment": "sandbox",
    "X-Apigee-Organization": "neosalpha-apigee",
    "X-Apigee-Scope": ""
  }
}
```

---

## Final Directory Structure

```
apigee-envoy-docker/
├── docker-compose.yml
├── decode-secrets.ps1
├── config/
│   ├── envoy.yaml                          # Envoy proxy config
│   ├── config.yaml                         # Adapter config (flat YAML)
│   ├── analytics-sa.json                   # GCP service account key
│   ├── provision-output.yaml               # Raw provisioning output (for reference)
│   ├── policy-secret/
│   │   ├── remote-service.crt              # JWKS public key (decoded from base64)
│   │   ├── remote-service.key              # RSA private key (decoded from base64)
│   │   └── remote-service.properties       # Key ID properties (decoded from base64)
│   └── analytics-secret/
│       └── client_secret.json              # Copy of analytics-sa.json
└── cli/
    └── apigee-remote-service-cli           # Linux binary (for provisioning via Docker)
```

---

## Common Issues and Fixes

| Problem | Cause | Fix |
|---------|-------|-----|
| `fs.WalkDir: copyToTempDir: open proxies\remote-service-gcp: file does not exist` | Windows CLI path separator bug | Use Linux binary via Docker (Step 5) |
| Adapter crash: `open /policy-secret/remote-service.key: no such file or directory` | Secrets not decoded from provisioning output | Run `decode-secrets.ps1` (Step 6) |
| Adapter crash: `unknown flag: --address` | Wrong command flags | Use `--config`, `--policy-secret`, `--analytics-secret` only |
| 403 with valid API key: `no apis: localhost:8080` | Product target doesn't match Host header | Set `apigee-remote-service-targets` to `localhost:8080` |
| 403 with valid API key: `no path: /headers` | Product has no path rules | Use `operationGroup` with `resource: "/**"` and `operationConfigType: "remoteservice"` |
| `Invalid JSON: Unknown name "access"` when creating product | `access` is an Edge-only field | Remove `"access"` from root level (keep it only in `attributes`) |
| `Operations must contain exactly one entity` | Multiple operations in one config | Use only one operation with `resource: "/**"` to cover all paths |
| `docker-compose.yml: attribute version is obsolete` | Old compose format | Remove `version: "3.8"` from docker-compose.yml |
| Analytics warning: `No analytics service account` | Missing `--analytics-sa` during provision | Re-provision with `--analytics-sa` flag pointing to SA JSON |

---

## Useful Commands

```powershell
# View all logs
docker compose logs -f

# View adapter logs only
docker compose logs -f apigee-adapter

# Restart all services
docker compose restart

# Stop everything
docker compose down

# Full rebuild (after config changes)
docker compose down
docker compose up -d

# Check Envoy admin dashboard
# Open in browser: http://localhost:9901

# Direct test to httpbin (bypass Envoy)
curl http://localhost:8000/headers
```

---

## Ports Reference

| Service | Port | Protocol | Description |
|---------|------|----------|-------------|
| Envoy Proxy | 8080 | HTTP | API gateway (client-facing) |
| Envoy Admin | 9901 | HTTP | Envoy admin dashboard |
| Apigee Adapter | 5000 | gRPC | ext_authz service |
| Apigee Adapter | 5001 | gRPC | Access log service |
| httpbin | 8000 | HTTP | Target backend (for testing) |
