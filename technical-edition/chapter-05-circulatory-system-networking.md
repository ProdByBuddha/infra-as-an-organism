# Chapter 5: The Circulatory System (Networking)

**Learning Objectives**:
- Understand modern networking patterns for distributed systems
- Implement intelligent request routing with Traefik
- Deploy service discovery and health checking with Consul
- Build event-driven architectures with NATS
- Master load balancing strategies for high availability
- Configure TLS termination and automatic certificate management
- Design API gateway patterns for microservices
- Implement zero-downtime deployments
- Troubleshoot network-related issues systematically

---

## Introduction: Your Infrastructure's Blood Vessels

Just as the human circulatory system delivers blood, oxygen, and nutrients to every cell, your infrastructure's networking layer delivers requests, data, and messages to every service.

Without effective networking:
- Services can't find each other
- Traffic overloads some servers while others sit idle
- Users hit slow or dead endpoints
- Deployments cause downtime
- Debugging becomes impossible

**With modern networking patterns, you get**:
- Automatic service discovery (services find each other dynamically)
- Intelligent load balancing (traffic distributed optimally)
- Zero-downtime deployments (update without interruption)
- Secure communication (TLS everywhere)
- Observable traffic patterns (see every request)

This chapter builds the circulatory system that keeps your infrastructure alive and efficient.

---

## The Evolution of Infrastructure Networking

### Phase 1: Static Configuration (2000s)

**Pattern**: Hardcoded IPs and ports everywhere

```nginx
# nginx.conf
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}
```

**Problems**:
- Add a server? Edit configs and reload
- Server IP changes? Update everywhere
- Server dies? Manual removal from config
- No health checking
- Configuration drift across environments

### Phase 2: Configuration Management (2010s)

**Pattern**: Ansible/Chef/Puppet generate configs

```yaml
# Ansible template
upstream backend {
{% for host in backend_servers %}
    server {{ host.ip }}:{{ host.port }};
{% endfor %}
}
```

**Better, but**:
- Still requires deployment to update
- Manual service registration
- Health checks basic
- Scaling is slow

### Phase 3: Dynamic Service Discovery (2015+)

**Pattern**: Services register themselves, discovery is automatic

```go
// Service registers itself on startup
consul.Register("api", "192.168.1.10:8080", []string{"production", "v2"})

// Other services discover it dynamically
endpoints := consul.Discover("api", "production")
```

**Benefits**:
- Automatic registration/deregistration
- Real-time health checking
- Instant scaling
- Zero configuration drift

### Phase 4: Service Mesh (2018+)

**Pattern**: Infrastructure handles all networking concerns

```yaml
# Services just talk to localhost
# Mesh handles:
# - Discovery
# - Load balancing
# - Retries
# - Circuit breaking
# - TLS
# - Observability
```

**Ultimate goal**: Services focus on business logic, infrastructure handles networking.

---

## Core Networking Patterns

### Pattern 1: Reverse Proxy / API Gateway

**Purpose**: Single entry point that routes requests to backend services

```
Internet
    ↓
┌───────────────────────┐
│   Reverse Proxy       │
│   (Traefik)           │
└───────┬───────────────┘
        │
        ├──→ Service A (api.example.com/users)
        ├──→ Service B (api.example.com/orders)
        └──→ Service C (api.example.com/products)
```

**Features**:
- TLS termination (HTTPS → HTTP internally)
- Load balancing
- Request routing based on path/host/headers
- Rate limiting
- Authentication
- Compression

### Pattern 2: Service Discovery

**Purpose**: Services find each other without hardcoded addresses

```
┌──────────────────────────────────────────┐
│         Service Registry (Consul)        │
│  api:        [10.0.1.5, 10.0.1.6]       │
│  database:   [10.0.2.10]                 │
│  cache:      [10.0.3.15, 10.0.3.16]     │
└──────────────────────────────────────────┘
           ↑                    ↓
      Register              Discover
           │                    │
    ┌──────┴────────┐    ┌─────┴──────┐
    │   Service A   │    │  Service B │
    │ (registers)   │    │ (queries)  │
    └───────────────┘    └────────────┘
```

**Benefits**:
- No hardcoded IPs
- Automatic scaling
- Health-based routing
- Multi-datacenter support

### Pattern 3: Load Balancing

**Purpose**: Distribute traffic across multiple instances

**Strategies**:

1. **Round Robin**: Simple rotation
```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (repeat)
```

2. **Least Connections**: Send to server with fewest active connections
```
Server A: 10 connections
Server B: 5 connections  ← Send here
Server C: 8 connections
```

3. **Weighted**: Based on server capacity
```
Server A (8 cores):  40% of traffic
Server B (4 cores):  20% of traffic
Server C (12 cores): 40% of traffic
```

4. **IP Hash**: Same client always goes to same server
```
hash(client_ip) % num_servers = server_index
Client 192.168.1.10 → Always Server B
```

### Pattern 4: Circuit Breaking

**Purpose**: Prevent cascading failures

```
Normal operation:
Client → Service A → Service B (working) → Success

Service B starts failing:
Client → Service A → Service B (failing) → Error
Client → Service A → Service B (failing) → Error
Client → Service A → Service B (failing) → Error
(Service A wastes resources on doomed requests)

Circuit breaker activates:
Client → Service A → CIRCUIT OPEN → Fail fast
(Service A returns error immediately, preserves resources)

After timeout:
Circuit breaker tests if Service B recovered
If yes: Resume normal operation
If no: Keep circuit open
```

### Pattern 5: Event-Driven Communication

**Purpose**: Async communication between services

```
Traditional (synchronous):
API → Order Service → Wait → Payment Service → Wait → Inventory

Event-driven (asynchronous):
API → Order Service → Publish "OrderCreated" event → Return immediately

Meanwhile:
  Payment Service → Subscribe "OrderCreated" → Process payment
  Inventory Service → Subscribe "OrderCreated" → Reserve items
  Email Service → Subscribe "OrderCreated" → Send confirmation
```

**Benefits**:
- Faster response times
- Services don't block each other
- Easy to add new consumers
- Natural retries and resilience

---

## Traefik: Modern Reverse Proxy and Load Balancer

Traefik is a cloud-native edge router that automatically discovers services and configures routing.

### Why Traefik?

**Traditional (Nginx)**:
```nginx
# Manual configuration required
upstream api {
    server api1:8080;
    server api2:8080;
}

server {
    location /api {
        proxy_pass http://api;
    }
}
```
*Every change requires config edit + reload*

**Traefik**:
```yaml
# docker-compose.yml
services:
  api:
    labels:
      - "traefik.http.routers.api.rule=PathPrefix(`/api`)"
```
*Traefik discovers automatically, no reload needed*

### Architecture

```
┌─────────────────────────────────────────────────┐
│                    Internet                      │
└────────────────────┬────────────────────────────┘
                     │
                     ↓ :80, :443
        ┌────────────────────────┐
        │       Traefik          │
        │   ┌────────────────┐   │
        │   │ EntryPoints    │   │ Define ports
        │   └────────┬───────┘   │
        │            ↓            │
        │   ┌────────────────┐   │
        │   │   Routers      │   │ Match requests
        │   └────────┬───────┘   │
        │            ↓            │
        │   ┌────────────────┐   │
        │   │  Middlewares   │   │ Transform requests
        │   └────────┬───────┘   │
        │            ↓            │
        │   ┌────────────────┐   │
        │   │   Services     │   │ Backend servers
        │   └────────┬───────┘   │
        └────────────┼───────────┘
                     │
        ┌────────────┼────────────┐
        │            ↓             │
        │   ┌─────────────┐       │
        │   │   Service A │       │
        │   └─────────────┘       │
        │   ┌─────────────┐       │
        │   │   Service B │       │
        │   └─────────────┘       │
        └──────────────────────────┘
```

### Complete Traefik Configuration

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: unless-stopped
    command:
      # API and dashboard
      - "--api.dashboard=true"
      - "--api.insecure=false"

      # Providers
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.file.directory=/etc/traefik/dynamic"

      # Entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"

      # Certificate resolver (Let's Encrypt)
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"

      # Logging
      - "--accesslog=true"
      - "--log.level=INFO"

      # Metrics
      - "--metrics.prometheus=true"
      - "--metrics.prometheus.entrypoint=metrics"
      - "--entrypoints.metrics.address=:8082"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"  # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/dynamic:/etc/traefik/dynamic:ro
      - ./letsencrypt:/letsencrypt
    labels:
      # Dashboard router
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.example.com`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dashboard.middlewares=auth"

      # Basic auth middleware
      - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$..."
    networks:
      - web

  # Example backend service
  api:
    image: myapp:latest
    container_name: api
    restart: unless-stopped
    labels:
      - "traefik.enable=true"

      # HTTP router (redirect to HTTPS)
      - "traefik.http.routers.api-http.rule=Host(`api.example.com`)"
      - "traefik.http.routers.api-http.entrypoints=web"
      - "traefik.http.routers.api-http.middlewares=redirect-to-https"

      # HTTPS router
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=letsencrypt"
      - "traefik.http.routers.api.middlewares=rate-limit,compress"

      # Service configuration
      - "traefik.http.services.api.loadbalancer.server.port=8080"
      - "traefik.http.services.api.loadbalancer.healthcheck.path=/health"
      - "traefik.http.services.api.loadbalancer.healthcheck.interval=10s"

      # Middlewares
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      - "traefik.http.middlewares.rate-limit.ratelimit.average=100"
      - "traefik.http.middlewares.rate-limit.ratelimit.burst=50"
      - "traefik.http.middlewares.compress.compress=true"
    networks:
      - web

networks:
  web:
    external: true
```

### Advanced Routing Rules

**Host-based routing**:
```yaml
# Route by domain
- "traefik.http.routers.api.rule=Host(`api.example.com`)"
- "traefik.http.routers.web.rule=Host(`www.example.com`)"
```

**Path-based routing**:
```yaml
# Route by path
- "traefik.http.routers.users.rule=PathPrefix(`/api/users`)"
- "traefik.http.routers.orders.rule=PathPrefix(`/api/orders`)"
```

**Header-based routing**:
```yaml
# Route by header (e.g., API version)
- "traefik.http.routers.api-v1.rule=Headers(`X-API-Version`, `v1`)"
- "traefik.http.routers.api-v2.rule=Headers(`X-API-Version`, `v2`)"
```

**Combined rules**:
```yaml
# Multiple conditions
- "traefik.http.routers.api.rule=Host(`api.example.com`) && PathPrefix(`/v2`) && Method(`GET`, `POST`)"
```

**Priority-based routing**:
```yaml
# Higher priority wins when rules overlap
- "traefik.http.routers.specific.rule=Host(`api.example.com`) && Path(`/special`)"
- "traefik.http.routers.specific.priority=100"
- "traefik.http.routers.general.rule=Host(`api.example.com`)"
- "traefik.http.routers.general.priority=1"
```

### Middleware Examples

**Rate Limiting**:
```yaml
labels:
  - "traefik.http.middlewares.api-ratelimit.ratelimit.average=100"
  - "traefik.http.middlewares.api-ratelimit.ratelimit.burst=200"
  - "traefik.http.middlewares.api-ratelimit.ratelimit.period=1s"
```

**IP Whitelisting**:
```yaml
labels:
  - "traefik.http.middlewares.ip-whitelist.ipwhitelist.sourcerange=192.168.1.0/24,10.0.0.0/8"
```

**Headers Modification**:
```yaml
labels:
  # Add headers
  - "traefik.http.middlewares.secure-headers.headers.customrequestheaders.X-Forwarded-Proto=https"
  - "traefik.http.middlewares.secure-headers.headers.sslredirect=true"

  # Security headers
  - "traefik.http.middlewares.secure-headers.headers.stsSeconds=31536000"
  - "traefik.http.middlewares.secure-headers.headers.contentTypeNosniff=true"
  - "traefik.http.middlewares.secure-headers.headers.browserXssFilter=true"
```

**Circuit Breaker**:
```yaml
labels:
  - "traefik.http.middlewares.circuit-breaker.circuitbreaker.expression=NetworkErrorRatio() > 0.3 || ResponseCodeRatio(500, 600, 0, 600) > 0.3"
```

**Retry**:
```yaml
labels:
  - "traefik.http.middlewares.retry.retry.attempts=3"
  - "traefik.http.middlewares.retry.retry.initialinterval=100ms"
```

**Authentication**:
```yaml
labels:
  # Basic auth
  - "traefik.http.middlewares.auth.basicauth.users=user:$$apr1$$..."

  # Forward auth (external service)
  - "traefik.http.middlewares.oauth.forwardauth.address=http://auth-service/verify"
  - "traefik.http.middlewares.oauth.forwardauth.authResponseHeaders=X-Auth-User"
```

### Load Balancing Strategies

**Round Robin** (default):
```yaml
labels:
  - "traefik.http.services.api.loadbalancer.server.port=8080"
```

**Weighted Round Robin**:
```yaml
# docker-compose.yml with scaling
services:
  api-v1:
    deploy:
      replicas: 3
    labels:
      - "traefik.http.services.api.loadbalancer.server.port=8080"
      - "traefik.http.services.api.loadbalancer.server.weight=1"

  api-v2:
    deploy:
      replicas: 1
    labels:
      - "traefik.http.services.api.loadbalancer.server.port=8080"
      - "traefik.http.services.api.loadbalancer.server.weight=3"
```

**Sticky Sessions**:
```yaml
labels:
  - "traefik.http.services.api.loadbalancer.sticky.cookie=true"
  - "traefik.http.services.api.loadbalancer.sticky.cookie.name=server_id"
  - "traefik.http.services.api.loadbalancer.sticky.cookie.secure=true"
  - "traefik.http.services.api.loadbalancer.sticky.cookie.httponly=true"
```

### Health Checks

```yaml
labels:
  - "traefik.http.services.api.loadbalancer.healthcheck.path=/health"
  - "traefik.http.services.api.loadbalancer.healthcheck.interval=10s"
  - "traefik.http.services.api.loadbalancer.healthcheck.timeout=3s"
  - "traefik.http.services.api.loadbalancer.healthcheck.scheme=http"
  - "traefik.http.services.api.loadbalancer.healthcheck.headers.Host=api.example.com"
```

---

## Consul: Service Discovery and Configuration

Consul provides service discovery, health checking, and distributed configuration.

### Architecture

```
┌─────────────────────────────────────────────────┐
│              Consul Cluster                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │ Server 1│──│ Server 2│──│ Server 3│         │
│  │ (Leader)│  │         │  │         │         │
│  └────┬────┘  └────┬────┘  └────┬────┘         │
└───────┼───────────┼────────────┼───────────────┘
        │           │            │
        │    Gossip Protocol     │
        │           │            │
   ┌────┴───┐  ┌───┴────┐  ┌───┴────┐
   │Agent A │  │Agent B │  │Agent C │
   │Node 1  │  │Node 2  │  │Node 3  │
   └───┬────┘  └───┬────┘  └───┬────┘
       │           │            │
   ┌───▼───┐   ┌──▼────┐   ┌──▼────┐
   │Service│   │Service│   │Service│
   │  API  │   │  API  │   │  DB   │
   └───────┘   └───────┘   └───────┘
```

### Complete Consul Setup

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  consul-server:
    image: consul:1.16
    container_name: consul-server
    restart: unless-stopped
    command: >
      agent
      -server
      -bootstrap-expect=1
      -ui
      -client=0.0.0.0
      -bind=0.0.0.0
      -data-dir=/consul/data
      -config-dir=/consul/config
    ports:
      - "8500:8500"  # HTTP API
      - "8600:8600"  # DNS
      - "8600:8600/udp"
    volumes:
      - consul_data:/consul/data
      - ./consul/config:/consul/config:ro
    networks:
      - consul
    environment:
      - CONSUL_BIND_INTERFACE=eth0

  # Consul agent on each node
  consul-agent:
    image: consul:1.16
    restart: unless-stopped
    command: >
      agent
      -retry-join=consul-server
      -client=0.0.0.0
      -bind=0.0.0.0
      -data-dir=/consul/data
    depends_on:
      - consul-server
    networks:
      - consul
    volumes:
      - ./consul/config:/consul/config:ro

  # Example service with Consul registration
  api:
    image: myapp:latest
    restart: unless-stopped
    depends_on:
      - consul-agent
    networks:
      - consul
    environment:
      - CONSUL_HTTP_ADDR=consul-agent:8500
      - SERVICE_NAME=api
      - SERVICE_PORT=8080
    volumes:
      - ./register-service.sh:/register-service.sh
    entrypoint: /register-service.sh

volumes:
  consul_data:

networks:
  consul:
    driver: bridge
```

### Service Registration

**Manual registration** (HTTP API):
```bash
curl -X PUT http://consul:8500/v1/agent/service/register \
  -d '{
    "ID": "api-1",
    "Name": "api",
    "Tags": ["production", "v2"],
    "Address": "10.0.1.5",
    "Port": 8080,
    "Check": {
      "HTTP": "http://10.0.1.5:8080/health",
      "Interval": "10s",
      "Timeout": "3s"
    }
  }'
```

**Service registration script**:
```bash
#!/bin/bash
# register-service.sh

# Get container IP
CONTAINER_IP=$(hostname -i)

# Register with Consul
cat > /tmp/service.json <<EOF
{
  "ID": "${SERVICE_NAME}-$(hostname)",
  "Name": "${SERVICE_NAME}",
  "Address": "${CONTAINER_IP}",
  "Port": ${SERVICE_PORT},
  "Tags": ["production"],
  "Check": {
    "HTTP": "http://${CONTAINER_IP}:${SERVICE_PORT}/health",
    "Interval": "10s",
    "Timeout": "3s",
    "DeregisterCriticalServiceAfter": "30s"
  }
}
EOF

curl -X PUT http://${CONSUL_HTTP_ADDR}/v1/agent/service/register \
  -d @/tmp/service.json

# Deregister on shutdown
trap "curl -X PUT http://${CONSUL_HTTP_ADDR}/v1/agent/service/deregister/${SERVICE_NAME}-$(hostname)" EXIT

# Start application
exec /app/start.sh
```

**Application-level registration** (Go example):
```go
package main

import (
    "fmt"
    consul "github.com/hashicorp/consul/api"
)

func registerService() error {
    config := consul.DefaultConfig()
    client, err := consul.NewClient(config)
    if err != nil {
        return err
    }

    registration := &consul.AgentServiceRegistration{
        ID:      "api-1",
        Name:    "api",
        Port:    8080,
        Address: "10.0.1.5",
        Tags:    []string{"production", "v2"},
        Check: &consul.AgentServiceCheck{
            HTTP:                           "http://10.0.1.5:8080/health",
            Interval:                       "10s",
            Timeout:                        "3s",
            DeregisterCriticalServiceAfter: "30s",
        },
    }

    return client.Agent().ServiceRegister(registration)
}

func main() {
    if err := registerService(); err != nil {
        panic(err)
    }

    // Deregister on shutdown
    defer func() {
        config := consul.DefaultConfig()
        client, _ := consul.NewClient(config)
        client.Agent().ServiceDeregister("api-1")
    }()

    // Start application...
}
```

### Service Discovery

**DNS-based discovery**:
```bash
# Query service via DNS
dig @consul-server -p 8600 api.service.consul SRV

# Result:
# api.service.consul. 0 IN SRV 1 1 8080 0a000105.addr.consul.
# (IP: 10.0.1.5, Port: 8080)
```

**HTTP API discovery**:
```bash
# Get all healthy instances of a service
curl http://consul-server:8500/v1/health/service/api?passing

# Response:
[
  {
    "Node": "node1",
    "Service": {
      "ID": "api-1",
      "Service": "api",
      "Address": "10.0.1.5",
      "Port": 8080,
      "Tags": ["production", "v2"]
    },
    "Checks": [
      {
        "Status": "passing",
        "Output": "HTTP GET http://10.0.1.5:8080/health: 200 OK"
      }
    ]
  }
]
```

**Application-level discovery** (Go example):
```go
func discoverService(serviceName string) ([]string, error) {
    config := consul.DefaultConfig()
    client, err := consul.NewClient(config)
    if err != nil {
        return nil, err
    }

    // Query healthy instances
    services, _, err := client.Health().Service(serviceName, "", true, nil)
    if err != nil {
        return nil, err
    }

    // Extract addresses
    var endpoints []string
    for _, service := range services {
        addr := fmt.Sprintf("%s:%d",
            service.Service.Address,
            service.Service.Port)
        endpoints = append(endpoints, addr)
    }

    return endpoints, nil
}

// Usage
endpoints, _ := discoverService("api")
// endpoints: ["10.0.1.5:8080", "10.0.1.6:8080"]
```

### Health Checks

**HTTP health check**:
```json
{
  "Check": {
    "HTTP": "http://10.0.1.5:8080/health",
    "Interval": "10s",
    "Timeout": "3s"
  }
}
```

**TCP health check**:
```json
{
  "Check": {
    "TCP": "10.0.1.5:5432",
    "Interval": "10s",
    "Timeout": "3s"
  }
}
```

**Script health check**:
```json
{
  "Check": {
    "Args": ["/usr/local/bin/check_service.sh"],
    "Interval": "30s",
    "Timeout": "10s"
  }
}
```

**TTL health check** (application reports health):
```json
{
  "Check": {
    "TTL": "30s",
    "DeregisterCriticalServiceAfter": "90s"
  }
}
```

Application updates TTL:
```bash
# Report healthy
curl -X PUT http://consul:8500/v1/agent/check/pass/service:api-1

# Report unhealthy
curl -X PUT http://consul:8500/v1/agent/check/fail/service:api-1
```

### Key-Value Store

Consul's KV store provides distributed configuration.

**Store configuration**:
```bash
# Store individual values
curl -X PUT http://consul:8500/v1/kv/config/database/host -d 'postgres.example.com'
curl -X PUT http://consul:8500/v1/kv/config/database/port -d '5432'

# Store JSON
curl -X PUT http://consul:8500/v1/kv/config/app -d '{
  "debug": false,
  "max_connections": 100,
  "timeout": 30
}'
```

**Retrieve configuration**:
```bash
# Get single value
curl http://consul:8500/v1/kv/config/database/host?raw

# Get with metadata
curl http://consul:8500/v1/kv/config/database/host

# Get all under prefix
curl http://consul:8500/v1/kv/config/database?recurse
```

**Watch for changes**:
```bash
# Block until value changes
curl "http://consul:8500/v1/kv/config/app?wait=5m&index=123"
```

**Application integration** (Go example):
```go
func loadConfig() (*AppConfig, error) {
    config := consul.DefaultConfig()
    client, _ := consul.NewClient(config)
    kv := client.KV()

    // Get config
    pair, _, err := kv.Get("config/app", nil)
    if err != nil {
        return nil, err
    }

    var appConfig AppConfig
    json.Unmarshal(pair.Value, &appConfig)

    // Watch for changes
    go watchConfig(kv)

    return &appConfig, nil
}

func watchConfig(kv *consul.KV) {
    var waitIndex uint64

    for {
        queryOpts := &consul.QueryOptions{
            WaitIndex: waitIndex,
            WaitTime:  5 * time.Minute,
        }

        pair, meta, err := kv.Get("config/app", queryOpts)
        if err != nil {
            log.Printf("Error watching config: %v", err)
            continue
        }

        if pair != nil && meta.LastIndex > waitIndex {
            log.Println("Config changed, reloading...")
            // Reload application config
            waitIndex = meta.LastIndex
        }
    }
}
```

---

## NATS: Event-Driven Messaging

NATS is a lightweight, high-performance messaging system for event-driven architectures.

### Architecture

```
┌──────────────────────────────────────────────┐
│              NATS Server                     │
│                                              │
│  ┌────────────┐  ┌────────────┐            │
│  │  Subject   │  │  Subject   │            │
│  │  orders.*  │  │  users.*   │            │
│  └─────┬──────┘  └─────┬──────┘            │
└────────┼─────────────────┼──────────────────┘
         │                 │
    Pub/Sub           Pub/Sub
         │                 │
    ┌────┴─────┐      ┌────┴─────┐
    │ Service  │      │ Service  │
    │   API    │      │   Auth   │
    └──────────┘      └──────────┘
```

### Messaging Patterns

**1. Publish-Subscribe** (fan-out):
```
Publisher → NATS → Subscriber 1
                 → Subscriber 2
                 → Subscriber 3
```
All subscribers receive the message.

**2. Queue Groups** (load balancing):
```
Publisher → NATS → Queue "workers"
                 → Worker 1 (gets message)
                 → Worker 2 (idle)
                 → Worker 3 (idle)
```
Only one subscriber in the queue receives the message.

**3. Request-Reply** (RPC):
```
Requester → NATS → Responder
          ← NATS ← Response
```
Synchronous request-response pattern.

### NATS Setup

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  nats:
    image: nats:2.10
    container_name: nats
    restart: unless-stopped
    command:
      - "-js"  # Enable JetStream
      - "-m 8222"  # Monitoring port
      - "--cluster_name=mycluster"
    ports:
      - "4222:4222"  # Client connections
      - "8222:8222"  # HTTP monitoring
      - "6222:6222"  # Cluster routing
    networks:
      - nats

networks:
  nats:
    driver: bridge
```

### Publishing Messages

**Go example**:
```go
package main

import (
    "github.com/nats-io/nats.go"
)

func publishEvent() error {
    // Connect to NATS
    nc, err := nats.Connect("nats://nats:4222")
    if err != nil {
        return err
    }
    defer nc.Close()

    // Publish simple message
    nc.Publish("orders.created", []byte("order-12345"))

    // Publish JSON
    type OrderCreated struct {
        OrderID    string  `json:"order_id"`
        UserID     string  `json:"user_id"`
        Total      float64 `json:"total"`
        Items      int     `json:"items"`
    }

    order := OrderCreated{
        OrderID: "order-12345",
        UserID:  "user-789",
        Total:   99.99,
        Items:   3,
    }

    data, _ := json.Marshal(order)
    nc.Publish("orders.created", data)

    return nil
}
```

### Subscribing to Messages

**Simple subscription**:
```go
func subscribe() error {
    nc, _ := nats.Connect("nats://nats:4222")
    defer nc.Close()

    // Subscribe to subject
    sub, err := nc.Subscribe("orders.created", func(msg *nats.Msg) {
        log.Printf("Received: %s", string(msg.Data))

        var order OrderCreated
        json.Unmarshal(msg.Data, &order)

        // Process order...
    })
    if err != nil {
        return err
    }

    // Keep subscription alive
    select {}
}
```

**Queue subscription** (load balanced):
```go
func queueSubscribe() error {
    nc, _ := nats.Connect("nats://nats:4222")
    defer nc.Close()

    // Multiple instances share work
    nc.QueueSubscribe("orders.created", "order-processors", func(msg *nats.Msg) {
        log.Printf("Worker processing: %s", string(msg.Data))
        // Process order...
    })

    select {}
}
```

**Wildcard subscriptions**:
```go
// Subscribe to all order events
nc.Subscribe("orders.*", handler)  // orders.created, orders.updated, orders.deleted

// Subscribe to all events
nc.Subscribe(">", handler)  // Matches everything
```

### Request-Reply Pattern

**Server** (responder):
```go
func startResponder() error {
    nc, _ := nats.Connect("nats://nats:4222")
    defer nc.Close()

    // Handle requests
    nc.Subscribe("get.user", func(msg *nats.Msg) {
        userID := string(msg.Data)

        // Fetch user from database
        user := fetchUser(userID)
        response, _ := json.Marshal(user)

        // Send response
        msg.Respond(response)
    })

    select {}
}
```

**Client** (requester):
```go
func requestUser(userID string) (*User, error) {
    nc, _ := nats.Connect("nats://nats:4222")
    defer nc.Close()

    // Send request and wait for response
    msg, err := nc.Request("get.user", []byte(userID), 2*time.Second)
    if err != nil {
        return nil, err
    }

    var user User
    json.Unmarshal(msg.Data, &user)

    return &user, nil
}
```

### JetStream (Persistent Messaging)

JetStream adds persistence, replay, and exactly-once delivery.

**Create stream**:
```go
func createStream() error {
    nc, _ := nats.Connect("nats://nats:4222")
    defer nc.Close()

    js, _ := nc.JetStream()

    // Create stream
    _, err := js.AddStream(&nats.StreamConfig{
        Name:     "ORDERS",
        Subjects: []string{"orders.*"},
        Storage:  nats.FileStorage,
        Retention: nats.LimitsPolicy,
        MaxAge:   24 * time.Hour,  // Keep for 24 hours
    })

    return err
}
```

**Publish to stream**:
```go
func publishToStream() error {
    nc, _ := nats.Connect("nats://nats:4222")
    defer nc.Close()

    js, _ := nc.JetStream()

    // Publish with acknowledgment
    ack, err := js.Publish("orders.created", []byte("order-123"))
    if err != nil {
        return err
    }

    log.Printf("Published to stream: %s, sequence: %d", ack.Stream, ack.Sequence)
    return nil
}
```

**Consume from stream**:
```go
func consumeFromStream() error {
    nc, _ := nats.Connect("nats://nats:4222")
    defer nc.Close()

    js, _ := nc.JetStream()

    // Create durable consumer
    _, err := js.Subscribe("orders.*", func(msg *nats.Msg) {
        log.Printf("Received: %s", string(msg.Data))

        // Process message...

        // Acknowledge
        msg.Ack()
    }, nats.Durable("order-processor"))

    select {}
}
```

---

## Integration: Traefik + Consul + NATS

Combining all three creates a powerful, dynamic infrastructure.

### Architecture

```
Internet
    ↓
┌───────────────────────────────────────────────┐
│  Traefik (Edge Router)                        │
│  - TLS termination                            │
│  - Load balancing                             │
│  - Request routing                            │
└──────────────┬────────────────────────────────┘
               │
        Discovers via
               │
       ┌───────▼──────────┐
       │     Consul       │
       │ - Service registry│
       │ - Health checks  │
       │ - Configuration  │
       └──────────────────┘
               ↑
          Register
               │
    ┌──────────┴──────────┐
    │                     │
┌───▼────┐         ┌──────▼───┐
│Service │         │ Service  │
│   A    │────────▶│    B     │
└────────┘  NATS   └──────────┘
       ↑    Events       ↓
       └──────────────────┘
```

### Complete Integration Example

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  # Consul for service discovery
  consul:
    image: consul:1.16
    command: agent -server -bootstrap-expect=1 -ui -client=0.0.0.0
    ports:
      - "8500:8500"
    networks:
      - infra

  # NATS for messaging
  nats:
    image: nats:2.10
    command: -js -m 8222
    ports:
      - "4222:4222"
      - "8222:8222"
    networks:
      - infra

  # Traefik with Consul provider
  traefik:
    image: traefik:v2.10
    command:
      - "--providers.consulcatalog=true"
      - "--providers.consulcatalog.endpoint.address=consul:8500"
      - "--providers.consulcatalog.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    depends_on:
      - consul
    networks:
      - infra

  # API service
  api:
    image: myapi:latest
    environment:
      - CONSUL_HTTP_ADDR=consul:8500
      - NATS_URL=nats://nats:4222
    depends_on:
      - consul
      - nats
    networks:
      - infra
    deploy:
      replicas: 3
      labels:
        # Consul metadata for Traefik
        - "traefik.enable=true"
        - "traefik.http.routers.api.rule=Host(`api.example.com`)"
        - "traefik.http.services.api.loadbalancer.server.port=8080"

networks:
  infra:
    driver: bridge
```

**Service implementation** (registers with Consul, subscribes to NATS):
```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "os"

    "github.com/hashicorp/consul/api"
    "github.com/nats-io/nats.go"
)

func main() {
    // Register with Consul
    consulClient, _ := api.NewClient(api.DefaultConfig())
    serviceID := registerService(consulClient)
    defer deregisterService(consulClient, serviceID)

    // Connect to NATS
    nc, _ := nats.Connect(os.Getenv("NATS_URL"))
    defer nc.Close()

    // Subscribe to events
    nc.Subscribe("orders.created", handleOrderCreated)

    // Start HTTP server
    http.HandleFunc("/health", healthCheck)
    http.HandleFunc("/api/orders", func(w http.ResponseWriter, r *http.Request) {
        createOrder(w, r, nc)
    })

    log.Println("Service starting on :8080")
    http.ListenAndServe(":8080", nil)
}

func registerService(client *api.Client) string {
    hostname, _ := os.Hostname()
    serviceID := "api-" + hostname

    registration := &api.AgentServiceRegistration{
        ID:      serviceID,
        Name:    "api",
        Port:    8080,
        Tags:    []string{"traefik.enable=true"},
        Check: &api.AgentServiceCheck{
            HTTP:     "http://" + hostname + ":8080/health",
            Interval: "10s",
            Timeout:  "3s",
        },
    }

    client.Agent().ServiceRegister(registration)
    log.Printf("Registered service: %s", serviceID)

    return serviceID
}

func deregisterService(client *api.Client, serviceID string) {
    client.Agent().ServiceDeregister(serviceID)
    log.Printf("Deregistered service: %s", serviceID)
}

func healthCheck(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}

func createOrder(w http.ResponseWriter, r *http.Request, nc *nats.Conn) {
    // Create order...
    order := map[string]interface{}{
        "order_id": "order-123",
        "user_id":  "user-456",
        "total":    99.99,
    }

    // Publish event
    data, _ := json.Marshal(order)
    nc.Publish("orders.created", data)

    // Respond
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(order)
}

func handleOrderCreated(msg *nats.Msg) {
    var order map[string]interface{}
    json.Unmarshal(msg.Data, &order)

    log.Printf("Processing order: %v", order)
    // Process order...
}
```

---

## Zero-Downtime Deployments

### Strategy 1: Rolling Updates

**How it works**:
1. Deploy new version to 1 server
2. Health check passes
3. Remove old version from 1 server
4. Repeat until all servers updated

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  api:
    image: myapi:${VERSION}
    deploy:
      replicas: 5
      update_config:
        parallelism: 1  # Update 1 at a time
        delay: 10s      # Wait 10s between updates
        order: start-first  # Start new before stopping old
        failure_action: rollback
      rollback_config:
        parallelism: 1
        delay: 5s
```

**Deploy**:
```bash
# Update to new version
VERSION=v2 docker-compose up -d --no-deps api

# Docker Compose will:
# 1. Start api_1 with v2
# 2. Wait for health check
# 3. Stop api_1 with v1
# 4. Repeat for api_2, api_3, etc.
```

### Strategy 2: Blue-Green Deployment

**How it works**:
1. Deploy new version (green) alongside old (blue)
2. Route small percentage of traffic to green
3. Monitor metrics
4. Gradually increase green traffic
5. Remove blue when green is 100%

**Implementation**:
```yaml
# docker-compose.yml
services:
  api-blue:
    image: myapi:v1
    deploy:
      replicas: 3
    labels:
      - "traefik.http.services.api.loadbalancer.server.weight=90"

  api-green:
    image: myapi:v2
    deploy:
      replicas: 3
    labels:
      - "traefik.http.services.api.loadbalancer.server.weight=10"
```

**Gradual migration**:
```bash
# 10% traffic to green
docker-compose up -d

# Monitor metrics...

# Increase to 50%
# (Update labels and restart)

# Increase to 100%
# (Update labels and restart)

# Remove blue
docker-compose stop api-blue
docker-compose rm api-blue
```

### Strategy 3: Canary Deployment

**How it works**:
1. Deploy new version to small subset (canary)
2. Route only specific users/regions to canary
3. Monitor error rates, latency
4. If healthy: expand canary
5. If unhealthy: rollback immediately

**Traefik configuration**:
```yaml
# Route by header for canary testing
labels:
  # Canary (new version)
  - "traefik.http.routers.api-canary.rule=Host(`api.example.com`) && Header(`X-Canary`, `true`)"
  - "traefik.http.routers.api-canary.service=api-v2"
  - "traefik.http.routers.api-canary.priority=100"

  # Stable (old version)
  - "traefik.http.routers.api-stable.rule=Host(`api.example.com`)"
  - "traefik.http.routers.api-stable.service=api-v1"
  - "traefik.http.routers.api-stable.priority=1"
```

Now users with `X-Canary: true` header get v2, everyone else gets v1.

---

## Troubleshooting Network Issues

### Issue 1: Service Not Discoverable

**Symptoms**: Requests fail with "service not found" or "no healthy instances"

**Debugging steps**:

1. **Check service registration**:
```bash
# List all services in Consul
curl http://consul:8500/v1/agent/services

# Check specific service
curl http://consul:8500/v1/health/service/api

# Look for:
# - Is service registered?
# - What's the health status?
# - Are there any instances?
```

2. **Check health checks**:
```bash
# View check status
curl http://consul:8500/v1/agent/checks

# Check logs
docker logs <service-container>
```

3. **Common causes**:
- Service didn't register (startup failed)
- Health check endpoint wrong
- Health check failing (return 200 OK)
- Network connectivity between service and Consul

**Fix**: Ensure service registers on startup and health endpoint works.

### Issue 2: Uneven Load Distribution

**Symptoms**: Some servers get all traffic, others idle

**Debugging**:

1. **Check Traefik dashboard**:
```
http://traefik:8080/dashboard/
→ View traffic distribution per backend
```

2. **Check load balancer algorithm**:
```yaml
# Verify configuration
labels:
  - "traefik.http.services.api.loadbalancer.server.port=8080"
  # Is sticky sessions enabled unnecessarily?
  - "traefik.http.services.api.loadbalancer.sticky.cookie=false"
```

3. **Check service weights**:
```yaml
# Are weights balanced?
- "traefik.http.services.api.loadbalancer.server.weight=1"
```

4. **Check health status**:
```bash
# Are all instances healthy?
curl http://consul:8500/v1/health/service/api?passing
```

**Fix**: Disable sticky sessions if not needed, ensure all instances healthy, verify equal weights.

### Issue 3: Intermittent Timeouts

**Symptoms**: Requests sometimes timeout, sometimes succeed

**Debugging**:

1. **Check timeout configuration**:
```yaml
# Traefik timeout settings
labels:
  - "traefik.http.services.api.loadbalancer.responseForwarding.flushInterval=100ms"
  - "traefik.http.services.api.loadbalancer.server.timeout=30s"
```

2. **Check health checks**:
```bash
# Are services flapping (going up/down)?
watch -n 1 'curl -s http://consul:8500/v1/health/service/api | jq ".[].Checks[].Status"'
```

3. **Check resource saturation**:
```promql
# In Prometheus/Grafana
container_memory_usage_bytes / container_spec_memory_limit_bytes * 100 > 90
container_cpu_usage_seconds_total > 0.9
```

4. **Check connection pool exhaustion**:
```bash
# Database connections maxed?
# Message queue full?
# Check application metrics
```

**Fix**: Increase timeouts, stabilize health checks, scale resources, increase connection pools.

### Issue 4: SSL/TLS Certificate Issues

**Symptoms**: HTTPS not working, certificate errors

**Debugging**:

1. **Check certificate resolver**:
```bash
# List certificates
docker exec traefik ls -la /letsencrypt/

# Check acme.json
docker exec traefik cat /letsencrypt/acme.json | jq
```

2. **Check Traefik logs**:
```bash
docker logs traefik | grep -i acme
docker logs traefik | grep -i certificate
```

3. **Check DNS**:
```bash
# Does domain point to server?
dig api.example.com

# Can Let's Encrypt reach server?
curl -I http://api.example.com/.well-known/acme-challenge/test
```

4. **Check rate limits**:
```
Let's Encrypt rate limits:
- 50 certificates per domain per week
- 5 duplicate certificates per week
```

**Fix**: Ensure DNS correct, port 80 accessible, wait if rate limited, check acme.json permissions.

---

## Best Practices for Networking

### DO These Things

✅ **Use service discovery** - No hardcoded IPs
✅ **Implement health checks** - Automatic failure detection
✅ **Enable TLS everywhere** - Encrypt all traffic
✅ **Monitor traffic patterns** - Understand your load
✅ **Use circuit breakers** - Prevent cascading failures
✅ **Implement rate limiting** - Protect from abuse
✅ **Log all requests** - Enable debugging
✅ **Use zero-downtime deployments** - No user impact
✅ **Test failover scenarios** - Verify resilience
✅ **Document network topology** - Team understanding

### DON'T Do These Things

❌ **Don't hardcode IPs** - Use service names
❌ **Don't skip health checks** - Services fail silently
❌ **Don't trust all traffic** - Validate and authenticate
❌ **Don't ignore timeouts** - Set realistic limits
❌ **Don't over-optimize** - Start simple, measure, improve
❌ **Don't forget DNS TTL** - Impacts failover speed
❌ **Don't deploy without testing** - Verify locally first
❌ **Don't ignore metrics** - You can't improve what you don't measure
❌ **Don't create single points of failure** - Everything redundant
❌ **Don't skip SSL certificates** - Security matters

---

## Chapter Summary

You've learned how to build a robust networking layer for your infrastructure:

**Core Concepts**:
- **Service discovery**: Dynamic service registration and discovery
- **Load balancing**: Intelligent traffic distribution
- **API gateway**: Centralized request routing and management
- **Event-driven**: Async communication for scalability
- **Zero-downtime**: Seamless deployments without user impact

**Practical Skills**:
- **Traefik**: Reverse proxy with automatic service discovery
- **Consul**: Service registry, health checking, configuration
- **NATS**: Lightweight messaging for event-driven architectures
- **Integration**: Combining all three for powerful infrastructure

**Production Patterns**:
- Circuit breaking for fault tolerance
- Health checks for automatic failure detection
- TLS everywhere for security
- Multiple deployment strategies (rolling, blue-green, canary)
- Systematic troubleshooting for network issues

**Key Takeaway**: Modern networking is dynamic, not static. Services discover each other automatically, traffic routes intelligently, and systems heal themselves. This is infrastructure that adapts to change instead of fighting it.

---

## What's Next: Chapter 6

In Chapter 6, we'll build **The Immune System (Security)**: CrowdSec, Fail2ban, Wazuh, and security patterns.

You'll learn:
- Intrusion detection and prevention systems
- Automated threat response with CrowdSec
- Log-based attack detection with Fail2ban
- Host-based intrusion detection with Wazuh
- Security hardening best practices
- Incident response workflows

**Preview**: Just as your body's immune system protects against pathogens, your infrastructure's security layer protects against attacks. We'll build defense-in-depth security that detects, responds to, and learns from threats automatically.

Ready to fortify your infrastructure? Turn to Chapter 6.

---

## Apex Networking Practices

Treat these combinations as the gold standard for modern circulatory systems:

- **Programmable Data Plane** — Deploy Cilium or Calico with eBPF for zero-copy packet processing, distributed firewalls, Hubble observability, and transparent encryption between workloads.
- **Service Mesh Excellence** — Run Istio or Linkerd with Envoy sidecars, SPIFFE identities, and mTLS by default; layer on circuit breaking, outlier detection, and rate limiting from the mesh instead of custom code.
- **Edge & Global Load Balancing** — Pair Cloudflare/Akamai global anycast with AWS Global Accelerator or GCP Cloud Load Balancing, and use NS1 or NSOne for traffic steering based on latency, geography, or health probes.
- **API Gateway with Policy Control** — Use Kong Gateway or Ambassador with OPA/Gatekeeper plugins, JWT/OAuth2 validation, and WASM filters for custom business logic while keeping ingress declarative.
- **Secure Connectivity Fabric** — Extend zero-trust to humans and services alike with WireGuard-based mesh (Tailscale, Netmaker) or Cloudflare Zero Trust, and broker database access through Boundary or Teleport for auditable sessions.
- **Network Automation & Testing** — Validate intent continuously with Batfish/Forward Networks, run chaos with Gremlin Network attacks, and codify routing policy in Nautobot or NetBox plus Ansible/Nornir for infrastructure as data.

Together, these practices give your organism elastic bandwidth, rock-solid security, and deep visibility from the core to the edge.
