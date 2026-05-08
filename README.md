# Hoppscotch goes Dokploy
After some brief document checking and testing I came up with the following [Dokploy](https://docs.dokploy.com/docs/core/installation) docker configuration to deploy [Hoppscotch](https://https://docs.hoppscotch.io/) on Dokploy. This doc only covers email authentication. If other authentication providers are required it needs to be activated (also requires oauth services deployed on the provider of choice). Hoppscotch comes with an open frontend that limits the requests to their own website. All other domains are blocked. For my taste that is still too open. Therefore I used the traeffik middleware (can after the container is once created)

> This doc only covers community version - not any enterprise features.

## Steps to take (tl;dr)
* Configure a new compose project in Dokploy
* Set the docker compose configurations
* Have a dedicated email account availble for SMTP auth
* Set the environment configuration
* Define a domain (use ssl is recommended)
* Deploy the container
* Get the container ID from the hoppscotch/hoppscotch:latest
* Run the database migration using `docker exec -it ... `
* Start configure the admin backend on https://apitester.mydomain.com/admin

### Optional add Basic auth
* Add the basic auth
* Re-Deploy the container
* Login to the website with basic auth

## Dokploy configuration
Create a new compose and add the the following docker compose configuration

```docker
services:
  apitester:
    image: hoppscotch/hoppscotch:latest
    expose:
      - "80"
    environment:
      # DB
      - DATABASE_URL=postgresql://${APITESTER_DB_USER}:${APITESTER_DB_PW}@db:5432/${APITESTER_DB}
      # Secrets
      - JWT_SECRET=${JWT_SECRET}
      - SESSION_SECRET=${SESSION_SECRET}
      - DATA_ENCRYPTION_KEY=${DATA_ENCRYPTION_KEY}
      # Token Config
      - TOKEN_SALT_COMPLEXITY=10
      - MAGIC_LINK_TOKEN_VALIDITY=3
      - REFRESH_TOKEN_VALIDITY=604800000
      - ACCESS_TOKEN_VALIDITY=86400000
      # Subpath & URLs
      - ENABLE_SUBPATH_BASED_ACCESS=true
      - VITE_BASE_URL=${APP_URL}
      - VITE_SHORTCODE_BASE_URL=${APP_URL}
      - VITE_ADMIN_URL=${APP_URL}/admin
      - VITE_BACKEND_GQL_URL=${APP_URL}/backend/graphql
      - VITE_BACKEND_WS_URL=${APP_WS_URL}/backend/graphql
      - VITE_BACKEND_API_URL=${APP_URL}/backend/v1
      - WHITELISTED_ORIGINS=${APP_URL},${APP_URL}/admin,${APP_URL}/backend
      # Auth
      - VITE_ALLOWED_AUTH_PROVIDERS=EMAIL
      # SMTP
      - MAILER_USE_CUSTOM_CONFIGS=true
    networks:
      - hoppscotch_net
      - dokploy-network
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
  db:
    networks: 
      - hoppscotch_net
    image: postgres:15
    environment:
      POSTGRES_USER: ${APITESTER_DB_USER}
      POSTGRES_PASSWORD: ${APITESTER_DB_PW}
      POSTGRES_DB: ${APITESTER_DB}
    volumes:
      - apitester_db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${APITESTER_DB_USER}"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped
networks:
  hoppscotch_net:
    driver: bridge
  dokploy-network:
    external: true
volumes:
  apitester_db:
```

Setup a domain (eg apitester.mydomain.com) and configure the environment using the following configuration

> Try to prevent special chars (eg $ etc.). That may break your deployment.

The configuration:
```env
APP_URL=https://apitester.mydomain.com
APP_WS_URL=wss://apitester.mydomain.com

APITESTER_DB_USER=apitester
APITESTER_DB_PW=myPhatDBPassw0rd
APITESTER_DB=apitester
DATABASE_URL=postgresql://apitester:myPhatDBPassw0rd:5432/apitester
JWT_SECRET=myPhatJWTS3cret
SESSION_SECRET=myPhatS3ss1nSecret (64 chars)
DATA_ENCRYPTION_KEY=theEp1cDbEncryptionKey (32 chars)
TOKEN_SALT_COMPLEXITY=10
MAGIC_LINK_TOKEN_VALIDITY=3
REFRESH_TOKEN_VALIDITY=604800000
ACCESS_TOKEN_VALIDITY=86400000
ENABLE_SUBPATH_BASED_ACCESS=true
VITE_BASE_URL=${{APP_URL}}
VITE_SHORTCODE_BASE_URL=${{APP_URL}}
VITE_ADMIN_URL=${{APP_URL}}/admin
VITE_BACKEND_GQL_URL=${{APP_URL}}/backend/graphql
VITE_BACKEND_WS_URL=${{APP_WS_URL}}/backend/graphql
VITE_BACKEND_API_URL=${{APP_URL}}/backend/v1
WHITELISTED_ORIGINS=${{APP_URL}},${{APP_URL}}/admin,${{APP_URL}}/backend,app://hoppscotch
REDIRECT_URL=${{APP_URL}}
```

> Don't forget to save :-)

Now deploy the container. The container may start but will restart all the time due to the missing database (migration required)

After the successful deployment go to your dokploy server that runs the container and get the container ID

```bash
docker ps --filter "ancestor=hoppscotch/hoppscotch:latest" --format "{{.ID}}"
```
Then use the ID to do the DB migration with the following command
```bash
docker exec -it <theId> sh -c "cd /dist/backend && npx prisma migrate deploy"
```

## Adding htaccess to the container
### Steps to take

1) create a new htacces configuration with (requirees htpasswd installed on your system)
```bash
htpasswd -nb myBasicAuthUser myBasicAuthPassword
```
2) Use the service name to inspect the get the routere name (requires acces to the dokploy server)
```bash
docker inspect $(docker ps | grep apitester | awk '{print $1}') | grep "router.*websecure" | head -3
```
Extend the service by adding the following labels by using the dokploy project name YYYY (eg my-tools) and XXXX (the router name). Do also escape the \$-char: change from  \$ to \$\$)

```docker
services:
  apitester:
    image: hoppscotch/hoppscotch:latest
    labels:
      # basic auth
      - traefik.http.middlewares.apitester-auth.basicauth.users=myBasicAuthUser:$$apr1$$c0wG3biY$$iX5.CeMGCP9WfvrV.kJ221
      # traeffic middleware configuration
      - traefik.http.routers.YYYY-XXXX-websecure.middlewares=apitester-auth@docker
```
4) Re-deploy the container
