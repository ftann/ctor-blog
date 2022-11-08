---
title: "From swag to traefik as reverse proxy"
keywords: ["network", "ingress"]
tags: ["compose", "docker", "ingress", "linux", "network", "proxy", "traefik"]
date: 2022-10-20T13:17:19+02:00
---

The current setup uses a reverse proxy as ingress to other web services. The proxy also has to serve
a valid certificate. [Swag][1] provided those components and was easy to integrate into the existing
landscape.

# Configuration

Originally swag required to you to configure the nginx proxy by modifying files. Although swag
nowadays gained support for automatic proxying through docker-mods. Docker-mods are a mechanism from
linuxserver.io to extend existing containers with additional functionality.

As longtime user of docker-compose I like the declarative approach to define infrastructure.
[Traefik][2] combined with command-line arguments and container labels enables a declarative way of
defining what to proxy where. The auto-magic is controlled by labels rather than downloaded
image-layers with shell scripts (**great win**).

A certain drawback for beginners is that swag provided configurations out-of-the-box. With traefik
those configuration have to be provided by ourselves. However, traefik is stateless and it's not 
required to store/backup the configuration files in a volume (compare compile-time vs runtime).
Additionally, traefik provides metrics that can be integrated into prometheus.

# Example

## Traefik

An example configuration could look like the following snippet.

```yaml
traefik:
  image: traefik
  command:
    - "--providers.docker=true"
    - "--providers.docker.exposedByDefault=false"
    - "--providers.docker.network=proxy"
    - "--providers.file.directory=/config"
    - "--providers.file.watch=true"
    - "--entryPoints.http=true"
    - "--entryPoints.http.address=:8080"
    - "--entryPoints.http.forwardedHeaders.insecure=false"
    - "--entryPoints.http.proxyProtocol.insecure=true"
    - "--entryPoints.https=true"
    - "--entryPoints.https.address=:8443"
    - "--entryPoints.https.http.tls=true"
    - "--entryPoints.https.http.tls.certResolver=le"
    - "--entryPoints.https.http.tls.domains[0].main=${DOMAIN}"
    - "--entryPoints.https.http.tls.domains[0].sans=*.${DOMAIN}"
    - "--entryPoints.https.forwardedHeaders.insecure=false"
    - "--entryPoints.https.proxyProtocol.insecure=true"
    - "--certificatesResolvers.le.acme.dnsChallenge.provider=cloudflare"
    - "--certificatesResolvers.le.acme.storage=/config/acme.json"
    - "--serversTransport.insecureSkipVerify=true"
    - "--metrics.prometheus=true"
    - "--metrics.prometheus.entrypoint=http"
  networks:
    - ingress
    - proxy
  ports:
    - "80:8080"
    - "443:8443"
  environment:
    CF_DNS_API_TOKEN_FILE: /run/secrets/cloudflare
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - traefik:/config:z
  secrets:
    - cloudflare
  labels:
    - "traefik.enable=true"
    # Https only.
    - "traefik.http.middlewares.httpsOnly.redirectScheme.scheme=https"
    - "traefik.http.middlewares.httpsOnly.redirectScheme.permanent=true"
    # Rate limit.
    - "traefik.http.middlewares.rateLimit.rateLimit.average=100"
    - "traefik.http.middlewares.rateLimit.rateLimit.burst=50"
    # Secure headers.
    - "traefik.http.middlewares.secureHeaders.headers.accessControlAllowMethods=GET,OPTIONS,PUT"
    - "traefik.http.middlewares.secureHeaders.headers.accessControlAllowOriginList=https://${DOMAIN}"
    - "traefik.http.middlewares.secureHeaders.headers.accessControlMaxAge=100"
    - "traefik.http.middlewares.secureHeaders.headers.addVaryHeader=true"
    - "traefik.http.middlewares.secureHeaders.headers.hostsProxyHeaders=X-Forwarded-Host"
    - "traefik.http.middlewares.secureHeaders.headers.sslProxyHeaders.X-Forwarded-Proto=https"
    - "traefik.http.middlewares.secureHeaders.headers.stsSeconds=63072000"
    - "traefik.http.middlewares.secureHeaders.headers.stsIncludeSubdomains=true"
    - "traefik.http.middlewares.secureHeaders.headers.stsPreload=true"
    - "traefik.http.middlewares.secureHeaders.headers.forceSTSHeader=true"
    - "traefik.http.middlewares.secureHeaders.headers.customFrameOptionsValue=SAMEORIGIN"
    - "traefik.http.middlewares.secureHeaders.headers.contentTypeNosniff=true"
    - "traefik.http.middlewares.secureHeaders.headers.browserXssFilter=true"
    - "traefik.http.middlewares.secureHeaders.headers.referrerPolicy=same-origin"
    - "traefik.http.middlewares.secureHeaders.headers.customResponseHeaders.X-Robots-Tag=none"
    # Chains.
    - "traefik.http.middlewares.secured.chain.middlewares=rateLimit,httpsOnly,secureHeaders"
```

## Web service

Plex is here the example service that should be proxied. Middlewares can be reused in other services
and merged into chains.

```yaml
plex:
  image: linuxserver/plex
  networks:
    - egress
    - proxy
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.plex.rule=Host(`plex.${DOMAIN}`)"
    - "traefik.http.routers.plex.entryPoints=https"
    - "traefik.http.routers.plex.middlewares=secured"
    - "traefik.http.services.plex.loadBalancer.server.port=32400"
```

And with those labels plex is reachable through traefik with `plex.your-domain.com`.

# Road to k8s pt2

Do you like the guide and want to give feedback or found a mistake? Then send me a mail
to `f4ntasyland /at/ protonmail /dot/ com`

You can always buy me a beer.
`（ ^_^）o自自o（^_^ ）`

_[xmr][0]:
473WTZ1gWFdjdEyioCQfbGQKRurZQoPDJLhuFJyubCrk4TRogKCtRum63bFMx2dP2y4AN1vf2fN6La7V7eB2cZ4vNJgMAcG_

[0]: https://www.getmonero.org/

[1]: https://github.com/linuxserver/docker-swag

[2]: https://github.com/traefik/traefik
