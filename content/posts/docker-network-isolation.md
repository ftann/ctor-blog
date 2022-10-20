---
title: "Docker network isolation"
keywords: ["docker", "network"]
tags: ["docker", "linux", "network"]
date: 2022-10-20T12:30:24+02:00
---

Docker compose provides a simple method to define separate networks for certain aspects (e.g.
communication with a database). By default these networks are connected to the default bridge. This
enables containers to send traffic to the networks outside the host. However, a database should be
disallowed to do so.

The opposite case is also possible where accessing external networks is desirable but attached
containers mustn't communicate with each other. Egress traffic is a contender for such networks. 

*This only works with the bridge network driver.*

# Internal networks

To prevent a container from sending packages to outside networks add the
`com.docker.network.bridge.enable_ip_masquerade` driver option. Containers attached to this network
still automatically get an ip and are reachable by their respective names.

```yaml
networks:
  database:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_ip_masquerade: "false"
```

# Egress networks

Networks can be configured to allow traffic to outside networks but disallow inter container
communication. To do so add the `com.docker.network.bridge.enable_icc` driver option. 

```yaml
networks:
  egress:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_icc: "false"
```

# Road to k8s

Do you like the guide and want to give feedback or found a mistake? Then send me a mail
to `f4ntasyland /at/ protonmail /dot/ com`

You can always buy me a beer.
`（ ^_^）o自自o（^_^ ）`

_[xmr][0]:
473WTZ1gWFdjdEyioCQfbGQKRurZQoPDJLhuFJyubCrk4TRogKCtRum63bFMx2dP2y4AN1vf2fN6La7V7eB2cZ4vNJgMAcG_

[0]: https://www.getmonero.org/
