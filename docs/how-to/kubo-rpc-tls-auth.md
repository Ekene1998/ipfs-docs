---
title: Secure Kubo RPC with TLS and HTTP Auth
description: Learn how to set up TLS for Kubo with Caddy reverse proxy for secure RPC API access over public networks.
---

# Secure Kubo RPC with TLS and HTTP Auth

This guide will help you set up two things:

- **Transport Encryption:** Caddy as a reverse proxy with automatic TLS certificate management for your Kubo node using a domain you control.
- **Authentication:** Basic HTTP authentication for the [Kubo RPC API](../reference/kubo/rpc.md).

This is highly recommended if you run your own Kubo node and want to use the Kubo RPC API over public networks, for example, to pin CIDs from CI, or other services. Since the Kubo RPC API is exposed over plain HTTP, TLS is used to ensure the connection to the API is encrypted.

## Prerequisites

Before starting, ensure you have:

- A domain name (referred to as `YOUR_DOMAIN`) with its A record pointing to your server's IP address
- Kubo running on a server/VM with a public IP address
- Port 443 open on your server's firewall
- [Caddy web server](https://caddyserver.com/) installed on the server

The guide assumes the Caddy process is managed by systemd. If you are using a different process manager or Docker, you will need to adjust the configuration accordingly.

## Configure Kubo

First, you'll need to configure Kubo to work with the reverse proxy. Edit your Kubo config file (usually located at `~/.ipfs/config`) and update the API section:

```
"API": {
  "HTTPHeaders": {
    "Access-Control-Allow-Origin": ["https://YOUR_DOMAIN"],
    "Access-Control-Allow-Credentials": ["true"]
  },
  "Authorizations": {
    "api": {
      "AuthSecret": "basic:hello:world123"
      "AllowedPaths": [
          "/api/v0"
        ]
    }
  }
}
```

This configuration:

- Sets CORS headers to allow requests from `YOUR_DOMAIN`. Kubo will match the `host` header in the request with the `Access-Control-Allow-Origin` from the configuration, so you need to ensure the origin is correct.
- Restricts API access to the Kubo RPC API, allowing access to the `/api/v0` endpoints with basic HTTP authentication.

> **Note:** You should set the `AuthSecret` to a stronger username and password combination.

## Configure Caddy

Create or edit your Caddyfile (typically at `/etc/caddy/Caddyfile`) with the following configuration, making sure to replace `YOUR_DOMAIN` with your actual domain name:

```
YOUR_DOMAIN {
  reverse_proxy localhost:5001

  log {
      output stdout
      format json
      level INFO
  }
}
```

This configuration:

- Sets up a reverse proxy to Kubo's API on port 5001
- Logs requests to the Kubo API in JSON format to stdout

## Restart Caddy

Restart the Caddy service to apply the changes:

```bash
sudo systemctl restart caddy
```

## Test the Connection

To verify everything is working correctly, test the connection using the IPFS CLI, making sure to replace `YOUR_DOMAIN` with your actual domain name:

```bash
ipfs id --api /dns/YOUR_DOMAIN/tcp/443/https --api-auth basic:hello:world123
```

If successful, you should see your node's identify displayed. The command connects to your Kubo node through the secure HTTPS endpoint using basic authentication.

## Security Considerations

- Change the `AuthSecret` to a strong username and password combination
- Consider restricting the `AllowedPaths` further based on your needs
- Keep your Caddy and Kubo installations updated
- Regularly monitor the logs for any suspicious activity

## Troubleshooting

If you encounter issues:

1. Check Caddy logs
2. Verify your domain's DNS settings, ensuring the A record is correct. Sometimes changes can take a few minutes to propagate (depending on the TTL of the DNS record).
3. Ensure port 443 is open and not blocked by your firewall
4. Check that Kubo is running and accessible on localhost:5001
