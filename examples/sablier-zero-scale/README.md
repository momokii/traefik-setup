# Sablier Zero-Scale Example

Auto-hibernate idle containers. Shows a loading page while the container starts up.

## How to activate

1. Uncomment the sablier plugin in `traefik/traefik.yaml`:
   ```yaml
   experimental:
     plugins:
       sablier:
         moduleName: "github.com/sablierapp/sablier"
         version: "v1.10.1"
   ```

2. Uncomment the sablier sections in `traefik/dynamic.yaml`:
   - `whoami-sablier-router` (router)
   - `whoami-sablier-service` (service)
   - `my-sablier` (middleware)

3. Replace `YOUR-DOMAIN` with your domain
4. Restart Traefik (plugin changes need restart):
   ```bash
   docker compose restart
   ```

5. Add DNS A record for the domain pointing to your VM
6. Start the services:
   ```bash
   docker compose up -d
   ```

7. Access the domain — you'll see a ghost loading page while the container starts, then the whoami response

## To deactivate

1. Comment the sections back in both configs
2. `docker compose down`
3. `docker compose restart` (to disable the plugin)
4. Remove DNS record
