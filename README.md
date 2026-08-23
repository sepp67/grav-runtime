# grav-runtime

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-GHCR-blue)](https://ghcr.io/sepp67/grav-runtime)

**Reusable Docker runtime for building and running Grav CMS applications with Nginx and PHP-FPM.**

`grav-runtime` provides the shared technical foundation for multiple Grav websites.
Application repositories inherit from this image and add only their project-specific
themes, plugins, configuration and initial content.

## Where does it fit?

```text
grav-runtime                 ← YOU ARE HERE
      │
      ▼
application image
(projet-gites,
 projet-lavallee, ...)
      │
      ▼
ansible-role-grav-site
      │
      ▼
persistent Grav instance
```

The runtime, application and deployment mechanism are deliberately separated so that
each component can evolve and be versioned independently.

## Responsibilities

### What it does

- Provides Grav Core and the Admin plugin.
- Provides the PHP runtime and Nginx web server.
- Exposes HTTP on port `80`.
- Provides a native `/healthz` health endpoint.
- Defines the persistence contract for Grav application images.
- Seeds persistent directories on first startup.
- Supports optional administrator bootstrap through environment variables.
- Provides the common runtime contract consumed by application images and the deployment role.

### What it does not do

- Does not contain website-specific themes, plugins or content.
- Does not contain production secrets.
- Does not deploy applications to servers.
- Does not manage DNS, TLS or reverse proxies.
- Does not manage host infrastructure.

Those responsibilities belong to the application repositories, `ansible-role-grav-site`,
or the surrounding infrastructure.

## Quick Start

Build the runtime locally:

```bash
docker build -t grav-runtime:local .
```

Start the test environment:

```bash
docker compose -f test/compose.yml up -d --build
```

Check the runtime:

```bash
curl http://localhost:8081/healthz
```

A Grav application image consumes the runtime through a versioned base image:

```dockerfile
FROM ghcr.io/sepp67/grav-runtime:<version>
```

The application image then adds only its project-specific layer.

## Tested & Supported

| Component | Support |
|---|---|
| Grav CMS | Version packaged and certified by the runtime release |
| PHP | 8.3 |
| Web server | Nginx |
| Application server | PHP-FPM |
| Container runtime | Docker |
| HTTP | Port 80 |
| Health check | `GET /healthz` |

Run the runtime test environment with:

```bash
docker compose -f test/compose.yml up -d --build
```

Runtime releases are versioned explicitly. Application images should always reference
a specific runtime version rather than an implicit moving version.

## Documentation & Related Components

Full documentation:

**https://docs.lavallee.tech/grav-stack/runtime/**

Related repositories:

- [`projet-gites`](https://github.com/sepp67/projet-gites) — Grav application image for the Gîtes website.
- [`projet-lavallee-website`](https://github.com/sepp67/projet-lavallee-website) — Grav application image for lavallee.tech.
- [`ansible-role-grav-site`](https://github.com/sepp67/ansible-role-grav-site) — reusable deployment role for compatible Grav applications.

## License

MIT