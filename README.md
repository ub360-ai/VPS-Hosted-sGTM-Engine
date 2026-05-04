# Meta Conversions API (CAPI) Infrastructure via Private sGTM Cluster

I have developed this repository to provide a robust, self-hosted tracking infrastructure that mitigates data loss caused by Intelligent Tracking Prevention (ITP), ad-blockers, and general browser-side limitations. By deploying a dual-node Server-Side Google Tag Manager (sGTM) cluster on a private VPS, I have established a first-party data pipeline that ensures high-fidelity attribution and 100% event deduplication.

## Infrastructure Overview

This architecture is built on the following stack:
- **Server Environment**: Linux-based VPS
- **Deployment Orchestration**: Coolify (Docker-based)
- **Containerization**: Official GTM Server Image (gcr.io)
- **Data Transport**: GA4 Protocol (acting as the transport layer)
- **End-Point**: Meta Events API (v20.0+)

## Technical Architecture and Data Flow

I implemented a dual-node strategy to isolate the production environment from the debugging environment:

1. **Tagging Cluster**: Handles high-volume, live production traffic. It processes incoming HTTP requests from the client and executes outgoing API calls to Meta.
2. **Preview Server**: Dedicated to container debugging. This separation ensures that the overhead of the GTM Debugger does not impact production latency.

### The Request Lifecycle
The data flow I designed follows this sequence:
- **Client-Side Trigger**: A lead event is pushed to the DataLayer on the user's browser.
- **Payload Transport**: The Google Tag (GA4) intercepts the DataLayer object, attaches a unique `event_id`, and routes the payload to my private VPS endpoint (`sgtm.yourdomain.com`).
- **Server-Side Processing**: The sGTM GA4 Client parses the incoming request, extracts first-party cookies (`_fbp`, `_fbc`), and prepares the server-side event.
- **API Execution**: The Facebook CAPI tag executes a POST request to the Meta Graph API with hashed user parameters and the deduplication key.

## Deployment Scenarios

I have optimized this infrastructure for two primary deployment paths: Coolify and Standard VPS (Raw Docker + Nginx).

### Scenario A: Coolify Deployment (Recommended)
In the Coolify ecosystem, I leverage the internal Traefik proxy. This eliminates the need for a manual Nginx configuration, preventing proxy-chaining conflicts and SSL management overhead.

#### Coolify Docker Compose
I utilize the following configuration within the Coolify service manager:

```yaml
version: '3.8'
services:
  tagging:
    image: 'gcr.io/cloud-tagging-10302018/gtm-cloud-image:stable'
    container_name: gtm-cluster
    restart: always
    environment:
      CONTAINER_CONFIG: '${CONTAINER_CONFIG}'
      PREVIEW_SERVER_URL: '${PREVIEW_SERVER_URL}'
    expose:
      - '8080'
  preview:
    image: 'gcr.io/cloud-tagging-10302018/gtm-cloud-image:stable'
    container_name: gtm-preview
    restart: always
    environment:
      CONTAINER_CONFIG: '${CONTAINER_CONFIG}'
      RUN_AS_PREVIEW_SERVER: 'true'
    expose:
      - '8080'
```

### Scenario B: Standard VPS Deployment (Nginx Proxy)
For deployments on a raw VPS without an orchestration layer like Coolify, I implement a manual Nginx reverse proxy to handle SSL termination and load distribution.

#### Standard Docker Compose
```yaml
version: '3.8'
services:
  preview-server:
    image: gcr.io/cloud-tagging-10302018/gtm-cloud-image:stable
    container_name: preview-server
    environment:
      CONTAINER_CONFIG: "YOUR_CONTAINER_CONFIG_STRING"
      RUN_AS_PREVIEW_SERVER: "true"
    ports:
      - "8081:8080"

  server-side-tagging-cluster:
    image: gcr.io/cloud-tagging-10302018/gtm-cloud-image:stable
    container_name: server-side-tagging-cluster
    environment:
      CONTAINER_CONFIG: "YOUR_CONTAINER_CONFIG_STRING"
      PREVIEW_SERVER_URL: "https://sgtm.yourdomain.com/"
    ports:
      - "8082:8080"
    depends_on:
      - preview-server
```

#### Nginx Configuration (Proxy Layer)
I use the following Nginx block to secure the infrastructure via HTTPS and route traffic to the appropriate Docker containers:

```nginx
upstream gtm_servers {
    server 127.0.0.1:8082; # Tagging Cluster
    server 127.0.0.1:8081; # Preview Server
}

server {
    listen 80;
    server_name sgtm.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name sgtm.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/sgtm.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sgtm.yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://gtm_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Implementation Process

### 1. Infrastructure Layer Deployment
I utilized Coolify to manage the Docker lifecycle. The deployment requires two distinct service configurations:
- **Tagging Service**: Configured to run on port 8080, receiving traffic from the primary tracking subdomain.
- **Preview Service**: Configured with `RUN_AS_PREVIEW_SERVER: true` to enable the debugging interface.

### 2. Client-Side Data Schema
I implemented a custom DataLayer schema to capture granular user data at the point of conversion. 

```javascript
window.dataLayer.push({
    'event': 'generate_lead',
    'event_id': unique_id,
    'user_data': {
        'email': 'user@example.com',
        'phone': '+1234567890',
        'address': {
            'first_name': 'John',
            'city': 'City',
            'state': 'State'
        }
    }
});
```

### 3. Server-Side Logic and Parameter Mapping
I resolved parameter mapping issues by explicitly defining the data schema in the sGTM container:
- **PII Parameters**: Mapped from the `user_data` nested object.
- **Connection Parameters**: Captured from `ip_override` and `user_agent` event data.
- **Attribution Parameters**: Extracted from `_fbc` and `_fbp` cookies.

## Final Validation
I verified the infrastructure using the following technical checkpoints:
1. **HTTP 200 OK**: Verified the `/healthz` endpoint on the tagging cluster.
2. **Event Match Quality (EMQ)**: Achieved a high-tier score by providing 8+ user signals.
3. **Traceability**: Confirmed that the `fbclid` correctly resolves to the `fbc` parameter in the outgoing server request.
