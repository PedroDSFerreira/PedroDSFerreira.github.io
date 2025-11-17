---
title: "Self-Hosting with Tailscale"
date: 2025-11-17
cover: "/posts/tailscale/cover.jpg"
tags: [tailscale, networking, self-hosted, homelab, docker]
---

Have you ever wanted to build your own Homelab, but thought it was way too complex to do so? In this post, I will talk about what Tailscale is and how you can use it to access your self-hosted services anywhere in the world, in a private and secure way.

I’ll also explore how to expose a service to the public internet using Tailscale’s “funnel” feature.

> TLDR: Jump to the "Getting Started" chapter to setup your homelab.

## What is Tailscale?

Tailscale is a zero-configuration VPN that connects your devices over the internet. Under the hood, it uses the WireGuard protocol to create a private mesh network (a **tailnet**), where every connected device (**node**) can communicate directly with one another. This means that, instead of manually configuring network settings like static IPs and port-forwarding, you only need to join your devices to the tailnet and they’re instantly part of your private network.

![Image by [Avery Pennarun](https://tailscale.com/blog/how-tailscale-works)](/assets/img/posts/tailscale/mesh-vpn.svg)
<center>
  <p>
    Image from <a href="https://tailscale.com/blog/how-tailscale-works">Avery Pennarun</a>
  </p>
</center>

One of the great features of Tailscale is its built-in DNS service, known as **MagicDNS**. Every new node in your tailnet is automatically assigned a unique subdomain, which makes connecting to your devices as simple as using easy-to-remember names instead of IP addresses.

Alongside this, Tailscale automatically provisions TLS certificates using Let’s Encrypt, securing the communication with your services without any additional configuration.

---

## Hosting Containerized Services With Tailscale

When running multiple containers, you might consider running a **Tailscale sidecar**, an auxiliary service that runs alongside your application containers. Its role is to expose the desired container to your tailnet (an API service, a webpage, etc), handling all the networking aspects, such as joining your tailnet, and handling secure connections and DNS resolution.

This is great if you only intend to host a few services. However, deploying a sidecar for every service can be taxing on your server, with the load increasing proportionally to the number of services you are exposing.

A more efficient approach is TSDProxy, an open-source service that runs next to your containers and acts as a reverse proxy, exposing your services through Tailscale. For each labeled container, it automatically creates a Tailscale node and routes the traffic accordingly.

![Image from [almeidapaulopt](https://almeidapaulopt.github.io/tsdproxy/docs/)](/assets/img/posts/tailscale/sidecar-vs-proxy.png)

<center>
  <p>
    Image from <a href="https://almeidapaulopt.github.io/tsdproxy/docs/">almeidapaulopt</a>
  </p>
</center>

Rather than running a separate sidecar for every service, TSDProxy acts as a central orchestrator that handles the registration and discovery process for you.

---

## Getting Started

All Docker-related configurations used in this guide are available on my [GitHub repository](https://github.com/PedroDSFerreira/homelab). It also includes templates for other services to help you get started quickly.

### 0. Requirements

Make sure you have a [Tailscale account](https://login.tailscale.com/login) and a machine with [Docker](https://www.docker.com/get-started/) installed.

### 1. Update Your ACL File

Tailscale’s Access Control List (ACL) file is where you define rules for how devices interact within your tailnet. Add the following snippet to your [ACL file](https://login.tailscale.com/admin/acls/file):

```json
{
  "tagOwners": {
    "tag:container": ["autogroup:admin"]
  }
}
```

This ensures that the network access of the devices with the tag `container` is managed by your admin group.

### 2. Enable MagicDNS and HTTPS Certificates

Go to the [DNS tab](https://login.tailscale.com/admin/dns) in the admin panel, and enable **MagicDNS** and **HTTPS Certificates.**

### 3. Generate an Auth Key

Containers need an auth key to join your tailnet. Head over to the [auth keys](https://login.tailscale.com/admin/settings/keys) page to create the new key.

Make sure to associate the `container` tag created in the previous step with the key. You can also mark it as **reusable** if you plan to deploy multiple containers using the same credentials.

Save this key, as you’ll need it for configuring your proxy.

---

### 4. Proxy Service Setup

To set up TSDProxy, create a `docker-compose.yaml` file with the following content:

```yaml
services:
  tsdproxy:
    image: ghcr.io/almeidapaulopt/tsdproxy:1.4.7
    container_name: tsd-proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - tsdproxy-data:/data
      - ./config/:/config
    restart: unless-stopped
    ports:
      - 8080:8080

volumes:
  tsdproxy-data:
```

Launch the proxy with:

```bash
docker compose up -d
```

A configuration file will be generated at `config/tsdproxy.yaml`. Open this file and add your auth key under the `tailscale` section (you may need to change the file’s write permissions):

```yaml
tailscale:
  providers:
    default:
      authKey: "<your-auth-key>"
```

Restart the TSDProxy container to apply the updated settings.

---

## Deploying and Exposing Your Services

With the proxy up and running, we can deploy the services to be exposed in the tailnet. Let’s take **Portainer,** an open-source container management platform, as an example.

### Deploying Portainer

First, add to the `docker-compose.yaml` file the Portainer configuration:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:alpine
    container_name: portainer
    restart: always
    ports:
      - 9000:9000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data

volumes:
  portainer-data:
```

When you launch Portainer using:

```bash
docker compose up -d
```

you can access the UI at `localhost:9000`. To integrate Portainer with TSDProxy and have it automatically registered on your tailnet, simply add the following labels to the Portainer service configuration:

```yaml
labels:
  tsdproxy.enable: "true"
  tsdproxy.name: "portainer"
```

After redeploying the service, TSDProxy detects Portainer and assigns it a unique subdomain. You can then access it at `https://portainer.<your-tailnet-domain>.ts.net`.

---

# Exposing Services to the Public Internet

Sometimes you might need to expose a service to the public internet, while keeping your overall network secure. Tailscale’s **funnel** feature makes this possible by allowing you to define specific services as externally accessible without opening up your entire tailnet.

To enable funnelling, start by updating your ACL configuration to specify that nodes tagged with `container` are eligible for funnel access. Add the following snippet to your ACL:

```json
"nodeAttrs": [
  {
    "target": ["tag:container"],
    "attr": ["funnel"]
  }
],
```

Next, modify your Portainer configuration to include a funnel label:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:alpine
    container_name: portainer
    restart: always
    ports:
      - 9000:9000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data
    labels:
      tsdproxy.enable: "true"
      tsdproxy.name: "portainer"
      tsdproxy.funnel: "true"

volumes:
  portainer-data:
```

After redeploying the configuration, Portainer remains accessible via its subdomain, but it’s now reachable from outside your tailnet. This approach is handy for services like public dashboards or API gateways that need to be accessed by external users.

That said, exposing anything to the public internet always carries risk. Attackers may try to access dashboards, exfiltrate data, or exploit weak endpoints. Only publish what absolutely needs to be public, enforce strong access controls, and ensure the service itself is hardened before making it public.

I would also recommend creating a separate provider with its own auth key and a tag with different permissions for internal vs external services in `tsdproxy.yaml` (see [docs](https://almeidapaulopt.github.io/tsdproxy/docs/serverconfig/#providers)). This keeps your public-facing components isolated and better contained.

---

## Wrapping Up

Using Docker and Tailscale is one of the easiest ways to start your self-hosting journey. With Tailscale handling most of the networking behind the scenes, you can run and access services anywhere without having to worry about complex networking.

For more self-hostable apps to explore, check out [selfh.st](https://selfh.st/apps/).

Happy homelabbing!
