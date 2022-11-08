---
title: "Envoy Egress aka Forward Proxy"
keywords: ["network", "egress"]
tags: ["compose", "docker", "egress", "envoy", "linux", "network", "proxy"]
date: 2022-11-08T14:43:03+01:00
---

The previous article introduced traefik as ingress proxy (aka reverse proxy). Looking at the traffic
landscape the ingress and egress proxies are responsible for north-to-south-traffic. Traffic enters
the cluster from the north and sends request to the ingress proxy. In this article we highlight how
to control traffic leaving the cluster via an egress proxy.

```


                               +----------------+                          | N 
                               |    ingress     |                          |  
                               +----------------+                          |  
                                       |                                   |  
                                       v                                   |  
        +----------------+     +----------------+     +----------------+   |  
        |                |     |                |     |                |   |  
        |   component x  |---->|   component a  |---->|   component b  |   |  
        |                |     |                |     |                |   |  
        +----------------+     +----------------+     +----------------+   |  
                                       |                                   |  
                                       v                                   |  
                               +----------------+                          |  
                               |    egress      |                          |  
                               +----------------+                          v S 
        --------------------------------------------------------------->      
        
        E                                                              W
        
                                                                   
```

> Brief overview how traffic flows through the cluster.

# Enter envoy

Envoy was envisioned as modern L7 proxy. However, it can proxy L3/L4 as well. As such it can be
used as dynamic forward proxy for HTTP, HTTPS and TCP with SNI.

The following configuration enables HTTP(S) forwarding on port 10000 and TCP forwarding on
port 10001.
```yaml
admin:
  address:
    socket_address:
      protocol: TCP
      address: 127.0.0.1
      port_value: 9901
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          protocol: TCP
          address: 0.0.0.0
          port_value: 10000
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
#                access_log:
#                  - name: envoy.access_loggers.file
#                    typed_config:
#                      "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
#                      path: /dev/stdout
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            cluster: dynamic_forward_proxy_cluster
                        - match:
                            connect_matcher: {}
                          route:
                            cluster: dynamic_forward_proxy_cluster
                            upgrade_configs:
                              - upgrade_type: CONNECT
                                enabled: true
                http_filters:
                  - name: envoy.filters.http.dynamic_forward_proxy
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.dynamic_forward_proxy.v3.FilterConfig
                      dns_cache_config:
                        name: dynamic_forward_proxy_cache_config
                        dns_lookup_family: V4_ONLY
                        typed_dns_resolver_config:
                          name: envoy.network.dns_resolver.cares
                          typed_config:
                            "@type": type.googleapis.com/envoy.extensions.network.dns_resolver.cares.v3.CaresDnsResolverConfig
                            resolvers:
                              - socket_address:
                                  address: "127.0.0.11"
                                  port_value: 53
                            dns_resolver_options:
                              use_tcp_for_dns_lookups: true
                              no_default_search_domain: true
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
    - name: listener_1
      address:
        socket_address:
          protocol: TCP
          address: 0.0.0.0
          port_value: 10001
      listener_filters:
        - name: envoy.filters.listener.tls_inspector
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.listener.tls_inspector.v3.TlsInspector
      filter_chains:
        - filters:
            - name: envoy.filters.network.sni_dynamic_forward_proxy
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.sni_dynamic_forward_proxy.v3.FilterConfig
                port_value: 443
                dns_cache_config:
                  name: dynamic_forward_proxy_cache_config
                  dns_lookup_family: V4_ONLY
                  typed_dns_resolver_config:
                    name: envoy.network.dns_resolver.cares
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.network.dns_resolver.cares.v3.CaresDnsResolverConfig
                      resolvers:
                        - socket_address:
                            address: "127.0.0.11"
                            port_value: 53
                      dns_resolver_options:
                        use_tcp_for_dns_lookups: true
                        no_default_search_domain: true
            - name: envoy.tcp_proxy
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
                stat_prefix: tcp
                cluster: dynamic_forward_proxy_cluster
  clusters:
    - name: dynamic_forward_proxy_cluster
      lb_policy: CLUSTER_PROVIDED
      cluster_type:
        name: envoy.clusters.dynamic_forward_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.clusters.dynamic_forward_proxy.v3.ClusterConfig
          dns_cache_config:
            name: dynamic_forward_proxy_cache_config
            dns_lookup_family: V4_ONLY
            typed_dns_resolver_config:
              name: envoy.network.dns_resolver.cares
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.network.dns_resolver.cares.v3.CaresDnsResolverConfig
                resolvers:
                  - socket_address:
                      address: "127.0.0.11"
                      port_value: 53
                dns_resolver_options:
                  use_tcp_for_dns_lookups: true
                  no_default_search_domain: true
```

# http_proxy

Applications that honor the proxy environment variables are can be instructed to use the proxy just
by setting the following environment variables:

```shell
http_proxy=http://envoy:10000
https_proxy=http://envoy:10000
no_proxy=127.0.0.1,localhost
```
> Mind the lower case.

# nftables redirection

Some applications need an additional push to use the forward proxy. For those a redirection with 
nftables forces traffic through the proxy.

```shell
nft add rule ip nat OUTPUT tcp dport 443 dnat to envoy:10001
```

Containers that modify the network must have the capability `NET_ADMIN`. In combination with docker
compose a small container with only `nftables` and the required permissions can redirect the
traffic.

Create a container image with nftables and use it with compose.

```dockerfile
FROM alpine
RUN apk add --no-cache nftables
```

Build a one-shot service that only modifies the network namespace.

```yaml
services:
  application-nftables:
    image: nftables
    build: ./nftables
    depends_on:
      - envoy
    command: "nft add rule ip nat OUTPUT tcp dport 443 dnat to envoy:10001"
    cap_add:
      - NET_ADMIN
    network_mode: service:application # Reuse an existing network namespace.
``` 

# Network isolation

Network isolation enables to allow traffic only through the egress proxy. Network isolation with
docker is discussed in a [previous post]({{< ref "/posts/docker-network-isolation.md" >}}).

Add a network without access to outside networks and one with only access to outside.

```yaml
networks:
  proxy_egress:
    driver_opts:
      com.docker.network.bridge.enable_ip_masquerade: "false"
  egress:
    driver_opts:
      com.docker.network.bridge.enable_icc: "false"
```

Attach envoy to both networks and the application that should use the proxy to only the internal
network.

```yaml
services:
  envoy:
    image: envoyproxy/envoy-distroless
    networks:
      - egress
      - proxy_egress
  
  application:
    image: application
    environment:
      http_proxy: http://envoy:10000
      https_proxy: http://envoy:10000
    networks:
      - proxy_egress
```

With separate networks and an egress proxy a further step towards k8s with proper service mesh (e.g.
istio) is done.
Next ahead ist monitoring traffic and resources.

# Road to k8s pt3

Do you like the guide and want to give feedback or found a mistake? Then send me a mail
to `f4ntasyland /at/ protonmail /dot/ com`

You can always buy me a beer.
`（ ^_^）o自自o（^_^ ）`

_[xmr][0]:
473WTZ1gWFdjdEyioCQfbGQKRurZQoPDJLhuFJyubCrk4TRogKCtRum63bFMx2dP2y4AN1vf2fN6La7V7eB2cZ4vNJgMAcG_

[0]: https://www.getmonero.org/
