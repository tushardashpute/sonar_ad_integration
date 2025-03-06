# SonarQube with SSL and Azure AD Integration using Let's Encrypt

This guide provides step-by-step instructions to deploy SonarQube with SSL encryption using Let's Encrypt and integrate it with Azure Active Directory (AAD) for authentication.

## Prerequisites
- A server (VM or container host) with a public DNS entry (e.g., `sonar.example.com`).
- Docker and Docker Compose installed.
- A valid Azure AD tenant and app registration.
- A registered domain with DNS settings configured for Let's Encrypt.

## 1. Configure Azure AD Authentication

### Register an Application in Azure AD
1. Go to [Azure AD App Registrations](https://portal.azure.com/)
2. Click **New registration** and set:
   - **Name**: `SonarQube`
   - **Supported account types**: Choose "Accounts in this organizational directory only"
   - **Redirect URI**: Select "Web" and enter `https://sonar.example.com/oauth2/callback/aad`
3. Click **Register**.
4. Copy **Application (client) ID** and **Directory (tenant) ID**.

### Configure Client Secret
1. In the **Azure AD App Registration**, navigate to **Certificates & secrets**.
2. Under **Client secrets**, click **New client secret**.
3. Set a description and expiration period, then click **Add**.
4. Copy the generated secret and store it securely.

### Set API Permissions
1. Navigate to **API Permissions**.
2. Click **Add a permission** → **Microsoft Graph**.
3. Choose **Delegated permissions** and select:
   - `openid`
   - `email`
   - `profile`
4. Click **Add permissions**.
5. Click **Grant admin consent** for your organization.

## 2. Create Directories for SSL and SonarQube
```sh
mkdir -p ~/sonarqube/{conf,data,logs,extensions,certs}
```
Place your SSL certificate (`cert.pem`) and private key (`privkey.pem`) inside `~/sonarqube/certs/`.

Also, copy your Nginx configuration file:
```sh
cp nginx.conf ~/sonarqube/nginx.conf
```

## 3. Download and Install Azure AD Plugin
```sh
mkdir -p ~/sonarqube/extensions/plugins
curl -L -o ~/sonarqube/extensions/plugins/sonar-auth-aad-plugin.jar https://github.com/hkamel/sonar-auth-aad/releases/download/1.3.1/sonar-auth-aad-plugin-1.3.1.jar
```

## 4. Generate SSL Certificate using Let's Encrypt

We will use [Certbot](https://certbot.eff.org/) to obtain an SSL certificate.

### Install Certbot
```sh
sudo apt update
sudo apt install certbot
```

### Obtain SSL Certificate
```sh
sudo certbot certonly --standalone -d sonar.example.com
```
This will generate SSL certificates at `/etc/letsencrypt/live/sonar.example.com/`.

Copy the certificates to the specified path:
```sh
cp /etc/letsencrypt/live/sonar.example.com/fullchain.pem ~/sonarqube/certs/
cp /etc/letsencrypt/live/sonar.example.com/privkey.pem ~/sonarqube/certs/
```

## 5. Run SonarQube with Nginx as a Reverse Proxy

### Create `docker-compose.yml`
```yaml
version: '3'
services:
  sonarqube:
    image: sonarqube:developer
    container_name: sonarqube
    restart: unless-stopped
    environment:
      - sonar.web.context=/
      - SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true
      - sonar.core.serverBaseURL=https://sonar.example.com
      - sonar.auth.aad.enabled=true
      - sonar.auth.aad.clientId=<your-client-id>
      - sonar.auth.aad.clientSecret=<your-client-secret>
      - sonar.auth.aad.tenantId=<your-tenant-id>
      - sonar.auth.aad.loginStrategy=Unique
      - sonar.auth.aad.allowUsersToSignUp=true
    volumes:
      - ~/sonarqube/conf:/opt/sonarqube/conf
      - ~/sonarqube/data:/opt/sonarqube/data
      - ~/sonarqube/logs:/opt/sonarqube/logs
      - ~/sonarqube/extensions:/opt/sonarqube/extensions
    networks:
      - sonarnet

  nginx:
    image: nginx:latest
    container_name: sonarqube-nginx
    restart: unless-stopped
    ports:
      - "443:443"
    volumes:
      - ~/sonarqube/certs:/etc/nginx/certs:ro
      - ~/sonarqube/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - sonarqube
    networks:
      - sonarnet

networks:
  sonarnet:
```

### Create `nginx.conf`
```nginx
 events {
    worker_connections 1024;
}

http {
    upstream sonarqube {
        server sonarqube:9000;
    }

    server {
        listen 443 ssl;
        server_name sonar.example.com;

        ssl_certificate /etc/nginx/certs/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/privkey.pem;

        location / {
            proxy_pass http://sonarqube;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_redirect http:// https://;
        }
    }
}
```

## 6. Deploy and Start Services

```sh
docker-compose up -d
```

## 7. Renew SSL Certificate (Automate with Cron Job)

```sh
sudo certbot renew --quiet
```
To automate, add a cron job:
```sh
sudo crontab -e
```
Add this line:
```sh
0 3 * * * certbot renew --quiet && docker restart sonarqube-nginx
```

## Troubleshooting

### Check SonarQube Logs
```sh
docker logs -f sonarqube
```

### Check Nginx Logs
```sh
docker logs -f sonarqube-nginx
```

### Verify SSL Configuration
```sh
openssl s_client -connect sonar.example.com:443
```

## Conclusion
You have successfully set up SonarQube with SSL using Let's Encrypt and integrated Azure AD authentication. 🎉

