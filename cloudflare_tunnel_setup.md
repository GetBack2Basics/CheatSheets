  GNU nano 7.2                                cloudflare-tunnel-guide.md                                          
# Cloudflare Tunnel Configuration Guide

This guide explains how to create, configure, and maintain Cloudflare Tunnels (Cloud-Managed Tunnels) for routing>

---

## Overview

Cloudflare Tunnels (`cloudflared`) allow you to securely expose a local container to the public internet without >

```
[ Internet ] ---> [ Cloudflare Edge ] ===( HTTP/2 Tunnel )===> [ cloudflared container ] ---> [ app container ]
```

---

## Part 1: Creating a Tunnel in Cloudflare Zero Trust

1. **Access the Zero Trust Dashboard:**
   * Go to the [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/).
   * Select your account.

2. **Create the Tunnel:**
   * Navigate to **Networks** -> **Tunnels**.
   * Click **Create a tunnel**.
   * Choose **Cloudflare Tunnel (connector)** and click **Next**.
   * Enter a descriptive name (e.g., `transport-crafter`) and click **Save tunnel**.

3. **Retrieve the Tunnel Token:**
   * On the **Install and run a connector** page, select the **Docker** tab.
   * Copy the full token string (starts with `ey...`, ~184 characters).
   * **This is your `TUNNEL_TOKEN`.**

---

## Part 2: Local Docker Integration

1. **Configure Environment Variables:**
   * Open the `.env` file in the project root.
   * Set the `TUNNEL_TOKEN` variable:
     ```env
     TUNNEL_TOKEN=<your-full-token-here>
     ```
   * **IMPORTANT:** Do NOT use quotes around the token value. Write it as a raw string:
     ```env
     TUNNEL_TOKEN=eyJhIjoi...FeiJ9
     ```

                                                [ Read 143 lines ]
^G Help         ^O Write Out    ^W Where Is     ^K Cut          ^T Execute      ^C Location     M-U Undo
^X Exit         ^R Read File    ^\ Replace      ^U Paste        ^J Justify      ^/ Go To Line   M-E Redo
