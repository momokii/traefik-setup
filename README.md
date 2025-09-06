# Traefik Setup Examples & Tutorials

This repository contains practical examples and configurations for implementing advanced Traefik features in production environments. Each setup demonstrates best practices for common use cases including SSL/TLS termination, load balancing, middleware implementation, and resource optimization.

## Table of Contents

- [Traefik Setup Examples \& Tutorials](#traefik-setup-examples--tutorials)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Quick Start](#quick-start)
  - [Configuration Examples](#configuration-examples)
    - [SSL/TLS Setup](#ssltls-setup)
    - [Canary Deployment \& Load Balancing](#canary-deployment--load-balancing)
    - [Middleware Configuration](#middleware-configuration)
    - [Zero-Scale with Sablier Plugin](#zero-scale-with-sablier-plugin)
  - [Network Architecture](#network-architecture)
  - [Configuration Files](#configuration-files)
    - [Core Files](#core-files)
    - [Service Examples](#service-examples)
  - [Best Practices](#best-practices)
    - [Security](#security)
    - [Performance](#performance)
    - [Configuration Management](#configuration-management)
    - [SSL/TLS](#ssltls)
  - [Troubleshooting](#troubleshooting)
    - [Common Issues](#common-issues)
    - [Useful Commands](#useful-commands)
    - [Configuration Validation](#configuration-validation)
  - [Additional Resources](#additional-resources)

## Overview

This setup demonstrates four key Traefik implementations:

1. **SSL/TLS Automatic Certificate Management** - Let's Encrypt integration with HTTP-01 challenge
2. **Weighted Round Robin Load Balancing** - Traffic distribution for A/B testing and canary deployments
3. **Advanced Middleware** - Rate limiting and IP allowlist for security
4. **Resource Optimization** - Automatic service scaling to zero using Sablier plugin

## Prerequisites

- Docker and Docker Compose installed
- Domain name(s) pointing to your server
- Basic understanding of Docker networking
- Public IP address for SSL certificate validation

## Quick Start

1. **Create the external network:**
   ```bash
   docker network create traefik-networks
   ```

2. **Configure your domains:**
   - Update `YOUR-DOMAIN` placeholders in configuration files
   - Set your email address in `traefik/traefik-config.yaml`
   - Configure your IP address in `traefik/dynamic-config.yaml`

3. **Start Traefik:**
   ```bash
   cd traefik
   docker-compose up -d
   ```

4. **Deploy example services:**
   ```bash
   # SSL setup example
   cd ../ssl-setup-test
   docker-compose up -d
   
   # Canary deployment example
   cd ../canary-deployment-test
   docker-compose up -d
   
   # Sablier zero-scale example
   cd ../sablier-test-zero-scale
   docker-compose up -d
   ```

## Configuration Examples

### SSL/TLS Setup

**Location:** `ssl-setup-test/`

Demonstrates automatic SSL certificate provisioning using Let's Encrypt with HTTP-01 challenge.

**Key Features:**
- Automatic certificate generation and renewal
- HTTP to HTTPS redirection
- Rate limiting middleware (3 requests per 10 seconds)
- IP allowlist security

**Configuration Highlights:**
```yaml
# Automatic HTTPS redirect
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https

# Let's Encrypt resolver
certificatesResolvers:
  myresolver:
    acme:
      email: YOUR_EMAIL_HERE
      storage: "/letsencrypt/acme.json"
      httpChallenge:
        entrypoint: web
```

**Docker Labels Example:**
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.app1-router.rule=Host(`your-domain.com`)"
  - "traefik.http.routers.app1-router.entrypoints=websecure"
  - "traefik.http.routers.app1-router.tls.certresolver=myresolver"
  - "traefik.http.routers.app1-router.middlewares=test-ratelimit@docker,test-ipallowlist@file"
```

### Canary Deployment & Load Balancing

**Location:** `canary-deployment-test/`

Implements weighted round-robin load balancing for A/B testing and gradual feature rollouts.

**Key Features:**
- Traffic splitting (90% to v1, 10% to v2)
- Independent service scaling
- Dynamic configuration management

**Traffic Distribution:**
- **Version 1 (Stable):** 90% of traffic
- **Version 2 (Canary):** 10% of traffic

**Dynamic Configuration:**
```yaml
http:
  services:
    app-wrr-service:
      weighted:
        services:
        - name: app-v1-service@docker
          weight: 9  # 90% traffic
        - name: app-v2-service@docker
          weight: 1  # 10% traffic
```

### Middleware Configuration

**Location:** Defined in `traefik/dynamic-config.yaml` and service labels

Showcases security and performance middleware implementations.

**Available Middlewares:**

1. **Rate Limiting:**
   ```yaml
   middlewares:
     test-ratelimit:
       ratelimit:
         average: 3      # Average requests allowed
         burst: 3        # Maximum burst requests
         period: 10      # Time period (seconds)
   ```

2. **IP Allowlist:**
   ```yaml
   middlewares:
     test-ipallowlist:
       ipAllowList:
         sourceRange:
           - "YOUR_IP_ADDRESS"
   ```

**Middleware Chaining:**
```yaml
# Multiple middlewares can be applied
middlewares: "test-ratelimit@docker,test-ipallowlist@file"
```

### Zero-Scale with Sablier Plugin

**Location:** `sablier-test-zero-scale/`

Automatically scales services to zero when not in use, reducing resource consumption.

**Key Features:**
- Automatic service hibernation after inactivity
- Custom wake-up page with configurable themes
- Session-based activity tracking
- Resource cost optimization

**Sablier Configuration:**
```yaml
middlewares:
  my-sablier:
    plugin:
      sablier:
        sablierUrl: http://sablier:10000
        sessionDuration: 1m
        names: whoami_sablier
        dynamic:
          displayName: Save Your Resources!
          refreshFrequency: 5s
          showDetails: "true"
          theme: ghost
```

**Benefits:**
- **Cost Reduction:** Services consume zero resources when idle
- **Environmental Impact:** Reduced server load and energy consumption
- **Automatic Management:** No manual intervention required

## Network Architecture

```
Internet → Traefik (Entry Point) → Docker Services
    ↓
[Port 80] → [Port 443] (SSL Redirect)
    ↓
Traefik Router → Middleware Chain → Backend Service
```

**Network Requirements:**
- External network: `traefik-networks`
- All services must be connected to this network
- Traefik container exposes ports 80, 443, and 8080

## Configuration Files

### Core Files

| File | Purpose | Configuration Type |
|------|---------|-------------------|
| `traefik-config.yaml` | Main Traefik configuration | Static |
| `dynamic-config.yaml` | Routes, services, middleware | Dynamic |
| `compose.yaml` | Traefik container setup | Infrastructure |

### Service Examples

| Directory | Use Case | Features |
|-----------|----------|----------|
| `ssl-setup-test/` | SSL/TLS implementation | Auto certificates, rate limiting |
| `canary-deployment-test/` | Load balancing | Weighted routing, A/B testing |
| `sablier-test-zero-scale/` | Resource optimization | Auto-scaling, hibernation |

## Best Practices

### Security
- **Always use HTTPS in production**
- **Implement rate limiting on public endpoints**
- **Configure IP allowlists for sensitive services**
- **Regularly update Traefik and plugin versions**
- **Store certificates in persistent volumes**

### Performance
- **Use external networks for service communication**
- **Enable file watching for dynamic configuration**
- **Implement health checks for backend services**
- **Monitor resource usage and scaling patterns**

### Configuration Management
- **Separate static and dynamic configurations**
- **Use environment variables for sensitive data**
- **Version control all configuration files**
- **Test configurations in staging environments**

### SSL/TLS
- **Use HTTP-01 challenge for most scenarios**
- **Ensure DNS points to your server before certificate generation**
- **Monitor certificate renewal logs**
- **Backup `acme.json` file regularly**

## Troubleshooting

### Common Issues

**SSL Certificate Generation Fails:**
```bash
# Check if domain points to your server
nslookup your-domain.com

# Verify Traefik logs
docker logs traefik-test

# Ensure port 80 is accessible from internet
```

**Service Not Accessible:**
```bash
# Check if service is on the correct network
docker network inspect traefik-networks

# Verify Traefik can reach the service
docker exec traefik-test ping service-name
```

**Sablier Not Working:**
```bash
# Check Sablier logs
docker logs sablier

# Verify plugin installation
# Check Traefik dashboard at http://localhost:8080
```

### Useful Commands

```bash
# View Traefik configuration
docker exec traefik-test cat /etc/traefik/traefik.yaml

# Check certificate status
docker exec traefik-test cat /letsencrypt/acme.json

# Monitor real-time logs
docker logs -f traefik-test

# Test service connectivity
docker exec traefik-test wget -qO- http://service-name:port
```

### Configuration Validation

- **Traefik Dashboard:** `http://localhost:8080` (when `api.insecure: true`)
- **Configuration API:** `http://localhost:8080/api/rawdata`
- **Health Check:** `http://localhost:8080/ping`

## Additional Resources

- [Traefik Official Documentation](https://doc.traefik.io/traefik/)
- [Sablier Plugin Documentation](https://github.com/sablierapp/sablier)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Docker Networking Guide](https://docs.docker.com/network/)

---

**Note:** Remember to replace all placeholder values (`YOUR-DOMAIN`, `YOUR_EMAIL_HERE`, `YOUR_IP_ADDRESS`) with your actual configuration before deployment.