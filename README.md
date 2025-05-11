# Vedic-Frontend

```bash
# First clone the repo
git clone git@github.com:MindRisers-Technologies/Vedic-healing---frontend.git

# Install packages
npm install

# Build application
npm run build

# Creates 2 main folders needed for deployment
node_modules .next

# setting up firewall, Open port for frontend
sudo ufw allow 3002

# Now create ecosystem.config.js file at project folder to run PM2
vi ecosystem.config.js

module.exports = {
  apps : [{
    name   : "Vedic-Healing",
    script : "node_modules/.bin/next",  # this is a executable file for the Next.js framework
    args: "start",
      env: {
        NODE_ENV: "production",
        NEXTAUTH_URL: "http://localhost:3002",
        PORT: 3002
      }
  }]
};

# Check if the application is running using PM2
pm2 start ecosystem.config.js

# Now setup nginx for reverse proxy
# Go to /etc/nginx/sites-available and create a config file
sudo vi vedic-healing.conf

server {
    listen 80;
    server_name vedic.janaksmriti.edu.np;

    location / {
        proxy_pass http://localhost:3002;
    }
}

# Create a symbolic link to sites-enabled folder
ln -s /etc/nginx/sites-available/vedic-healing.conf /etc/nginx/sites-enabled/

# Test the configuration file
nginx -t

# Reload nginx to see the change
systemctl reload nginx

# Visit website
vedic.janaksmriti.edu.np
```

Environment File

```bash
# .env file
# provided by the frontend team.

# base URL for API requests.
NEXT_PUBLIC_API_BASE_URL=http://ip:port/api/v1/

# For User
NEXT_PUBLIC_API_USERBASE_URL=http://ip:port/auth/

# For User Register
NEXT_PUBLIC_API_REGISTERBASE_URL=http://ip:port/api/v1/auth/
```

GitHub Actions (CI/CD)

```bash
name: Vedic Frontend CI-CD
on:
  push:
    branches:
      - master
  workflow_dispatch:  # to manually trigger github-action
      
# Build job
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "22"
      - name: Install dependencies
        run: npm ci
     ## Adds .env file in github action container runner
      - name: Create .env file
        run: echo "NEXT_PUBLIC_API_BASE_URL=${{ secrets.NEXT_PUBLIC_API_BASE_URL }}" > .env
     ## Uses the above created .env file during build   
      - name: Application Build
        run: npm run build
      - name: Compress dependencies
        run: tar -cvzf bundle.tar.gz node_modules .next  # bundles node_modules and .next into a single tar file
        
# Deployment job
      - name: Copy tar files to Deployment server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          password: ${{ secrets.PASSWORD }}
          port: ${{ secrets.PORT }}
          source: bundle.tar.gz
          target: /var/www/html/vedic-healing/Vedic-healing---frontend/  # pastes the tar file in project directory in server
      - name: Extract tar files on Deployment server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          password: ${{ secrets.PASSWORD }}
          port: ${{ secrets.PORT }}
          script: |
            # export PATH=$PATH:/root/.nvm/versions/node/v22.14.0/bin
            cd /var/www/html/vedic-healing/Vedic-healing---frontend/
            tar -xvzf bundle.tar.gz  # extract the tar file
            rm bundle.tar.gz
            # pm2 restart ecosystem.config.js
```

# Vedic-Backend

```bash
# First clone the repo from github
git clone git@github.com:MindRisers-Technologies/Vedic-Healing-_-Backend.git

# Install pip to install other python packages
sudo apt install python3-pip

# Install to create a virtual-environment
sudo apt install python3-venv

# create venv in a same directory as manage.py
python3 -m venv venv

# Activates virtual environment
source venv/bin/activate

# installs all the necessary packages needed to run the project
pip install -r requirements.txt

# Install django in project folder if not included in requirements.txt
pip install django

# Install gunicorn in the project folder
pip install gunicorn

# Install PostgreSQL and PostgreSQL-Client if not installed
# sudo apt install postgresql postgresql-client

# Create a database for project in server
# sudo -i -u postgres
# psql
CREATE DATABASE vedic;
# since user postgress already exists just change the password for it inside vedic database
ALTER USER postgres WITH PASSWORD 'newpassword';
GRANT ALL PRIVILEGES ON DATABASE vedic TO postgres;
# Exit psql
\q
# Connect to the database vedic as a user postgres
psql -U postgres -d vedic
# shows List of databases
\l

# Creates migration files from model changes made by developers
python3 manage.py makemigrations

# Applies migration files to the database
python3 manage.py migrate

# Collects static files 
python3 manage.py collectstatic

# Creates SuperUser for Admin access
python3 manage.py createsuperuser
Email: 
Username:
Password:

# Deactivates virtual environment
deactivate

# To activate gUnicorn which provides production-ready serving for django application
# sudo vi /etc/systemd/system/vedic.socket
[Unit]
Description=vedic socket

[Socket]
ListenStream=/run/vedic.sock

[Install]
WantedBy=sockets.target

# sudo vi /etc/systemd/system/vedic.service
[Unit]
Description=vedic daemon
Requires=vedic.socket
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/var/www/html/vedic-healing/Vedic-Healing-_-Backend
ExecStart=/var/www/html/vedic-healing/Vedic-Healing-_-Backend/venv/bin/gunicorn \
 --access-logfile - \
 --reload \
 --workers 3 \
 --bind unix:/run/vedic.sock \
 project.wsgi:application

[Install]
WantedBy=multi-user.target

# Start gunicorn socket for vedic
# This will automatically start vedic.service also
sudo systemctl start vedic.socket

# Enable the socket to start on boot
sudo systemctl enable vedic.socket

# Configure Nginx to Proxy Pass to Gunicorn
# sudo vi /etc/nginx/sites-available/vedic.conf
server {
    listen 8090; # Insert port where the project is running
    server_name _; # Insert server_ip where the project is running

    location / {
        proxy_pass http://unix:/run/vedic.sock;  # tells Nginx to use a Unix socket file instead of a TCP/IP port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /var/www/html/vedic-healing/Vedic-Healing-_-Backend;
    }

    location /media/ {
        root /var/www/html/vedic-healing/Vedic-Healing-_-Backend;
    }
}

# Create a symbolic link of conf file in sites-enabled folder
ln -s /etc/nginx/sites-available/vedic.conf /etc/nginx/sites-enabled/

# Test nginx configuration file
sudo nginx -t

# setting up firewall, Open port for backend
sudo ufw allow 8090

# Reload nginx to load the changes from new config file
sudo systemctl reload nginx
```

Environment File

```bash
# .env file
# provided by backend team

SECRET_KEY = '*'
DEBUG = True

# Only allowed hosts can access the web
ALLOWED_HOSTS =localhost,127.0.0.1,server_ip,vedic.janaksmriti.edu.np

# setting that specifies which external domains are trusted to make cross-origin POST requests
# protects against Cross-Site Request Forgery (CSRF) attacks
CSRF_TRUSTED_ORIGINS = "http://localhost:8000,http://127.0.0.1:8000,http://server_ip:8090,https://vedic.janaksmriti.edu.np"

# Cross-Origin Resource Sharing
# If frontend is on a different domain or port from backend, browser blocks requests unless server explicitly allows it
CORS_ALLOWED_ORIGINS =http://localhost:3000,http://127.0.0.1:3000,http://server_ip:8090,https://vedic.janaksmriti.edu.np

# For static files
STATIC_ROOT = /var/www/html/vedic-healing/Vedic-Healing-_-Backend/static

IMAGE_URL = 'http://server_ip:8090'
# Swagger UI for API documentation 
SWAGGER_URL = 'http://server_ip:8090'
# For user uploaded media
MEDIA_ROOT =/var/www/html/vedic-healing/Vedic-Healing-_-Backend/media

# Database connection
DB_ENGINE = 'django.db.backends.postgresql'
DB_NAME = 'vedic'
DB_USER = 'postgres'
DB_PASSWORD = '*'
DB_PORT = '5432'
DB_HOST = 'localhost'

EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_USE_TLS=True
EMAIL_PORT = 587
EMAIL_HOST_USER = 'vedichealing@gmail.com'
EMAIL_HOST_PASSWORD = '*'
```

GitHub Actions (CI/CD)

```bash
name: CI-CD for Vedic Healing

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Code Checkout
      uses: actions/checkout@v4

    - name: Vedic Backend Deployment
      uses: appleboy/ssh-action@v1.0.3
      with:
          host: ${{secrets.HOST }}
          username: ${{secrets.USERNAME }}
          password: ${{secrets.PASSWORD }}
          port: ${{secrets.PORT }}
          script: |
            echo "Opening Directory..."
            cd /var/www/html/vedic-healing/Vedic-Healing-_-Backend

            # If need to switch to another branch or pull in updates without losing your current work.
            git stash
            
            echo "Updating local to match remote repository"
            git pull

            echo "Entering Virtual Environment ..."
            source venv/bin/activate

            echo "Installing Requirements..."
            pip install -r requirements.txt
            
            echo "Running migration if new changes found"
            python3 manage.py makemigrations

            echo "Running Migrate..."
            python3 manage.py migrate

            echo "Collecting static files"
            python3 manage.py collectstatic --noinput

            echo "Deactivating Virtual Environment..."
            deactivate

            echo "Restarting Vedic Healing"
            sudo systemctl restart vedic
```

![image.png](image.png)

![image.png](image%201.png)

![image.png](image%202.png)