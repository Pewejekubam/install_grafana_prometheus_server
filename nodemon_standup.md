# **HowTo: Setup Grafana, Prometheus, and TLS with Let's Encrypt on Ubuntu 22.04 Server**

---

## **Step 1: Install Required Dependencies**

1.1 Update the package index:
```bash
sudo apt update
```

1.2 Install required dependencies:
```bash
sudo apt -y install gnupg2 apt-transport-https software-properties-common wget nginx certbot python3-certbot-nginx
```

---

## **Step 2: Add Grafana GPG Key and Repository**

2.1 Add the Grafana GPG key with checksum validation:
```bash
curl -fsSL https://packages.grafana.com/gpg.key | gpg --dearmor | sudo tee /usr/share/keyrings/grafana-archive-keyring.gpg > /dev/null
sha256sum /usr/share/keyrings/grafana-archive-keyring.gpg
```
**Comment:** Verify the checksum against Grafana's official release notes to ensure the key's integrity.

2.2 Add the Grafana APT repository:
```bash
echo "deb [signed-by=/usr/share/keyrings/grafana-archive-keyring.gpg] https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list > /dev/null
```

---

## **Step 3: Install Grafana**

3.1 Update the package index:
```bash
sudo apt update
```

3.2 Install Grafana:
```bash
sudo apt -y install grafana
```

**Comment:** After installation, ensure the ownership and permissions of Grafana's configuration files are correct:
```bash
sudo chown -R grafana:grafana /etc/grafana
sudo chmod -R 755 /etc/grafana
```

---

## **Step 4: Start and Enable Grafana Service**

4.1 Reload systemd to ensure it recognizes the Grafana service:
```bash
sudo systemctl daemon-reload
```

4.2 Start and enable Grafana:
```bash
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

**Comment:** Reloading `systemd` ensures that any new or updated service files are recognized.

---

## **Step 5: Configure Grafana**

5.1 Open the Grafana configuration file:
```bash
sudo vim /etc/grafana/grafana.ini
```

5.2 Modify the `[server]` section:
```ini
[server]
http_addr = 127.0.0.1   # Bind to local IP
http_port = 3000        # HTTP port
root_url = /            # Explicitly set root_url for port mapping setups
domain = example.com    # Replace with your domain
```

**Comment:** Replace `example.com` with the Fully Qualified Domain Name (FQDN) of your server. This is the domain name users will use to access Grafana.

5.3 Save the file and exit:
```bash
:wq
```

5.4 Restart the Grafana service:
```bash
sudo systemctl restart grafana-server
```

---

## **Step 6: Configure Nginx for Certbot**

6.1 Create a temporary Nginx configuration for certbot:
```bash
sudo vim /etc/nginx/sites-available/certbot.conf
```

6.2 Add the following configuration:
```nginx
server {
    listen 80;
    server_name example.com;  # Replace with your FQDN

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 404;  # Only allow certbot validation
    }
}
```

**Comment:** Replace `example.com` with your server's Fully Qualified Domain Name (FQDN). This ensures certbot can validate the domain ownership.

6.3 Save the file and exit:
```bash
:wq
```

6.4 Link the temporary configuration:
```bash
sudo ln -s /etc/nginx/sites-available/certbot.conf /etc/nginx/sites-enabled/
```

6.5 Test the Nginx configuration:
```bash
sudo nginx -t
```

6.6 Reload Nginx to apply the configuration:
```bash
sudo systemctl reload nginx
```

---

## **Step 7: Obtain and Install Let's Encrypt TLS Certificate**

7.1 Request a staging certificate for testing:
```bash
sudo certbot --nginx -d example.com --staging
```

**Description:** The staging certificate is used to test the certbot process without hitting Let's Encrypt's production rate limits. Replace `example.com` with your FQDN.

7.2 Request a production certificate after testing:
```bash
sudo certbot --nginx -d example.com
```

**Description:** The production certificate is the actual TLS certificate used for securing your domain. Replace `example.com` with your FQDN.

**Success Criteria:** If the certbot process is successful, you will see a message similar to:
```plaintext
Congratulations! You have successfully enabled HTTPS on https://example.com
```

**Dependency:** The success of this step is critical for the remainder of the setup. If certbot fails, HTTPS will not be enabled, and subsequent steps relying on the certificate (e.g., Nginx configuration) will also fail.

7.3 Enable automatic certificate renewal:
```bash
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
sudo certbot renew --dry-run
```

---

## **Step 8: Switch to Final Nginx Configuration**

8.1 Replace the temporary Nginx configuration with the final configuration:
```bash
sudo vim /etc/nginx/sites-available/grafana.conf
```

8.2 Add the following configuration:
```nginx
# HTTP server block for certbot and certificate renewal
server {
    listen 80;
    server_name example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$host$request_uri;  # Redirect HTTP to HTTPS
    }
}

# Grafana default on 443
server {
    listen 443 ssl;
    server_name {{ domain_name }};

    ssl_certificate /etc/letsencrypt/live/{{ domain_name }}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{{ domain_name }}/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:3000/;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme
        
        # IP Whitelisting for Grafana access
        allow 192.168.0.0/16;       # Replace with trusted IP addresses
        allow 10.168.0.0/16;        # Example of a trusted CIDR block
        deny all;                   # Deny access to all other IPs
    }
}

# Prometheus on 9443 with IP whitelisting
server {
    listen 9443 ssl;
    server_name {{ domain_name }};

    ssl_certificate /etc/letsencrypt/live/{{ domain_name }}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{{ domain_name }}/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:9090/;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # IP Whitelisting for Prometheus access
        allow 192.168.0.0/16;       # Replace with trusted IP addresses
        allow 10.168.0.0/16;        # Example of a trusted CIDR block
        deny all;                   # Deny access to all other IPs
    }
}
```

8.3 Save the file and exit:
```bash
:wq
```

8.4 Link the final configuration:
```bash
sudo ln -s /etc/nginx/sites-available/grafana.conf /etc/nginx/sites-enabled/
```

8.5 Remove the temporary configuration:
```bash
sudo rm /etc/nginx/sites-enabled/certbot.conf
```

8.6 Test the Nginx configuration:
```bash
sudo nginx -t
```

8.7 Reload Nginx to apply the final configuration:
```bash
sudo systemctl reload nginx
```

---

## **Step 9: Configure Prometheus**

9.3 Set ownership and permissions for the authentication file:
```bash
sudo chown prometheus:prometheus /etc/prometheus/web.yml
sudo chmod 644 /etc/prometheus/web.yml
```

9.4 Validate the updated Prometheus configuration:
```bash
promtool check config /etc/prometheus/prometheus.yml
```

---

## **Step 10: Configure Prometheus as a systemd Service**

10.2 Reload systemd:
```bash
sudo systemctl daemon-reload
```

**Comment:** Reloading ensures the new Prometheus service file is recognized by `systemd`.

---

## **Step 11: Install node_exporter**

11.5 Validate node_exporter is running on the default port `9100`:
```bash
curl http://localhost:9100/metrics
```

---

## **Step 12: Integrate node_exporter with Prometheus**

12.1 Validate the updated Prometheus configuration:
```bash
promtool check config /etc/prometheus/prometheus.yml
```

---

## **Step 13: Integrate Prometheus with Grafana**

13.1 Add Prometheus as a data source in Grafana and validate the connection.

---
