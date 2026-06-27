Cloudflare Tunnel Setup (Docker)

Summary
Create and run a Cloudflare Tunnel (cloudflared) to expose a local service securely through Cloudflare without directly opening inbound ports on your host.

Discussion
Cloudflare Tunnel is useful when you want secure public access to an internal app (for example, a dashboard or API container) while keeping your origin private behind Cloudflare.

Requirements
- Cloudflare account with Zero Trust access
- Domain managed in Cloudflare
- Docker and Docker Compose on the host
- Local service/container already running and reachable on the Docker network

Execution Mode (Required)
- The agent must suggest exactly ONE command at a time.
- After each command, the agent must wait for the user to confirm success or share output before continuing.
- Do not batch commands.
- If a command fails, stop and troubleshoot before moving forward.

Workflow

Part 0: Ask for required inputs before any command
The agent must ask:
1. What is your Docker Compose service name (service key in docker-compose.yml)?
2. What subdomain do you want to use?
3. What is your app’s internal port (for example 80, 3000, 8080)?
4. What is your project path on the server?

Rule: The docker-compose service name should match the requested subdomain whenever possible.
Example: If subdomain is ozlanka, the service key should be ozlanka.

Part 1: Align docker-compose service name with subdomain (if needed)
If service name does not match the chosen subdomain, update docker-compose.yml first, then recreate services.

Agent command sequence (one-by-one with confirmation):
1) Open project directory:
   cd <project_path>

2) Review compose services:
   docker compose config --services

3) Edit docker-compose.yml so the service key equals the chosen subdomain.
   (Agent should provide exact edit guidance and wait for confirmation.)

4) Recreate stack after service rename:
   docker compose down --remove-orphans

5) Start updated app service:
   docker compose up -d <subdomain>

6) Verify running containers:
   docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

7) Verify compose service list now includes <subdomain>:
   docker compose config --services

Do not continue to Cloudflare steps until the renamed service is healthy.

Part 2: Create tunnel in Cloudflare Zero Trust
1. Go to Cloudflare Zero Trust dashboard.
2. Navigate to Networks → Tunnels.
3. Click Create a tunnel.
4. Choose Cloudflare Tunnel (connector).
5. Name the tunnel (recommended: same as subdomain) and save.
6. In Install and run a connector, select Docker.
7. Copy the docker run command Cloudflare provides.

Important token rule:
- Do NOT edit .env for tunnel token.
- Do NOT extract token into environment variables.
- Run the Cloudflare-provided docker command directly on the server.

Part 3: Confirm Docker network to use
The cloudflared container must be on the same Docker network as the app service.

Agent command sequence (one-by-one with confirmation):
1) From project directory:
   docker network ls

2) Identify compose default network (typically <project_name>_default).

3) Confirm app container is attached to that network:
   docker inspect <app_container_name> --format '{{json .NetworkSettings.Networks}}'

Part 4: Run cloudflared using Cloudflare-provided command
Use the exact command from Cloudflare and set the correct network.

Example format:
docker run -d --name cloudflared-<subdomain> --restart always --network <compose_project>_default cloudflare/cloudflared:latest tunnel --no-autoupdate run --token "eyJ..."

Agent command behavior:
- Suggest one docker run command only.
- Wait for output/confirmation.
- If container name already exists, stop and guide cleanup before retry.

Part 5: Configure public hostname routing
1. In Cloudflare Zero Trust → Networks → Tunnels → <your-tunnel> → Public Hostnames, click Add a public hostname.
2. Fill in:
   - Subdomain: <subdomain>
   - Domain: <your-domain>
   - Path: (leave empty)
   - Service: http://<compose-service-name>:<internal-port>
3. Save.

CRITICAL
- Service must be http://<compose-service-name>:<internal-port>.
- Use compose service name (from docker-compose.yml), not auto-generated container name.
- If you renamed the service to match subdomain, this should be http://<subdomain>:<port>.

Part 6: Validate tunnel health
Agent command sequence (one-by-one with confirmation):
1) Check logs:
   docker logs -f cloudflared-<subdomain>

Expected signs:
- "Registered tunnel connection" messages
- "Updated to new configuration"
- No "Tunnel not found" or "no such host"

2) Check tunnel status in Zero Trust dashboard shows Active.

3) Open https://<subdomain>.<your-domain> in browser.

Verification
- Tunnel status shows Active in Zero Trust dashboard.
- Public hostname resolves and loads your service.
- Origin service remains private (no direct public inbound rule required).

Troubleshooting

"Tunnel not found" error
- Usually wrong token or wrong tunnel command copied.
- Re-copy the docker command from Cloudflare tunnel page and run it exactly.
- Ensure no extra characters were added around token.

"Control stream encountered a failure while serving"
- Usually means the tunnel has no public hostname configured yet.
- Add hostname route in Part 5.

502 Bad Gateway / "no such host"
- cloudflared cannot resolve/reach the app service.
- Verify cloudflared is on app’s compose network.
- Verify app service is running.
- Verify Service URL uses compose service name.
- Test from cloudflared container:
  docker exec cloudflared-<subdomain> wget -qO- http://<compose-service-name>:<internal-port>

Public hostname returns 502/503
- Validate tunnel route service URL/port matches compose service.
- Confirm app container is healthy.
- Confirm app responds internally from cloudflared container.

DNS or hostname issues
- Confirm hostname is attached to correct tunnel.
- Confirm domain is active in Cloudflare and DNS has propagated.

Intermittent disconnects
- Check host resource pressure and network stability.
- Ensure container restart policy is enabled.
- Review Cloudflare status/incident feed if issue is widespread.

Notes and Cautions
- Treat tunnel tokens as secrets; never paste them into chat or share them in plain text.
- Run Cloudflare-provided docker command directly instead of storing token in .env for this workflow.
- Prefer Cloudflare Access policies for app-level authentication.
- Pin image versions in production instead of always using latest.
- When renaming docker-compose services, run docker compose down --remove-orphans before docker compose up -d.
- Keep cloudflared on the same Docker network as the app service.
