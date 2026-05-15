# Canary Deployment Example

Weighted round-robin load balancing: 90% traffic to v1, 10% to v2.

## How to activate

1. Uncomment the canary sections in `traefik/dynamic.yaml`:
   - `app-ab-test-router` (router)
   - `app-wrr-service` (weighted service)
2. Replace `YOUR-AB-TEST-DOMAIN` with your domain
3. Add DNS A record for the domain pointing to your VM
4. Start the services:

```bash
docker compose up -d
```

5. Send requests to the domain — ~90% will show v1 content, ~10% v2

## To deactivate

1. Comment the sections back in `dynamic.yaml`
2. `docker compose down`
3. Remove DNS record
