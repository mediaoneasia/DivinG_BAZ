EC2 deployment checklist

1) Open EC2 Security Group for port 80 (and 443 for TLS)

# Using AWS Console: EC2 → Instances → select instance → Security → Security Group → Edit inbound rules → Add rule: HTTP (TCP/80) Source=0.0.0.0/0

# Using AWS CLI (replace sg-XXXXXXXX with your SG id):
aws ec2 authorize-security-group-ingress --group-id sg-XXXXXXXX \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-XXXXXXXX \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

2) Ensure OS firewall allows ports (if applicable)
# UFW:
sudo ufw allow 'Nginx Full'
# firewalld:
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

3) Install and enable systemd unit for Gunicorn
# Edit the template at deploy/gunicorn.service, then copy to systemd and enable:
sudo cp deploy/gunicorn.service /etc/systemd/system/gunicorn.service
# Edit placeholders:
#  - <USER> -> system user that should run the app (e.g., ubuntu or www-data)
#  - <GROUP> -> usually same as user
#  - <APP_DIR> -> full path to app directory (e.g., /home/ubuntu/DivinG_BAZ)
#  - <GUNICORN_BIN> -> full path to gunicorn executable (e.g., /home/ubuntu/.local/bin/gunicorn)
#  - <GUNICORN_BIN_PATH> -> directory containing gunicorn binary

sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn
sudo systemctl status gunicorn

4) Nginx reverse proxy
# Copy the nginx SSL template at deploy/nginx-ssl.conf to /etc/nginx/sites-available/diving-ssl
sudo cp deploy/nginx-ssl.conf /etc/nginx/sites-available/diving-ssl
sudo ln -sfn /etc/nginx/sites-available/diving-ssl /etc/nginx/sites-enabled/diving-ssl
sudo nginx -t && sudo systemctl reload nginx

5) Obtain TLS certs (Let's Encrypt)
# Using certbot (nginx plugin):
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com

6) Test externally
curl -I http://<EC2-PUBLIC-IP>/
curl -I https://example.com/

Notes:
- Replace placeholders in templates before copying.
- If your EC2 instance uses a non-root user with a virtualenv, point ExecStart to that virtualenv gunicorn binary.
