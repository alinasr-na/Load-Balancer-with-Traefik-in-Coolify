# Guide to Setting Up Internal Load Balancer with Traefik in Coolify

This guide explains how to set up an internal load balancer for an application running on multiple Docker containers on a single server, using Traefik within the Coolify environment. This process enhances your application's **availability** and **scalability**.

## 📋 Prerequisites

1.  **Coolify and Traefik Installed**: You have a server with Coolify installed, and the Traefik service is active.
2.  **Multi-Container Deployment**: Your application is deployed on at least two containers on the same server.
3.  **Domain Name**: You have a domain name (e.g., from Cloudflare) for which you can manage DNS records.

## 🚀 Step-by-Step Guide

### Step 1: Identify Container Information

First, you need to find the names and internal ports of your containers. To do this, run the following command in your server's terminal:

```bash
docker ps
```

The output of this command will look something like this (this is an example):

```
CONTAINER ID   IMAGE   ...   PORTS      NAMES
...
9a68244fd673   ...     ...   3000/tcp   swc4o40o88w4ogwkws8wgsg0-070818606214
d80b0cc9440b   ...     ...   3000/tcp   pckwsowwkco0ok8s808ksog8-070454034323
...
```

From this output, you need two pieces of information:
*   **Container Names**: `swc4o40o88w4ogwkws8wgsg0-070818606214` and `pckwsowwkco0ok8s808ksog8-070454034323`
*   **Internal Port**: `3000` (The `PORTS` column shows that this port is not mapped to the host and is internal).

### Step 2: Configure DNS in Cloudflare

1.  Log in to your Cloudflare account and select your domain.
2.  Create an `A` record with your desired subdomain (e.g., `test`).
3.  In the **IPv4 address** field, enter the public IP address of your Coolify server.
4.  **Important**: Ensure the proxy status (the orange cloud) is **disabled** (grey cloud / "DNS only"). This allows Traefik to request an SSL certificate directly from Let's Encrypt.

### Step 3: Create the Traefik Dynamic Configuration File

The best way to add custom configurations is to create a separate file, so it doesn't get overwritten by Coolify updates.

1.  In the Coolify UI, navigate to **Servers** -> your server -> **Settings** -> **Proxy**.
2.  Click the **+ Add** button.
3.  Enter a **filename** with a `.yaml` extension. For example: `loadbalancer.yaml`
4.  Paste the following **content** into the file's content box:

```yaml
http:
  middlewares:
    redirect-to-https:
      redirectscheme:
        scheme: https
    gzip:
      compress: true
  routers:
    lb-https:
      tls:
        certResolver: letsencrypt
      middlewares:
        - gzip
      entryPoints:
        - https
      service: lb-https
      # >>> Enter your domain here
      rule: Host(`test.meditechx.net`)
    lb-http:
      middlewares:
        - redirect-to-https
      entryPoints:
        - http
      service: noop
      # >>> Enter your domain here
      rule: Host(`test.meditechx.net`)
  services:
    lb-https:
      loadBalancer:
        servers:
          # >>> Enter your container names and internal port here
          - url: 'http://swc4o40o88w4ogwkws8wgsg0-070818606214:3000'
          - url: 'http://pckwsowwkco0ok8s808ksog8-070454034323:3000'
    noop:
      loadBalancer:
        servers:
          - url: ''
```

5.  Save the file. Traefik will automatically load this new configuration.

## ✅ Verification and Troubleshooting

### How to Check if the Load Balancer is Working

The easiest way is to check the logs of your application containers.

1.  Refresh your browser multiple times by visiting `https://test.meditechx.net`.
2.  Then, check the logs of each container separately:

    ```bash
    docker logs swc4o40o88w4ogwkws8wgsg0-070818606214
    docker logs pckwsowwkco0ok8s808ksog8-070454034323
    ```

    If everything is configured correctly, you will see that the requests have been distributed between the two containers.

### Common Error: `502 Bad Gateway`

This error means that Traefik was unable to connect to your containers.
*   **Main Cause**: This is usually due to an incorrect container name or port in the `loadbalancer.yaml` file.
*   **Solution**: Double-check the container names and internal ports against the output of the `docker ps` command.

### Load Balancing Algorithm

The default algorithm used by Traefik is **`wrr` (Weighted Round Robin)**. When you don't define a specific "weight" for the servers, this algorithm behaves exactly like simple **Round Robin**, distributing requests to the servers in a cyclical order.
