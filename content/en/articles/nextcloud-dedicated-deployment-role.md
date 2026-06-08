---

title: "Why Nextcloud Requires a Dedicated Deployment Role Instead of a Generic Docker Application Role"
date: 2026-06-08
author: "Sébastien Lavallée"
description: "Lessons learned while deploying Nextcloud in a staging environment managed with Ansible and Docker."
---

As part of my infrastructure redesign, I recently created a dedicated staging environment separate from production. The objective was straightforward: validate new application versions in an isolated environment before deploying them to production.

Most of my applications follow the same deployment pattern:

* Pull a container image from GitHub Container Registry.
* Expose a single HTTP port.
* Configure a reverse proxy.
* Validate the application.

This approach works perfectly for applications such as:

* Grav CMS
* Webcam Stream
* Invoice Demo
* Static websites

To support this model, I created a generic Ansible role called `staging_docker_app`.

The role simply performs the following tasks:

1. Pull the container image.
2. Create the container.
3. Publish the required port.
4. Start the application.

For simple services, this pattern is efficient, reusable, and easy to maintain.

Unfortunately, Nextcloud quickly demonstrated the limitations of this approach.

## The Initial Assumption

At first, I assumed that Nextcloud could be deployed exactly like my other applications.

My staging configuration looked like this:

```yaml
app_name: nextcloud
app_image: ghcr.io/sepp67/ansible-role-nextcloud-stack:latest
app_container_name: nextcloud-staging
app_host_port: 18080
app_container_port: 80
```

The deployment succeeded.

The container started.

The reverse proxy worked.

However, the application itself was not functional.

## Understanding the Problem

Unlike a simple web application, Nextcloud is not a single service.

My production deployment consists of four containers:

```text
Nextcloud Stack
├── Nextcloud
├── PostgreSQL
├── Redis
└── Cron
```

The compose file illustrates the architecture clearly:

```yaml
services:
  db:
    image: postgres

  redis:
    image: redis

  app:
    image: nextcloud

  cron:
    image: nextcloud
```

The application container depends on:

* PostgreSQL for persistent storage.
* Redis for caching and file locking.
* Cron jobs for background processing.

Launching only the Nextcloud container inevitably results in an incomplete deployment.

## The Architecture Mismatch

My generic deployment role was designed around the assumption:

```text
One application = One container
```

Nextcloud follows a different model:

```text
One application = Multiple cooperating services
```

This is a completely different deployment pattern.

Trying to force both models into the same Ansible role quickly leads to:

* Conditional logic everywhere.
* Special cases.
* Reduced maintainability.
* Poor readability.

Eventually the generic role stops being generic.

## The Better Solution

Instead of complicating the existing role, I created a dedicated role:

```text
staging_nextcloud_stack
```

The responsibilities became much clearer.

### Generic Role

```text
staging_docker_app
```

Used for:

* Grav
* Webcam Stream
* Invoice Demo
* Static websites

Responsibilities:

* Pull image.
* Create container.
* Publish port.
* Start service.

### Dedicated Nextcloud Role

```text
staging_nextcloud_stack
```

Responsibilities:

* Create persistent directories.
* Generate environment variables.
* Generate Docker Compose configuration.
* Deploy PostgreSQL.
* Deploy Redis.
* Deploy Nextcloud.
* Deploy Cron.
* Start the stack.

This separation keeps both roles simple and maintainable.

## Reusing Existing Infrastructure Components

Another lesson learned was avoiding duplicate responsibilities.

My staging environment already contained a role called:

```text
docker_host
```

This role installs:

* Docker CE
* Docker Compose Plugin
* Buildx
* Containerd

Initially I attempted to install Docker directly from the Nextcloud role.

This quickly created overlap and unnecessary complexity.

The final architecture became:

```text
docker_host
        │
        ▼
staging_nextcloud_stack
```

The Docker installation is managed once.

Application-specific deployment logic remains inside the application role.

## Final Architecture

The staging environment now looks like this:

```text
vm-proxy-staging
│
├── lavallee.staging.local
├── facturier.staging.local
├── grav.staging.local
├── webcam.staging.local
└── nextcloud.staging.local

Application VMs
│
├── vm-lavallee-staging
├── vm-facturier-staging
├── vm-grav-staging
├── vm-camera-staging
└── vm-nextcloud-staging
```

The Nextcloud VM itself contains:

```text
vm-nextcloud-staging
│
├── PostgreSQL
├── Redis
├── Nextcloud
└── Cron
```

managed through Docker Compose and deployed automatically with Ansible.

## Conclusion

The goal of infrastructure automation is not to make every deployment identical.

The goal is to make every deployment predictable, maintainable, and easy to understand.

For simple applications, a generic container deployment role is often sufficient.

For applications composed of multiple services, such as Nextcloud, a dedicated deployment role provides a cleaner architecture and a more maintainable codebase.

The key lesson is simple:

> Generic roles should stay generic. Complex applications deserve dedicated deployment logic.
