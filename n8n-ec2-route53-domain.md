<h1>🌐 Connecting Hostinger Domain to AWS Route 53 and EC2</h1>

This guide will help you connect your domain purchased from Hostinger to AWS Route 53 and then point it to an EC2 instance.

📌 Prerequisites

Domain purchased on Hostinger
.

AWS account with Route 53 and EC2 services enabled.

An EC2 instance (e.g., Ubuntu, Amazon Linux).

Basic knowledge of DNS records.

🚀 Steps
1. Create a Hosted Zone in Route 53

Go to the AWS Management Console → Route 53.

Click Hosted zones → Create hosted zone.

Enter your domain name (e.g., example.com).

Choose Public hosted zone.

Click Create.

Route 53 will generate Name Servers (NS records) like:

ns-123.awsdns-45.com
ns-678.awsdns-90.net
ns-234.awsdns-12.org
ns-345.awsdns-56.co.uk

2. Update Nameservers in Hostinger

Log in to your Hostinger control panel
.

Go to Domains → Select your domain.

Find DNS / Nameservers settings.

Replace Hostinger’s default nameservers with the Route 53 NS records you got in Step 1.

Save changes.

⚠️ Important:

Hostinger updates may take up to 24 hours to fully propagate worldwide.

During this period, some users may still see the old nameservers while others see the new ones.

3. Get Your EC2 Public IP

Go to the EC2 Dashboard in AWS.

Select your instance → Copy the Public IPv4 address.
Example: 13.232.45.67

4. Add DNS Records in Route 53

Open your Hosted Zone in Route 53.

Click Create Record.

Add an A Record:

Record name: Leave blank (for root domain example.com) OR put www if you want www.example.com.

Record type: A – IPv4 address

Value: Your EC2 Public IP (e.g., 13.232.45.67)

TTL: 300 (default).

Save.

(Optional) Add a CNAME record for www:

Record name: www

Record type: CNAME

Value: example.com

5. Test Your Domain

Open terminal and run:

nslookup example.com


It should show your EC2’s public IP.

Open a browser → Visit http://example.com → It should load your EC2 web server.

✅ Notes

Ensure EC2 Security Groups allow inbound traffic on port 80 (HTTP) and 443 (HTTPS if using SSL).

If using Elastic IP (recommended), assign it to your EC2 so your IP won’t change.

For HTTPS, use AWS Certificate Manager (ACM) or Let’s Encrypt.

Patience is key: DNS propagation delays are normal when switching from Hostinger to Route 53.

<h1> n8n Installation on AWS EC2 with Domain and SSL </h1>


Install n8n on an AWS EC2 instance, accessible via a domain (domain.shop) with HTTPS and correct OAuth2 redirect handling.

Step 1: Launch a new EC2 instance

AMI: Ubuntu 22.04 LTS

Security Group: Open ports:

22 (SSH)

80 (HTTP)

443 (HTTPS)

Note the public IP for initial setup.

Step 2: Connect to your EC2
ssh ubuntu@<EC2_PUBLIC_IP>


Replace <EC2_PUBLIC_IP> with your instance IP.

Step 3: Install Docker & Docker Compose
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker


Comment: Docker is required to run n8n in a container. Docker Compose simplifies multi-service setups.

Step 4: Create n8n project folder
mkdir -p ~/n8n/n8n
cd ~/n8n


~/n8n/n8n will be used to store n8n data persistently.

Fix folder permissions for n8n (common issue):

sudo chown -R 1000:1000 ~/n8n/n8n


This avoids EACCES: permission denied errors when the container tries to write files.

Step 5: Create Docker Compose file
vim docker-compose.yml


Paste the following:

version: '3'

services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"  # Internal port, not exposed outside (Nginx handles public)
    environment:
      # Basic authentication for security
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=yourpassword

      # Domain and webhook configuration
      - N8N_HOST=yesgive.shop
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://yesgive.shop/

      # Production environment
      - NODE_ENV=production
    volumes:
      - ./n8n:/home/node/.n8n  # Persist data
    # Optional: run as root if permission issues occur
    # user: root


Comments:

N8N_HOST and WEBHOOK_URL ensure OAuth2 and webhooks use your domain instead of localhost.

Volume mounting allows data persistence.

Permissions may need fixing if you get encryption key errors.

Step 6: Start n8n
docker-compose up -d


Comment: The -d runs the container in detached mode.

Step 7: Install Nginx as a reverse proxy
sudo apt install -y nginx
sudo ufw allow 'Nginx Full'


Create Nginx configuration:

sudo vim /etc/nginx/sites-available/n8n


Paste:

server {
    server_name yesgive.shop www.yesgive.shop;

    location / {
        proxy_pass http://localhost:5678;  # Forward to Docker n8n container
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}


Enable config:

sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx


Comment:

proxy_pass http://localhost:5678 is internal to EC2. External users only see https://yesgive.shop.

Nginx handles SSL and HTTP→HTTPS redirection.

Step 8: Install free SSL (Let’s Encrypt)
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yesgive.shop -d www.yesgive.shop


Follow prompts to secure your site.

Certbot auto-configures HTTPS and renews certificates.

Step 9: Test OAuth2 and webhooks

Open: https://yesgive.shop

When adding OAuth2 credentials, redirect URLs should now be like:

https://yesgive.shop/rest/oauth2-credential/callback


✅ No more localhost:5678.

Step 10: Optional: Start n8n on boot
docker-compose restart


For production, you can also use Docker Compose as a systemd service.

Common Issues & Solutions
Issue	Cause	Solution
EACCES: permission denied /home/node/.n8n/config	Docker container cannot write to host volume	Fix permissions: sudo chown -R 1000:1000 ~/n8n/n8n
OAuth redirects to localhost	N8N_HOST or WEBHOOK_URL not set	Set N8N_HOST=yesgive.shop and WEBHOOK_URL=https://yesgive.shop/ in env
SSL not working	Nginx not configured or certificate missing	Run sudo certbot --nginx -d yesgive.shop
Nginx shows default page	Wrong site-enabled configuration	Ensure symlink exists: sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/

✅ Result:

n8n running on AWS EC2

Accessible via your domain https://yesgive.shop

OAuth2 credentials and webhooks work externally

SSL enabled and auto-renewed

Data persists between container restarts
