# SSL Setup Example

whoami service with rate limiting (3 req/10sec) and IP allowlist.

## How to activate

1. Replace `YOUR-DOMAIN` in the compose labels with your domain
2. Replace `YOUR_IP_ADDRESS` in `traefik/dynamic.yaml` under `test-ipallowlist` with your IP
3. Add DNS A record for the domain pointing to your VM
4. Start the service:

```bash
docker compose up -d
```

5. Access the domain — only requests from your IP will be allowed, and rate limited to 3 per 10 seconds

## To deactivate

```bash
docker compose down
```
