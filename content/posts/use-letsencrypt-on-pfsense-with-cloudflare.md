---
title: "Use let's encrypt on pfsense with cloudflare"
date: 2021-01-04T20:33:18+01:00
slug: "make-hugo-blog"
description: "About making a hugo blog site"
keywords: ["cloudflare", "letsencrypt", "pfsense"]
draft: false
tags: ["cloudflare", "letsencrypt", "pfsense"]
math: false
toc: false
---

[Let's encrypt][0] provides the masses with free certificates for servers. It's shown how to use
these certificates on a [pfSense][1] with [cloudflare][2] dns.
The requirements are a working firewall with pfsense and a cloudflare account.

## Cloudflare token

Go to `My Profile` > `API Tokens` and click on `Create Token`. Use the `Edit zone DNS` template and
configure the zone and optionally the ip address filtering according to your needs.
Copy the generated token.

## Pfsense

### Package

Install the `acme` package first. Go to `Services` > `ACME Certificates`.

### Account

Add an account key and ensure that the staging server is selected as long as the configuration
isn't finished.
Click on `Create account key` and then on `Register account key`.
Save.

### Certificate

Create a new certificate where the created account is selected. In the `Domain SAN list` section add
a new domain. Make sure to select `DNS-Cloudflare` in the `Method` column. Then add the desired
domain and paste the token into the `Token` field.
Save and hit `Issue/Renew`. This will take some time (about 1-2min).

The certificate is automatically re-issued every 60 days, but the web server only uses the new cert
if it is restarted. To do so follow the short description in section `Actions list`.

If everything works as expected change the server from staging to production in your account.

## Use

In order to use the certificate it must be set as web ui certificate under `Advanced` >
`Admin Access` > `SSL/TLS Certificate`.
Done.

[0]: https://letsencrypt.org
[1]: https://www.pfsense.org
[2]: https://www.cloudflare.com
