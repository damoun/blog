---
layou: post
title: Configure Let's Encrypt certificate on your TrueNAS server
categories: [ops]
tags: [truenas, install, server, nas, freenas, lets encrypt, acme]
---

I'm still rocking a HP N54L at home as my NAS using [TrueNAS Core](https://www.truenas.com/truenas-core/) 13.
The only thing that was bothering me is my un-encrypted web UI to manage it. Since Let's Encrypt is widely use, I was looking for a solution with Let's Encrypt certificate on my TrueNAS and so, I found the solution:
Using [acme.sh](https://github.com/acmesh-official/acme.sh) !

# Requirements

This post require a TrueNAS Core server, a DNS zone already on Cloudflare and a sub-domain name for your nas.

# Install acme.sh

Let's open a shell on your TrueNAS server by using the web UI or by ssh with the root user.
Even if it's a bad practice the pipe curl into a shell, the simplest way to install asme.sh on your TrueNas is by using the following command:

```shell
curl https://get.acme.sh | sh -s email=my@example.com
```

# Create your first certificate

My NAS is not expose to internet so my only one solution is to use a verification by DNS.
Since I use Cloudflare to host my zone, I have to create an API Token for acme.sh on my account: [dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens).

This token must have the following permissions on your zone:
- DNS:Read
- DNS:Edit

As soon as your token is created, you can run the following command to generate your certificate. You can [find your Account & Zone ID](https://developers.cloudflare.com/fundamentals/get-started/basic-tasks/find-account-and-zone-ids/) on your zone overview on the left column.

```shell
setenv CF_Account_ID <Cloudflare Account ID>
setenv CF_Zone_ID <Cloudflare Zone ID>
setenv CF_Token <Cloudflare API Token>
/root/.acme.sh/acme.sh --issue -d <nas sub-domain> --dns dns_cf
```

acme.sh will save your token and IDs on the disk for renewal purpose.

# Deploy your certificate on TrueNAS

The last step is to deploy your certificate to TrueNAS using its API so let's [create an API Key](https://www.truenas.com/docs/scale/scaletutorials/toptoolbar/managingapikeys/#adding-an-api-key) from your web UI. Then deploy your certificate:

```shell
setenv DEPLOY_TRUENAS_APIKEY <TrueNAS API key>
/root/.acme.sh/acme.sh --insecure --deploy -d <nas sub-domain> --deploy-hook truenas
```
