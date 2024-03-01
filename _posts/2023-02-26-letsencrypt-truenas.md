---
layou: post
title: Secure Your TrueNAS Server with Let's Encrypt Certificate
categories: [ops]
tags: [truenas, install, server, nas, freenas, lets encrypt, acme]
---

At home, my trusty HP N54L serves as my NAS, powered by [TrueNAS Core](https://www.truenas.com/truenas-core/) 13. However, one lingering concern was the lack of encryption for its web UI. Eager to enhance security, I sought a solution and discovered the effectiveness of Let's Encrypt certificates, specifically through the use of [acme.sh](https://github.com/acmesh-official/acme.sh).

## Requirements

Before diving into the configuration process, ensure you have a TrueNAS Core server, an existing DNS zone on Cloudflare, and a designated sub-domain for your NAS.

## Installing acme.sh

To begin, access the shell on your TrueNAS server either through the web UI or SSH with the root user. While it's generally discouraged to pipe `curl` into a shell, the most straightforward method to install acme.sh on your TrueNAS is to use the following command:

```shell
curl https://get.acme.sh | sh -s email=my@example.com
```

## Creating Your First Certificate

As my NAS is not exposed to the internet, I opted for DNS verification. For Cloudflare users, an API Token needs to be created on your account: [dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens). The token must have the following permissions on your zone: DNS:Read and DNS:Edit.

Once the token is generated, use the following command to create your certificate. Retrieve your Account & Zone ID from your [Cloudflare zone overview](https://developers.cloudflare.com/fundamentals/get-started/basic-tasks/find-account-and-zone-ids/):

```shell
setenv CF_Account_ID <Cloudflare Account ID>
setenv CF_Zone_ID <Cloudflare Zone ID>
setenv CF_Token <Cloudflare API Token>
/root/.acme.sh/acme.sh --issue -d <nas sub-domain> --dns dns_cf
```

acme.sh will store your token and IDs on disk for future renewals.

## Deploying Your Certificate on TrueNAS

The final step involves deploying your certificate to TrueNAS using its API. Start by [creating an API Key](https://www.truenas.com/docs/scale/scaletutorials/toptoolbar/managingapikeys/#adding-an-api-key) from your web UI. Subsequently, deploy your certificate with the following commands:

```shell
setenv DEPLOY_TRUENAS_APIKEY <TrueNAS API key>
/root/.acme.sh/acme.sh --insecure --deploy -d <nas sub-domain> --deploy-hook truenas
```

By following these steps, you can bolster the security of your TrueNAS server by configuring a Let's Encrypt certificate, ensuring encrypted communication for your NAS web interface.
