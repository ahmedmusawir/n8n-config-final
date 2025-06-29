# n8n-config-final
This is the final config for our n8n self-hosted rig: nginx config and docker-compose.yml

## nginx config with webhook headers. essential for 'connection lost' error fix

```
server {
    server_name n8n.cyberizewebdevelopment.com;
    
    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # CRITICAL: WebSocket upgrade headers (fixes "Connection lost")
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Additional settings for better n8n performance
        proxy_buffering off;
        proxy_cache_bypass $http_upgrade;
        proxy_http_version 1.1;
        
        # Increase timeouts for long-running workflows
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        client_max_body_size 100M;
    }
    
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/n8n.cyberizewebdevelopment.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/n8n.cyberizewebdevelopment.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = n8n.cyberizewebdevelopment.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
    
    listen 80;
    server_name n8n.cyberizewebdevelopment.com;
    return 404; # managed by Certbot
}
```

## The docker-compose.yml w/ proxy trust var is essential for 'connection lost' error fix

```
version: "3.8"

services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      # Keep localhost binding for security (good practice from your friend)
      - "127.0.0.1:5678:5678"
    environment:
      # --- Basic Configuration ---
      - TZ=America/New_York
      
      # --- CORE URL Configuration ---
      - N8N_URL=https://n8n.cyberizewebdevelopment.com
      - WEBHOOK_URL=https://n8n.cyberizewebdevelopment.com
      
      # --- CRITICAL: Enable Express Proxy Trust ---
      - N8N_PROXY_TRUST=true
      - N8N_HOST=0.0.0.0
      
      # --- WebSocket Configuration (Fixes Connection Lost) ---
      - N8N_DISABLE_UI=false
      - N8N_SECURE_COOKIE=false  # Since proxy handles HTTPS
      
      # --- Resend SMTP Configuration ---
      - N8N_EMAIL_MODE=smtp
      - N8N_SMTP_HOST=smtp.resend.com
      - N8N_SMTP_PORT=465
      - N8N_SMTP_USER=resend
      - N8N_SMTP_PASS=re_JTNwA6CA_6zbsd9BdAKBsHZtEerufNMnJ
      - N8N_SMTP_SENDER=invitations@cyberizedev.com
      - N8N_SMTP_SSL=true
      - N8N_USER_MANAGEMENT=true
      - N8N_USER_INVITATION=true
      
      # --- Permissions & Task Runners ---
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true

    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```
