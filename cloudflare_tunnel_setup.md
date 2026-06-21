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
    
    Workflow
    
    Part 1: Create the tunnel in Cloudflare Zero Trust
    1. Go to Cloudflare Zero Trust dashboard.
    2. Navigate to Networks → Tunnels.
    3. Click Create a tunnel.
    4. Choose Cloudflare Tunnel (connector).
    5. Name the tunnel (e.g., jobhuntcrafter) and save.
    6. In Install and run a connector, select Docker.
    7. Copy ONLY the token string (the long eyJ... value after --token). Do not copy the full docker command — just the token.
    
    Part 2: Configure local environment
    On your server, add the token to .env (no quotes, no extra characters):
    
    cd /home/ubuntu/JobHunt_Crafter_AI
    nano .env
    
    Find or add this line:
    
    TUNNEL_TOKEN=***
    
    Save and exit (Ctrl+O, Enter, Ctrl+X in nano).
    
    Do NOT paste the token into the chat. Write it directly on the server. Tokens are secrets and should never be shared in plain text over messaging channels.
    
    Part 3: Start the app container
    Make sure your app container is running on the compose project's default network:
    
    cd /home/ubuntu/JobHunt_Crafter_AI
    docker compose up -d jobhuntcrafter
    
    Verify it is running:
    
    docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
    
    Part 4: Run cloudflared container on the same Docker network
    The cloudflared container must be on the same Docker network as the app container so it can resolve the app by service name. Find your compose network name:
    
    docker network ls | grep jobhunt
    
    Then run cloudflared on that network (replace jobhunt_crafter_ai_default with your actual network name if different):
    
    docker run -d \
      --name cloudflared-uat \
      --restart always \
      --network jobhunt_crafter_ai_default \
      cloudflare/cloudflared:latest \
      tunnel --no-autoupdate run --token "$TUNNEL_TOKEN"
    
    NOTE: This command reads TUNNEL_TOKEN from your environment. If you have not exported it, use this instead:
    
    TOKEN=*** grep '^TUNNEL_TOKEN' .env | cut -d= -f2)
    docker run -d \
      --name cloudflared-uat \
      --restart always \
      --network jobhunt_crafter_ai_default \
      cloudflare/cloudflared:latest \
      tunnel --no-autoupdate run --token "$TOKEN"
    
    Part 5: Configure public hostname routing
    1. In Cloudflare Zero Trust → Networks → Tunnels → <your-tunnel> → Public Hostnames, click Add a public hostname.
    2. Fill in:
       - Subdomain: jobhuntcrafter
       - Domain: getback2basics.net
       - Path: (leave empty)
       - Service: http://jobhuntcrafter:80
    3. Save.
    
    CRITICAL: The Service value must be http://<compose-service-name>:<internal-port>. The service name is the name defined in docker-compose.yml (e.g., jobhuntcrafter), NOT the auto-generated container name (e.g., jobhunt_crafter_ai-jobhuntcrafter-1). The port is the internal port the container listens on (80 for nginx), NOT the host-mapped port (8080).
    
    Part 6: Validate tunnel health
    1. Check container logs:
    
       docker logs -f cloudflared-uat
    
       You should see:
       - "Registered tunnel connection" messages with location codes (e.g., syd05, cbr01)
       - "Updated to new configuration" with your ingress hostname and service
       - No "Tunnel not found" or "no such host" errors
    
    2. Check the Cloudflare Zero Trust dashboard — the tunnel status should show as Active (not Inactive).
    
    3. Visit https://jobhuntcrafter.getback2basics.net in your browser. It should load your app.
    
    Verification
    - Tunnel status shows Active in Zero Trust dashboard.
    - Public hostname resolves and loads your service.
    - Origin service remains private (no direct public inbound rule required).
    
    Troubleshooting
    
    "Tunnel not found" error
    - The token in .env does not match the tunnel in Cloudflare. Re-copy the token from the Cloudflare dashboard and update .env on the server.
    - Verify the token decodes correctly: echo '<token>' | base64 -d
    - Ensure you did not accidentally copy extra characters or the full docker command.
    
    "Control stream encountered a failure while serving"
    - Usually means the tunnel has no public hostname configured yet. Add the hostname route in Part 5.
    
    502 Bad Gateway / "no such host"
    - The cloudflared container cannot reach the app. This means cloudflared is on the wrong Docker network.
    - Verify cloudflared is on the same network as your app: docker inspect cloudflared-uat | grep NetworkMode
    - Verify the app is running: docker ps | grep jobhuntcrafter
    - Verify the Service URL in the tunnel config uses the compose service name, not the container name.
    - Test connectivity from cloudflared to app: docker exec cloudflared-uat wget -qO- http://jobhuntcrafter:80
    
    Public hostname returns 502/503
    - Validate the upstream service URL/port in the tunnel route matches the compose service.
    - Confirm the app container is running and healthy: docker ps
    - Confirm the app responds internally: docker exec cloudflared-uat wget -qO- http://jobhuntcrafter:80
    
    DNS or hostname issues
    - Confirm hostname is attached to the correct tunnel.
    - Confirm domain is active in Cloudflare and DNS has propagated.
    
    Intermittent disconnects
    - Check host resource pressure and network stability.
    - Ensure container restart policy is enabled.
    - Review Cloudflare status/incident feed if issue is widespread.
    
    Notes and Cautions
    - Treat tunnel tokens as secrets; never paste them into chat or share them in plain text. Always write them directly on the server.
    - Prefer Cloudflare Access policies for app-level authentication.
    - Pin image versions in production instead of always using latest.
    - When renaming a docker-compose service, old containers hold the name and block recreation. Always run docker compose down --remove-orphans before docker compose up -d after renaming.
    - The cloudflared container must be on the same Docker network as the app container. If you use docker run, specify --network <compose-project>_default. If you use docker-compose, depends_on handles this automatically.
