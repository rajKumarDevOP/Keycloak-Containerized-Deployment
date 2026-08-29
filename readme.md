# Keycloak Deployment

Docker Compose based deployment of **Keycloak 16.1.0** with MySQL, a custom Keycloak provider, and a customized NMIMS login theme.

---

## 1. Overview

This project provides a containerized deployment of Keycloak using Docker Compose.

The deployment includes:

* Keycloak `16.1.0`
* MySQL `8.0`
* Docker Compose
* Persistent MySQL storage
* Custom Keycloak provider
* Custom Keycloak login theme
* NMIMS branding
* Reverse proxy support
* HTTPS frontend URL configuration
* JVM memory configuration
* Token Exchange feature
* Fine-grained Admin Authorization

---

## 2. Architecture

```text
                         ┌──────────────────────┐
                         │      LMS Users       │
                         └──────────┬───────────┘
                                    │
                                    │ HTTPS
                                    ▼
                         ┌──────────────────────┐
                         │ Reverse Proxy / LB   │
                         │ lms.svkm.ac.in/kck   │
                         └──────────┬───────────┘
                                    │
                                    │ HTTP
                                    ▼
              ┌────────────────────────────────────────┐
              │             Docker Host                │
              │                                        │
              │   ┌──────────────────────────────┐     │
              │   │      Keycloak Container      │     │
              │   │                              │     │
              │   │      Keycloak 16.1.0         │     │
              │   │                              │     │
              │   │  Custom Provider             │     │
              │   │  Custom NMIMS Theme          │     │
              │   │                              │     │
              │   │  Container Port: 8080        │     │
              │   └──────────────┬───────────────┘     │
              │                  │                     │
              │                  │ MySQL               │
              │                  ▼                     │
              │   ┌──────────────────────────────┐     │
              │   │       MySQL Container        │     │
              │   │                              │     │
              │   │       MySQL 8.0              │     │
              │   │       Database: keycloak     │     │
              │   │                              │     │
              │   │       Persistent Volume      │     │
              │   └──────────────────────────────┘     │
              │                                        │
              └────────────────────────────────────────┘
```

---

# 3. Project Structure

```text
keycloak-deployment/
│
├── docker-compose.yml
├── README.md
│
├── provider/
│   └── keycloak-lms-provider.jar
│
├── login.ftl
├── info.ftl
├── error.ftl
├── saml-post-form.ftl
├── messages_en.properties
└── nmims-logo.png
```

### File Description

| File / Directory            | Description                          |
| --------------------------- | ------------------------------------ |
| `docker-compose.yml`        | Docker Compose configuration         |
| `provider/`                 | Custom Keycloak provider             |
| `keycloak-lms-provider.jar` | Custom Keycloak provider JAR         |
| `login.ftl`                 | Customized login page                |
| `info.ftl`                  | Customized information page          |
| `error.ftl`                 | Customized error page                |
| `saml-post-form.ftl`        | Customized SAML POST form            |
| `messages_en.properties`    | Customized Keycloak messages         |
| `nmims-logo.png`            | NMIMS branding/logo                  |
| `README.md`                 | Project and deployment documentation |

---

# 4. Prerequisites

The deployment server must have:

* Docker
* Docker Compose
* Sufficient CPU and memory
* Network access for pulling Docker images
* DNS configured for the application domain
* Reverse proxy/load balancer if HTTPS is terminated externally

Verify Docker:

```bash
docker --version
```

Verify Docker Compose:

```bash
docker compose version
```

---

# 5. Docker Compose Configuration

The complete deployment is managed through `docker-compose.yml`.

```yaml
version: '3.8'

services:

  # ==================================================================
  # MySQL Database
  # ==================================================================
  mysql:
    image: mysql:8.0
    container_name: keycloak-mysql

    environment:
      MYSQL_DATABASE: keycloak
      MYSQL_USER: kcuser
      MYSQL_PASSWORD: kcpass
      MYSQL_ROOT_PASSWORD: rootpass

    command:
      - --default-authentication-plugin=mysql_native_password

    volumes:
      - mysql_data:/var/lib/mysql

    restart: unless-stopped

    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "7"


  # ==================================================================
  # Keycloak
  # ==================================================================
  keycloak:
    image: quay.io/keycloak/keycloak:16.1.0
    container_name: keycloak

    restart: unless-stopped

    environment:

      # --------------------------------------------------------------
      # Database Configuration
      # --------------------------------------------------------------
      DB_VENDOR: mysql
      DB_ADDR: mysql
      DB_DATABASE: keycloak
      DB_USER: kcuser
      DB_PASSWORD: kcpass

      # --------------------------------------------------------------
      # Keycloak Administrator
      # --------------------------------------------------------------
      KEYCLOAK_USER: kckp-admin
      KEYCLOAK_PASSWORD: admin1@2o26#

      # --------------------------------------------------------------
      # Reverse Proxy / Frontend Configuration
      # --------------------------------------------------------------
      PROXY_ADDRESS_FORWARDING: "true"
      KEYCLOAK_FRONTEND_URL: https://lms.svkm.ac.in/kck

      # --------------------------------------------------------------
      # JVM Configuration
      # --------------------------------------------------------------
      JAVA_OPTS: >-
        -Xms512m
        -Xmx1024m
        -Djboss.as.management.blocking.timeout=1200
        -Dkeycloak.profile.feature.token_exchange=enabled
        -Dkeycloak.profile.feature.admin_fine_grained_authz=enabled

    # --------------------------------------------------------------
    # Custom Provider and Theme
    # --------------------------------------------------------------
    volumes:

      # Custom Keycloak Provider
      - ./provider/keycloak-lms-provider.jar:/opt/jboss/keycloak/standalone/deployments/keycloak-lms-provider.jar

      # Custom Login Theme
      - ./login.ftl:/opt/jboss/keycloak/themes/nmims/login/login.ftl

      # Custom Logo
      - ./nmims-logo.png:/opt/jboss/keycloak/themes/nmims/login/resources/img/nmims-logo.png

      # Custom Information Page
      - ./info.ftl:/opt/jboss/keycloak/themes/nmims/login/info.ftl

      # Custom Error Page
      - ./error.ftl:/opt/jboss/keycloak/themes/nmims/login/error.ftl

      # Custom SAML POST Form
      - ./saml-post-form.ftl:/opt/jboss/keycloak/themes/nmims/login/saml-post-form.ftl

      # Custom Messages
      - ./messages_en.properties:/opt/jboss/keycloak/themes/nmims/login/messages/messages_en.properties

    # --------------------------------------------------------------
    # Port Mapping
    # --------------------------------------------------------------
    ports:
      - "8083:8080"

    # --------------------------------------------------------------
    # Start MySQL Before Keycloak
    # --------------------------------------------------------------
    depends_on:
      - mysql

    # --------------------------------------------------------------
    # Keycloak Startup
    # --------------------------------------------------------------
    command:
      - "-b"
      - "0.0.0.0"


# ==================================================================
# Persistent Volumes
# ==================================================================
volumes:
  mysql_data:
```

---

# 6. MySQL Configuration

MySQL runs as a separate Docker container.

```yaml
mysql:
  image: mysql:8.0
```

The MySQL container is named:

```text
keycloak-mysql
```

The database configuration is:

| Setting        | Value            |
| -------------- | ---------------- |
| Image          | `mysql:8.0`      |
| Database       | `keycloak`       |
| User           | `kcuser`         |
| Password       | `kcpass`         |
| Root Password  | `rootpass`       |
| Container Name | `keycloak-mysql` |

---

## MySQL Persistence

MySQL data is stored in a Docker named volume:

```yaml
volumes:
  - mysql_data:/var/lib/mysql
```

The volume is declared at the bottom of the Compose file:

```yaml
volumes:
  mysql_data:
```

This ensures that MySQL data is retained when the container is recreated.

Check the volume:

```bash
docker volume ls
```

Inspect it:

```bash
docker volume inspect keycloak-deployment_mysql_data
```

---

# 7. Keycloak Database Configuration

Keycloak connects to the MySQL service using the Docker Compose service name:

```yaml
DB_ADDR: mysql
```

The important point is that `mysql` is the internal Docker DNS name.

```text
Keycloak
   │
   │ DB_ADDR=mysql
   │
   ▼
MySQL Container
```

Keycloak configuration:

```yaml
DB_VENDOR: mysql
DB_ADDR: mysql
DB_DATABASE: keycloak
DB_USER: kcuser
DB_PASSWORD: kcpass
```

There is no need to expose MySQL's port to the host because Keycloak communicates with MySQL through the internal Docker network.

---

# 8. Keycloak Configuration

Keycloak uses:

```yaml
image: quay.io/keycloak/keycloak:16.1.0
```

The container is named:

```text
keycloak
```

---

## Port Mapping

```yaml
ports:
  - "8083:8080"
```

This means:

```text
Host Port             Container Port
─────────             ───────────────
8083        ────────► 8080
```

Keycloak can therefore be accessed from the Docker host through:

```text
http://SERVER-IP:8083
```

---

# 9. Keycloak Administrator

The initial Keycloak administrator is configured using:

```yaml
KEYCLOAK_USER: kckp-admin
KEYCLOAK_PASSWORD: admin1@2o26#
```

For production deployments, these credentials should not be stored directly in the Compose file.

Use environment variables or a secrets-management solution instead.

---

# 10. Reverse Proxy Configuration

The deployment is configured to run behind a reverse proxy.

```yaml
PROXY_ADDRESS_FORWARDING: "true"
```

The public Keycloak URL is:

```yaml
KEYCLOAK_FRONTEND_URL: https://lms.svkm.ac.in/kck
```

Expected traffic flow:

```text
User
 │
 │ HTTPS
 ▼
https://lms.svkm.ac.in/kck
 │
 ▼
Reverse Proxy
 │
 │ HTTP
 ▼
Docker Host:8083
 │
 ▼
Keycloak Container:8080
```

The reverse proxy should correctly forward the original host and protocol information.

---

# 11. JVM Configuration

Keycloak is configured with:

```text
-Xms512m
-Xmx1024m
```

### JVM Memory

| Parameter    |   Value |
| ------------ | ------: |
| Initial Heap |  512 MB |
| Maximum Heap | 1024 MB |

Additional JVM configuration:

```text
-Djboss.as.management.blocking.timeout=1200
```

This increases the JBoss management blocking timeout.

---

# 12. Enabled Keycloak Features

## Token Exchange

```text
-Dkeycloak.profile.feature.token_exchange=enabled
```

Token Exchange functionality is enabled through the JVM configuration.

---

## Fine-Grained Admin Authorization

```text
-Dkeycloak.profile.feature.admin_fine_grained_authz=enabled
```

This enables fine-grained authorization functionality for Keycloak administration.

---

# 13. Custom Keycloak Provider

A custom provider is included in:

```text
provider/keycloak-lms-provider.jar
```

The JAR is mounted into:

```text
/opt/jboss/keycloak/standalone/deployments/
```

Docker Compose configuration:

```yaml
- ./provider/keycloak-lms-provider.jar:/opt/jboss/keycloak/standalone/deployments/keycloak-lms-provider.jar
```

Verify the provider:

```bash
docker exec keycloak ls -lh \
  /opt/jboss/keycloak/standalone/deployments/
```

Expected:

```text
keycloak-lms-provider.jar
```

Check the Keycloak logs for deployment errors:

```bash
docker logs keycloak
```

---

# 14. Custom Keycloak Theme

The deployment uses a custom theme named:

```text
nmims
```

The theme is mounted into:

```text
/opt/jboss/keycloak/themes/nmims/login/
```

The following files are customized:

```text
login.ftl
info.ftl
error.ftl
saml-post-form.ftl
messages/messages_en.properties
resources/img/nmims-logo.png
```

This provides customization for:

* Login page
* Error page
* Information page
* SAML POST page
* Login messages
* NMIMS logo and branding

---

# 15. Deployment

## Step 1 — Navigate to Project

```bash
cd /data/keycloak-deployment
```

---

## Step 2 — Verify Files

```bash
ls -la
```

Expected:

```text
docker-compose.yml
README.md
login.ftl
info.ftl
error.ftl
saml-post-form.ftl
messages_en.properties
nmims-logo.png
provider/
```

Verify provider:

```bash
ls -lh provider/
```

Expected:

```text
keycloak-lms-provider.jar
```

---

## Step 3 — Validate Compose File

Before starting the application:

```bash
docker compose config
```

If the configuration is valid, Docker Compose will output the resolved configuration without reporting YAML errors.

---

## Step 4 — Pull Images

```bash
docker compose pull
```

This downloads:

```text
mysql:8.0
quay.io/keycloak/keycloak:16.1.0
```

---

## Step 5 — Start the Stack

```bash
docker compose up -d
```

---

# 16. Verify Deployment

Check containers:

```bash
docker compose ps
```

Expected services:

```text
keycloak
keycloak-mysql
```

You can also check:

```bash
docker ps
```

---

# 17. Check MySQL

View MySQL logs:

```bash
docker logs keycloak-mysql
```

Follow logs:

```bash
docker logs -f keycloak-mysql
```

Check MySQL container:

```bash
docker inspect keycloak-mysql
```

---

# 18. Check Keycloak

View Keycloak logs:

```bash
docker logs keycloak
```

Follow logs:

```bash
docker logs -f keycloak
```

Check container:

```bash
docker inspect keycloak
```

---

# 19. Verify Network Connectivity

List Docker networks:

```bash
docker network ls
```

Inspect the Compose network:

```bash
docker network inspect keycloak-deployment_default
```

Both containers should be connected to the same Docker network.

```text
keycloak
    │
    │ Docker Network
    │
    ▼
keycloak-mysql
```

---

# 20. Verify Keycloak Port

Check whether port `8083` is listening:

```bash
ss -lntp | grep 8083
```

You can also test locally:

```bash
curl -I http://localhost:8083
```

---

# 21. Access Keycloak

For local/internal testing:

```text
http://SERVER-IP:8083
```

For the configured public URL:

```text
https://lms.svkm.ac.in/kck
```

The public URL should be accessed through the configured reverse proxy.

---

# 22. Container Management

## Start

```bash
docker compose up -d
```

## Stop

```bash
docker compose stop
```

## Restart

```bash
docker compose restart
```

## Restart Only Keycloak

```bash
docker compose restart keycloak
```

## Restart Only MySQL

```bash
docker compose restart mysql
```

## Stop and Remove Containers

```bash
docker compose down
```

> `docker compose down` does not remove the named `mysql_data` volume unless `-v` is explicitly used.

---

# 23. Important Docker Volume Warning

Do **not** run:

```bash
docker compose down -v
```

unless you intentionally want to remove the MySQL volume.

The `-v` option removes the named volume:

```text
mysql_data
```

which contains the Keycloak database.

Removing it can result in complete loss of the Keycloak database.

---

# 24. Updating the Custom Provider

Replace the provider:

```bash
cp new-provider.jar provider/keycloak-lms-provider.jar
```

Restart Keycloak:

```bash
docker compose restart keycloak
```

Verify:

```bash
docker exec keycloak ls -lh \
  /opt/jboss/keycloak/standalone/deployments/
```

Check logs:

```bash
docker logs -f keycloak
```

---

# 25. Updating the Theme

Update the required theme files:

```text
login.ftl
info.ftl
error.ftl
saml-post-form.ftl
messages_en.properties
nmims-logo.png
```

Restart Keycloak:

```bash
docker compose restart keycloak
```

If the changes are not visible, perform a browser hard refresh or clear browser cache.

---

# 26. MySQL Backup

Since MySQL contains the Keycloak persistent data, regular database backups are required.

First access the MySQL container:

```bash
docker exec -it keycloak-mysql bash
```

A database dump can be created using `mysqldump`.

Example from the Docker host:

```bash
docker exec keycloak-mysql \
  mysqldump -u root -prootpass keycloak > keycloak_backup.sql
```

This creates:

```text
keycloak_backup.sql
```

in the current directory on the Docker host.

---

# 27. MySQL Restore

Restore the database using:

```bash
cat keycloak_backup.sql | \
docker exec -i keycloak-mysql \
mysql -u root -prootpass keycloak
```

For production, always validate backups by performing periodic restore tests.

---

# 28. Troubleshooting

## Keycloak Container Is Not Starting

Check:

```bash
docker compose ps
```

Then:

```bash
docker logs keycloak
```

Look for:

* Database connection errors
* Provider deployment errors
* Invalid configuration
* JVM memory errors
* Port conflicts

---

## MySQL Container Is Not Starting

Check:

```bash
docker logs keycloak-mysql
```

Check:

```bash
docker compose ps
```

Verify the MySQL volume:

```bash
docker volume ls
```

---

## Keycloak Cannot Connect to MySQL

Verify that Keycloak uses:

```yaml
DB_ADDR: mysql
```

Do not use:

```yaml
DB_ADDR: localhost
```

inside the Keycloak container.

`localhost` refers to the Keycloak container itself.

The Docker Compose service name:

```text
mysql
```

is used for container-to-container communication.

---

## Provider Not Loading

Check:

```bash
docker exec keycloak ls -lh \
  /opt/jboss/keycloak/standalone/deployments/
```

Then:

```bash
docker logs keycloak
```

Look for deployment or JAR compatibility errors.

---

## Theme Not Loading

Check:

```bash
docker exec keycloak ls -lh \
  /opt/jboss/keycloak/themes/nmims/login/
```

Check the logo:

```bash
docker exec keycloak ls -lh \
  /opt/jboss/keycloak/themes/nmims/login/resources/img/
```

Restart:

```bash
docker compose restart keycloak
```

---

## Port 8083 Already in Use

Check:

```bash
sudo ss -lntp | grep 8083
```

or:

```bash
sudo lsof -i :8083
```

If required, change the host port:

```yaml
ports:
  - "8084:8080"
```

---

# 29. Security Recommendations

The example Compose file contains passwords directly in the environment section.

For production environments, credentials should not be committed to Git.

Instead, use environment variables or a secrets-management solution.

For example:

```yaml
environment:
  MYSQL_PASSWORD: ${MYSQL_PASSWORD}
  MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}

  DB_PASSWORD: ${MYSQL_PASSWORD}

  KEYCLOAK_PASSWORD: ${KEYCLOAK_PASSWORD}
```

Recommended production practices:

* Do not use MySQL `root` for Keycloak.
* Create a dedicated MySQL user.
* Do not commit passwords to Git.
* Use Docker secrets or an external secret manager where appropriate.
* Restrict access to MySQL.
* Do not expose MySQL port `3306` publicly.
* Use HTTPS for external Keycloak access.
* Restrict Keycloak administration access.
* Regularly back up the MySQL database.
* Test database restoration.
* Pin application versions.
* Test custom provider compatibility before upgrades.

---

# 30. Production Credential Example

A safer Compose configuration can use environment variables:

```yaml
environment:
  MYSQL_DATABASE: keycloak
  MYSQL_USER: kcuser
  MYSQL_PASSWORD: ${MYSQL_PASSWORD}
  MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
```

Keycloak:

```yaml
environment:
  DB_VENDOR: mysql
  DB_ADDR: mysql
  DB_DATABASE: keycloak
  DB_USER: kcuser
  DB_PASSWORD: ${MYSQL_PASSWORD}

  KEYCLOAK_USER: ${KEYCLOAK_USER}
  KEYCLOAK_PASSWORD: ${KEYCLOAK_PASSWORD}
```

Example `.env`:

```text
MYSQL_PASSWORD=<database-password>
MYSQL_ROOT_PASSWORD=<root-password>
KEYCLOAK_USER=kckp-admin
KEYCLOAK_PASSWORD=<keycloak-admin-password>
```

Add `.env` to `.gitignore`:

```text
.env
```

Never commit production secrets to the repository.

---

# 31. Logging

MySQL uses Docker's JSON logging driver:

```yaml
logging:
  driver: "json-file"
  options:
    max-size: "100m"
    max-file: "7"
```

This limits individual log files to:

```text
100 MB
```

with up to:

```text
7 files
```

This helps prevent unlimited Docker log growth.

---

# 32. Monitoring

Useful commands for basic monitoring:

### Container Status

```bash
docker compose ps
```

### Resource Usage

```bash
docker stats
```

### Keycloak Resource Usage

```bash
docker stats keycloak
```

### MySQL Resource Usage

```bash
docker stats keycloak-mysql
```

### Disk Usage

```bash
docker system df
```

---

# 33. Upgrade Considerations

The deployment is currently pinned to:

```text
Keycloak 16.1.0
```

Do not upgrade Keycloak simply by changing:

```yaml
image: quay.io/keycloak/keycloak:16.1.0
```

to another version in production.

Before upgrading, test:

* Custom provider compatibility
* Custom theme compatibility
* Database migration
* Existing realms
* Clients
* Authentication flows
* SAML integrations
* Reverse proxy configuration
* Frontend URL behavior
* JVM configuration

Maintain a database backup before performing any Keycloak upgrade.

---

# 34. Deployment Checklist

## Infrastructure

* [ ] Docker installed
* [ ] Docker Compose available
* [ ] Sufficient server resources
* [ ] DNS configured
* [ ] Reverse proxy configured
* [ ] HTTPS configured

## MySQL

* [ ] MySQL image available
* [ ] MySQL container running
* [ ] Database created
* [ ] MySQL persistent volume created
* [ ] Database credentials configured
* [ ] Database backup configured

## Keycloak

* [ ] Keycloak image available
* [ ] Keycloak container running
* [ ] Database connection verified
* [ ] Admin credentials configured
* [ ] JVM settings verified
* [ ] Token Exchange enabled
* [ ] Fine-grained Admin Authorization enabled

## Custom Components

* [ ] Provider JAR deployed
* [ ] Provider deployment verified
* [ ] Custom theme deployed
* [ ] Login page verified
* [ ] NMIMS logo verified
* [ ] SAML page verified

## Validation

* [ ] `docker compose config` successful
* [ ] `docker compose ps` verified
* [ ] Keycloak logs reviewed
* [ ] MySQL logs reviewed
* [ ] Port `8083` accessible
* [ ] Public URL accessible
* [ ] Login tested
* [ ] Database backup tested

---

# 35. Quick Deployment Reference

```bash
# Navigate to project
cd /data/keycloak-deployment

# Validate Compose configuration
docker compose config

# Pull images
docker compose pull

# Start the complete stack
docker compose up -d

# Check status
docker compose ps

# Check Keycloak logs
docker logs -f keycloak

# Check MySQL logs
docker logs -f keycloak-mysql

# Restart Keycloak
docker compose restart keycloak

# Restart MySQL
docker compose restart mysql

# Stop services
docker compose stop

# Remove containers
docker compose down

# Check resource usage
docker stats
```

---

# 36. Deployment Summary

| Component          | Configuration                      |
| ------------------ | ---------------------------------- |
| Application        | Keycloak                           |
| Keycloak Version   | `16.1.0`                           |
| Keycloak Image     | `quay.io/keycloak/keycloak:16.1.0` |
| Keycloak Container | `keycloak`                         |
| Keycloak Port      | `8080`                             |
| Host Port          | `8083`                             |
| Database           | MySQL                              |
| MySQL Version      | `8.0`                              |
| MySQL Container    | `keycloak-mysql`                   |
| Database Name      | `keycloak`                         |
| Database User      | `kcuser`                           |
| MySQL Persistence  | `mysql_data`                       |
| Custom Provider    | `keycloak-lms-provider.jar`        |
| Custom Theme       | `nmims`                            |
| Initial JVM Heap   | `512 MB`                           |
| Maximum JVM Heap   | `1024 MB`                          |
| Public URL         | `https://lms.svkm.ac.in/kck`       |
| Deployment Method  | Docker Compose                     |

---

# 37. Deployment Flow

```text
                 Git Repository
                       │
                       │
                       ▼
              Deployment Server
                       │
                       │
                docker compose
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
       MySQL Container      Keycloak Container
             │                   │
             │                   ├── Custom Provider
             │                   │
             │                   └── Custom Theme
             │
             ▼
        mysql_data
          Volume
             │
             │
             └──────────────┐
                            │
                            ▼
                    Persistent Data


Client
  │
  │ HTTPS
  ▼
Reverse Proxy
  │
  │ HTTP :8083
  ▼
Keycloak :8080
```

---

# 38. Important Note

This deployment uses the legacy Keycloak **16.1.0** distribution, which is based on the WildFly/JBoss architecture.

The custom provider and theme are therefore deployed using the corresponding Keycloak 16.x directory structure.

Any migration to a newer Keycloak architecture should be planned and tested separately.

---

## Maintainer

**Keycloak Deployment Project**

This repository contains the Docker Compose configuration, custom provider, custom theme, and deployment documentation required to run the Keycloak environment.
