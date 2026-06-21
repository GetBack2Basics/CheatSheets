Cloudflare Tunnel Setup (Docker)
Summary
Create and run a Cloudflare Tunnel (cloudflared) to expose a local service securely through Cloudflare without directly opening inbound ports on your host.

Discussion
Cloudflare Tunnel is useful when you want secure public access to an internal app (for example, a dashboard or API container) while keeping your origin private behind Cloudflare.

Requirements
Cloudflare account with Zero Trust access
Domain managed in Cloudflare
Docker and Docker Compose (or docker run) on the host
Local service/container already running and reachable on the Docker network
Workflow
Part 1: Create the tunnel in Cloudflare Zero Trust
Go to Cloudflare Zero Trust dashboard.
Navigate to Networks → Tunnels.
Click Create a tunnel.
Choose Cloudflare Tunnel (connector).
Name the tunnel and save.
In Install and run a connector, select Docker.
Copy the generated token (this is your TUNNEL_TOKEN).
Part 2: Configure local environment
Set token in .env (no quotes):

TUNNEL_TOKEN=<your-full-token-here>
Part 3: Run cloudflared container
Example with Docker run:

docker run -d \
  --name cloudflared \
  --restart unless-stopped \
  cloudflare/cloudflared:latest \
  tunnel --no-autoupdate run --token "$TUNNEL_TOKEN"
Example with Docker Compose service:

services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token ${TUNNEL_TOKEN}
Part 4: Configure public hostname routing
In the tunnel configuration, add a public hostname mapping to your local service, for example:

Hostname: app.example.com
Service: http://app:8080 (or your actual internal service URL)
Part 5: Validate tunnel health
docker logs -f cloudflared
You should see successful connector registration and active connection messages.

Verification
Tunnel status appears healthy in Zero Trust dashboard.
Public hostname resolves and loads your service.
Origin service remains private (no direct public inbound rule required).
Troubleshooting
Tunnel does not connect
Verify TUNNEL_TOKEN is exact and unquoted in .env.
Confirm outbound connectivity from host.
Check container logs for auth/token errors.
Public hostname returns 502/503
Validate upstream service URL/port in tunnel route.
Confirm app container is reachable from cloudflared container/network.
Check app logs and container health.
DNS or hostname issues
Confirm hostname is attached to the correct tunnel.
Confirm domain is active in Cloudflare and DNS has propagated.
Intermittent disconnects
Check host resource pressure and network stability.
Ensure container restart policy is enabled.
Review Cloudflare status/incident feed if issue is widespread.
Notes and Cautions
Treat tunnel tokens as secrets; rotate if leaked.
Prefer Cloudflare Access policies for app-level authentication.
Pin image versions in production instead of always using latest.
