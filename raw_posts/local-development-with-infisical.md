# Syncing Developer Secrets With Infisical

**By FerretCode**

## So what is Infisical?

When working with a team of developers, it's important to keep development & production secrets alike synced between each other to ensure a consistent state. However, there are several secret management tools that could fit the bill for this requirement.

While SOPS, HashiCorp vault, and more could be good candidates for our secret management flow, I've found that Infisical has been my favorite platform to work with for this purpose.

Infisical is an open-source secret management tool which is both self-hostable and also exposes an Infisical-hosted cloud dashboard. It has multiple integrations for Docker, docker-compose, Kubernetes, and more to sync secrets from your Infisical instance to other developer machines & in production.

## Building the Initial Local Development Flow

In one of our recent projects, I was tasked with setting up our Infisical -> local & production syncing mechanism. For this project, our backend stack was Go, TimescaleDB, MediaMTX for RTSP, and RabbitMQ. In local development, we used Make & docker-compose and eventually deployed into production on a Kubernetes cluster.

At first, our approach was to take a local `.env` file containing an Infisical machine identity & project ID to generate an Infisical auth token that can be used by the CLI to pull secrets.

`.env`:

```.env
INFISICAL_CLIENT_ID=
INFISICAL_CLIENT_SECRET=
PROJECT_ID=
```

`Makefile`:

```Makefile
include .env

.PHONY up
up:
    INFISICAL_TOKEN=$$(infisical login --method=universal-auth --client-id=$(INFISICAL_CLIENT_ID) --client-secret=$(INFISICAL_CLIENT_SECRET) --silent --plain) \
	PROJECT_ID=$(PROJECT_ID) docker-compose up --build
```

From there, we would add an entrypoint to each service in our docker-compose config to use infisical CLI & pull the secrets that way:

`docker-compose.yaml`:

```yaml
services:
    app:
        environment:
            - INFISICAL_TOKEN=${INFISICAL_TOKEN}
            - PROJECT_ID=${PROJECT_ID}
        depends_on:
            postgres:
                condition: service_healthy
            rabbitmq:
                condition: service_healthy
        ports:
            - "4000:4000"
        build:
            context: .
            dockerfile: Dockerfile
        volumes:
            - ./:/app
        entrypoint:
            - infisical
            - run
            - --projectId
            - ${PROJECT_ID}
            - --
            - air
            - -c
            - .air.toml
    postgres:
        image: postgres:latest
        restart: unless-stopped
        hostname: postgres.local
        healthcheck:
        test: ["CMD-SHELL", "pg_isready -U postgres"]
        interval: 5s
        timeout: 20s
        retries: 5
        ports:
            - "5432:5432"
        environment:
            - INFISICAL_TOKEN=${INFISICAL_TOKEN}
            - PROJECT_ID=${PROJECT_ID}
        entrypoint:
            - /bin/sh
            - -c
            - |
                # Install Infisical CLI
                apt-get update && apt-get install -y bash curl && curl -1sLf \
                    'https://dl.cloudsmith.io/public/infisical/infisical-cli/setup.deb.sh' | bash \
                    && apt-get update && apt-get install -y infisical

                infisical run --projectId=$PROJECT_ID -- docker-entrypoint.sh postgres
        volumes:
            - postgres-data:/var/lib/postgresql/data
```

While this did work, you can probably see that this solution was ugly, inflexible, and cumbersome. Having to add a new entrypoint for each service & conforming to the image's base operating system was definitely not a very good flow. Additionally, you couldn't specify which environment variables each service needed, so every service would be injected with every environment variable -- definitely not good.

## Finding an Optimal Solution

The solution we decided on was as follows:

From the perspective of the developer, you would:

1. Install the Infisical CLI locally
2. Authenticate by either:

    - Logging in with `infisical login` through the web dashboard
    - Generating an Infisical token with a machine identity: `export INFISICAL_TOKEN=$(infisical login --method=universal-auth --client-id=<identity-client-id> --client-secret=<identity-client-secret> --silent --plain)`

3. `make up`
4. That's it.

Here were the changes we made our Docker setup:

Instead of generating an Infisical token within our Makefile & using Docker entrypoints, we changed our makefile to directly inject the secrets into the local environment:

```Makefile
.PHONY up
up:
    infisical run -- docker-compose up --build
```

With the secrets now in our local environment, we could inject them directly into each service in our docker-compose:

```yaml
services:
    app:
        build:
            context: .
            dockerfile: Dockerfile
        ports:
            - 4000:4000
        environment:
            - PORT
            - DEV
            - DATABASE_URL
            - RATELIMIT_RPS
            - RATELIMIT_MAX_BURST
            - RATELIMIT_ENABLED
            - CORS_TRUSTED_ORIGINS
            - RABBITMQ_URL
        volumes:
            - ./:/app
        depends_on:
            - postgres
            - rabbitmq
```

This flow works a lot better:

-   It's much cleaner
-   Allows for selective secret injection
-   Simpler on the developer side as well since they don't have to configure a `.env`

## Production Flow

Finally, for our production flow. We use Kubernetes to deploy our services, and conveniently, Infiscial has a native Kubernetes operator to sync secrets into a single Kubernetes secret from the dashboard.

The config for each environment in Kubernetes looks something like this:

```yaml
apiVersion: secrets.infisical.com/v1alpha1
kind: InfisicalSecret
metadata:
    name: infisicalsecret
    namespace: project-name
spec:
    hostAPI: https://app.infisical.com/api
    resyncInterval: 15
    authentication:
        universalAuth:
            secretsScope:
                projectSlug: slug
                envSlug: prod
                secretsPath: "/"
            credentialsRef:
                secretName: universal-auth-credentials
                secretNamespace: default

    managedKubeSecretReferences:
        - secretName: managed-secret
          secretNamespace: project-namespace
          creationPolicy: "Orphan"
```

Each `InfisicalSecret` resource is used to configure how, where, and how often the Infisical Operator will sync secrets from the cloud (or a self-hosted instance).

From there, once the secrets are consolidated, you can inject them selectively into your Kubernetes deployments:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: project-api
  labels:
    app: project-api
  namespace: project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: project-api
  template:
    metadata:
      labels:
        app: project-api
    spec:
      containers:
        - name: project-api
          image: app-image:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: PORT
              valueFrom:
                secretKeyRef:
                  name: managed-secret
                  key: PORT
                  optional: false
            - name: DEV
              valueFrom:
                secretKeyRef:
                  name: managed-secret
                  key: DEV
                  optional: false
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: managed-secret
                  key: DATABASE_URL
                  optional: false
            ...
```

## To Recap

Throughout this project, we developed a robust way of syncing secrets between local developer machines & Infisical to ensure a consistent environment between developers. Additionally, we used the Infisical Kubernetes Operator to sync production secrets into the cluster from the cloud.
